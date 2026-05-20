# Getting started

***

Use this page for first-pass concepts, common tools, and early service checks.

### Cheat sheets

| Cheat sheet                               | Focus                                          |
| ----------------------------------------- | ---------------------------------------------- |
| [Common ports](../cheat-sheets/page-3.md) | Default ports and quick service lookup         |
| [TMUX](../cheat-sheets/tmux.md)           | Session, window, and pane shortcuts            |
| [VIM](../cheat-sheets/vim.md)             | Fast editing and navigation                    |
| [Nmap](../cheat-sheets/nmap.md)           | Common scans, NSE scripts, and banner grabbing |
| [SMB](../cheat-sheets/smb.md)             | Share enumeration and access commands          |
| [SNMP](../cheat-sheets/snmp.md)           | `snmpwalk` and community string checks         |

## Common Terms

### Shell :

The shell is a program that takes the input from the user and passes these commands to OS (Operating System) to perform a specific function.

<figure><img src="../../.gitbook/assets/image (2) (1).png" alt=""><figcaption></figcaption></figure>

### Port :

Ports are virtual points where network connections begin and end.

They allow a computer to route different types of traffic simultaneously over a single network connection by mapping specific data streams to distinct software processes (e.g., SSH vs. web requests).

{% include "../../.gitbook/includes/protocol-comparison-tcp-vs....md" %}

<figure><img src="../../.gitbook/assets/image (1) (1) (1).png" alt=""><figcaption></figcaption></figure>

### Web Server :

Software running on a host machine that directly handles HTTP/HTTPS traffic from a client browser over TCP ports 80 and 443.

It processes incoming requests, maps them to physical files, or hands them off to the application backend and it has a huge attack surface.

Attack Surface Breakdown:

Web Server Vulnerabilities and Web Application Flaws (OWASP Top 10)

{% embed url="https://owasp.org/Top10/2025/" %}

## Common Tools

### Using SSH :

Secure Shell is a network protocol that runs on port 22 by default and provides users a secure way to access a computer remotely.

SSH can be configured with password authentication or passwordless using public-key authentication using public/private key-pair.

<figure><img src="../../.gitbook/assets/image (3) (1).png" alt=""><figcaption></figcaption></figure>

### Using Netcat :

Netcat, ncat or nc is used to interact with TCP/UDP Ports.

It can be used for many things during a pentest.

Its primary usage is for connecting to shells.

Netcat can be used to connect to any listening port and interact with the service on that port

<figure><img src="../../.gitbook/assets/image (4).png" alt=""><figcaption></figcaption></figure>

Netcat can be used this way to obtain the banner running on that Port and IP.

This is called as Banner Grabbing.

Netcat can also be used to transfer files.

```
nc -nv <Target_IP> <Target_Port>
```

