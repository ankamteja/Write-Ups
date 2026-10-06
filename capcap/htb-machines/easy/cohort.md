# Cohort

**Platform:** Hack The Box · **Difficulty:** Easy · **OS:** Linux **Status:** user ✓ · root pending (privesc identified and confirmed, not yet executed)

> Box is **ACTIVE** — keep this private until Cohort retires (HTB ToS). The root exploit step is deliberately left as a checklist, not written out, for the same reason.

***

### 1. Recon

nmap — 3 ports, HTTP redirects to a hostname.

```bash
nmap -p22,80,443 10.129.244.174 -Pn -sCV -oA cohort
```

* **22** — OpenSSH 9.6p1 (Ubuntu)
* **80 / 443** — nginx 1.24.0, both redirect to `https://cohort.htb/`
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

### 3. SSRF — confirm + filter bypass

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

### 4. Foothold — marimo pre-auth RCE (CVE-2026-39987)

```bash
echo '10.129.244.174 nb-1be3782a8afd3ad5.cohort.htb' | sudo tee -a /etc/hosts
```

Login page title → **marimo** (reactive Python notebook), not Jupyter. Single password field; version string `0.20.4`.

marimo ≤ 0.20.4: **unauthenticated WebSocket RCE**. `/terminal/ws` skips the `validate_auth()` check every other WS endpoint enforces, so an unauthenticated connection yields a PTY shell. The password never matters — walk around it.

* Endpoint: `wss://nb-1be3782a8afd3ad5.cohort.htb/terminal/ws`
* Frames: raw xterm PTY bytes (not JSON)
* Small `websockets` Python client → shell as `uid=1000(marimo)`

Grabbed `user.txt`. Planted an SSH key in `~marimo/.ssh/authorized_keys` for a stable session.

> Lesson: the `echo >> authorized_keys` runs **on the target** (`marimo@cohort`), not on Kali. Watch the prompt. WebSocket PTY is fragile — move to SSH early.

### 5. Privilege escalation — identified & confirmed, exploit pending

Enumeration as `marimo` (no sudo rights):

* `notebooks/retention.py` — no creds (red herring)
* `/opt/sysmon/` — custom root process, but **not readable** → eliminated
* **PackageKit** is the path:

```
hi  packagekit   1.2.8-2ubuntu1.2     ← held (apt-mark hold)
```

```
Installed: 1.2.8-2ubuntu1.2    ← held back
Candidate: 1.2.8-2ubuntu1.5    ← security-patched version available but blocked
```

A package **deliberately held at a pre-patch version** = planted vuln. The fix landed in `-2ubuntu1.5` (noble-security); the box is frozen one step before it. polkitd running; no persistent packagekitd (D-Bus activated).

→ **PackageKit local privesc via TOCTOU race (CVE-2026-41651).**

***

### TODO (root)

* \[ ] Read CVE-2026-41651 primary advisory for the exact D-Bus `InstallFiles` sequence
* \[ ] Build malicious `.deb` with a `postinst` that SUIDs bash
* \[ ] Race loop: two `InstallFiles` calls, retry until it wins (races lose often)
* \[ ] `bash -p` → root → `root.txt`

### Notes to self

* Path was reconstructed from the box itself, not a writeup: SSRF filter is string-only → decimal bypass; `/status` leaked the vhost; title tag said marimo not Jupyter; package `hold` + security-candidate gap pointed at PackageKit. That chain of reasoning is the transferable part.
* Why TOCTOU works (understand before exploiting): the privileged daemon checks authorization at one moment (time-of-check) and performs the install at a later moment (time-of-use). If the operation's target/flags change inside that window, the check approved one thing while root executes another. A `.deb` carries `postinst`, which `dpkg` runs as root by design — so winning the race on an arbitrary package = root code execution.