Then there is also Socat which is called as Netcat on steriods.A [standalone binary](https://github.com/andrew-d/static-binaries) of `Socat` can be transferred to a system after obtaining remote code execution to get a more stable reverse shell connection.

Socat also supports forwarding ports and connecting to serial devices.

1. **Port Forwarding :**

When you hack a target network, you often find a central server (like a database) that is completely hidden from the internet behind a firewall.

You cannot talk to it directly.

But if you have already compromised the public web server sitting right next to it, you can tell that web server: _"Take any traffic I send to you on Port X, and automatically forward it to the hidden database on Port Y."_

2. **Connecting to Serial Devices :**

* Serial Devices: Physical hardware components (like routers, IoT devices, or microchips) that transmit data sequentially, one bit at a time.
* Security Purpose: Involves physically opening a device's casing, connecting a cable directly to diagnostic pins on the circuit board (like UART), and using tools to read raw data or drop straight into a root command line without a network connection and password.

### TMUX :

Terminal multiplexers, like `tmux` or `Screen`, are great utilities for expanding a standard Linux terminal's features, like having multiple windows within one terminal and jumping between them.

{% embed url="https://tmuxcheatsheet.com/" %}

### VIM:

Vim is a great text editor that can be used for writing code or editing text files on Linux systems.

{% embed url="https://vimsheet.com/" %}

## Service Scanning

### Nmap :

Nmap is used to scan ports and let us know the services that are running.

Basic nmap scan (scans 1000 most common ports by default and by default it runs a tcp scan) :

<figure><img src="../../.gitbook/assets/image (3).png" alt=""><figcaption></figcaption></figure>

```
nmap IP
```

We can use the `-sC` parameter to specify that `Nmap` scripts should be used to try and obtain more detailed information. The `-sV` parameter instructs `Nmap` to perform a version scan. In this scan, Nmap will fingerprint services on the target system and identify the service protocol, application name, and version. The version scan is underpinned by a comprehensive database of over 1,000 service signatures. Finally, `-p-` tells Nmap that we want to scan all 65,535 TCP ports.

<br>

<figure><img src="../../.gitbook/assets/image (1) (1).png" alt=""><figcaption></figcaption></figure>

```
nmap -sV -sC -p- IP
```

-sV → version detection (what software + version)\
-sC → run default scripts (extra info gathering)\
-p- → scan all 65535 ports (not just top 1000)

### Nmap Scripts :

Specifying `-sC` will run many useful default scripts against a target, but there are cases when running a specific script is required.

```
nmap --script <script-name> IP
nmap --script ftp-anon IP          # check anonymous FTP
```

```
nmap --script vuln IP              # check known vulns
nmap --script ftp-anon IP          # anonymous FTP check
nmap --script http-enum IP         # web enumeration
nmap --script smb-vuln-ms17-010 IP # EternalBlue check
```

## Attacking Network Services

### Banner Grabbing :

**Fingerprinting** = identifying exactly what is running on a target.

Like a human fingerprint — unique to each person. Every service leaves identifying information.

**Banner grabbing is one way to fingerprint :**

```
# Using Nmap
nmap -sV --script=banner IP

# Using Netcat manually
nc -nv 10.129.42.253 21

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

### FTP :

`Nmap` scan of the default port for FTP (21) reveals the vsftpd 3.0.3 installation that we identified previously. Further, it also reports that anonymous authentication is enabled and that a `pub` directory is available.

<figure><img src="../../.gitbook/assets/image (5).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../../.gitbook/assets/image (6).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../../.gitbook/assets/image (7).png" alt=""><figcaption></figcaption></figure>

### SMB :

SMB (Server Message Block) is a prevalent protocol on Windows machines that provides many vectors for vertical and lateral movement. `Nmap` has many scripts for enumerating SMB, such as [smb-os-discovery.nse](https://nmap.org/nsedoc/scripts/smb-os-discovery.html), which will interact with the SMB service to extract the reported operating system version.

<figure><img src="../../.gitbook/assets/image (8).png" alt=""><figcaption></figcaption></figure>

```
nmap -A -p{Port} IP
```

This is an Aggressive scan.

<figure><img src="../../.gitbook/assets/image (10).png" alt=""><figcaption></figcaption></figure>

#### Why This Matters in Pentesting

The text is demonstrating the progression of enumeration. You start by finding an open port (`445`), then you use specialized tools (NSE scripts or `-A` flags) to pull the banners and exact software versions. Once you have those precise versions (e.g., _Windows 7 SP1_ or _Samba 4.6.2_), you can cross-reference them against public vulnerability databases to find specific, actionable exploits like EternalBlue.<br>

### Shares :

* SMB (Server Message Block): Protocol used to share folders and files remotely.
* Security Risk: Shares frequently expose sensitive data (like hardcoded passwords).
* Tool (`smbclient`): Used to enumerate and interact with SMB shares from the command line.
  * `-L` : Lists all available shares on the target host.
  * `-N` : Suppresses the password prompt (forces an anonymous connection attempt).

Listing available shares:

```
smbclient -N -L \\\\IP
```

<figure><img src="../../.gitbook/assets/image.png" alt=""><figcaption></figcaption></figure>

After finding non default share i.e users, we attempt to connect to it.

```
smbclient \\\\IP\\{non default share}
```

<figure><img src="../../.gitbook/assets/image (1).png" alt=""><figcaption></figcaption></figure>

As this failed, we need to find valid creds elsewhere and then...

```
smbclient -U <username> \\\\<Target_IP>\\<Share_Name> 
```

after this, the tool will ask password and then once inside the `smb: \>` prompt, it functions similarly to a basic FTP or Linux shell.

<figure><img src="../../.gitbook/assets/image (2).png" alt=""><figcaption></figcaption></figure>

we can now use the get command to get the passwords.txt file.

### SNMP :

* SNMP (Ports 161/162 TCP/UDP): Used for device management.
* Versions 1 & 2c: Insecure; rely on unencrypted plaintext community strings instead of passwords.
* Default Danger: Default strings are almost always `public` (read) or `private` (write).
* Pentest Value: Leaks running processes (revealing command-line passwords), internal routing tables (revealing hidden network paths), and exact software versions.

#### snmpwalk :

```
snmpwalk -v 2c -c public 10.129.42.253 1.3.6.1.2.1.1.5.0
```

* `snmpwalk`: A tool that sends sequential requests to "walk" through a device's management data tree.
* `-v 2c`: Specifies that it is using SNMP version 2c (the insecure, plaintext version).
* `-c public`: Supplies `public` as the community string (password).
* `10.129.42.253`: The target IP address.
* `1.3.6.1.2.1.1.5.0`: This long string of numbers is an OID (Object Identifier). In the universal SNMP database schema, this exact OID points directly to the system's Hostname.

<figure><img src="../../.gitbook/assets/image (11).png" alt=""><figcaption></figcaption></figure>

If the community string fails then you must use onesixtyone which bruteforces...

```
onesixtyone -c dict.txt 10.129.42.254
```

* `onesixtyone`: A highly efficient, multi-threaded scanner built specifically to brute-force SNMP community strings.
* `-c dict.txt`: Passes a text file containing a dictionary of common community strings (e.g., `public`, `private`, `internal`, `manager`, `cisco`).

<figure><img src="../../.gitbook/assets/image (12).png" alt=""><figcaption></figcaption></figure>
