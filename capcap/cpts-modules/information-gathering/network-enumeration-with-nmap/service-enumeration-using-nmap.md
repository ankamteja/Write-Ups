---
description: Identify service versions, inspect banners, and verify findings manually.
---

# Service Enumeration

Use service enumeration to confirm what is actually running on a port.

That gives you:

* product names
* version numbers
* hostnames and OS clues
* better exploit search terms

Exact versions matter. They help you match public vulnerabilities, verify defaults, and narrow your next step quickly.

### Run service version detection

Start with a quick scan to find obvious ports.

Then run a full TCP scan with version detection in the background.

This keeps traffic lower at first and helps you avoid wasting time on closed ports.

#### Scan all TCP ports with `-sV`

Use `-p-` to scan all TCP ports and `-sV` to identify services and versions.

```bash
impale7@htb[/htb]$ sudo nmap 10.129.2.28 -p- -sV

Starting Nmap 7.80 ( https://nmap.org ) at 2020-06-15 19:44 CEST
[Space Bar]
Stats: 0:00:03 elapsed; 0 hosts completed (1 up), 1 undergoing SYN Stealth Scan
SYN Stealth Scan Timing: About 3.64% done; ETC: 19:45 (0:00:53 remaining)
```

What this does:

* `10.129.2.28` targets the host
* `-p-` scans all TCP ports
* `-sV` performs service version detection

{% hint style="info" %}
Press the space bar during a running scan to print progress without stopping it.
{% endhint %}

#### Show periodic scan status automatically

Use `--stats-every` when you want regular progress updates.

```bash
impale7@htb[/htb]$ sudo nmap 10.129.2.28 -p- -sV --stats-every=5s

Starting Nmap 7.80 ( https://nmap.org ) at 2020-06-15 19:46 CEST
Stats: 0:00:05 elapsed; 0 hosts completed (1 up), 1 undergoing SYN Stealth Scan
SYN Stealth Scan Timing: About 13.91% done; ETC: 19:49 (0:00:31 remaining)
Stats: 0:00:10 elapsed; 0 hosts completed (1 up), 1 undergoing SYN Stealth Scan
SYN Stealth Scan Timing: About 39.57% done; ETC: 19:48 (0:00:15 remaining)
```

What this does:

* `10.129.2.28` targets the host
* `-p-` scans all TCP ports
* `-sV` performs service version detection
* `--stats-every=5s` prints progress every 5 seconds

#### Increase verbosity during the scan

Use `-v` or `-vv` to see open ports as Nmap discovers them.

```bash
impale7@htb[/htb]$ sudo nmap 10.129.2.28 -p- -sV -v

Starting Nmap 7.80 ( https://nmap.org ) at 2020-06-15 20:03 CEST
NSE: Loaded 45 scripts for scanning.
Initiating ARP Ping Scan at 20:03
Scanning 10.129.2.28 [1 port]
Completed ARP Ping Scan at 20:03, 0.03s elapsed (1 total hosts)
Initiating Parallel DNS resolution of 1 host. at 20:03
Completed Parallel DNS resolution of 1 host. at 20:03, 0.02s elapsed
Initiating SYN Stealth Scan at 20:03
Scanning 10.129.2.28 [65535 ports]
Discovered open port 995/tcp on 10.129.2.28
Discovered open port 80/tcp on 10.129.2.28
Discovered open port 993/tcp on 10.129.2.28
Discovered open port 143/tcp on 10.129.2.28
Discovered open port 25/tcp on 10.129.2.28
Discovered open port 110/tcp on 10.129.2.28
Discovered open port 22/tcp on 10.129.2.28
<SNIP>
```

What this does:

* `10.129.2.28` targets the host
* `-p-` scans all TCP ports
* `-sV` performs service version detection
* `-v` increases scan verbosity

### Review the service results

Once the scan finishes, Nmap shows open ports, service names, and version details.

```bash
impale7@htb[/htb]$ sudo nmap 10.129.2.28 -p- -sV

Starting Nmap 7.80 ( https://nmap.org ) at 2020-06-15 20:00 CEST
Nmap scan report for 10.129.2.28
Host is up (0.013s latency).
Not shown: 65525 closed ports
PORT      STATE    SERVICE      VERSION
22/tcp    open     ssh          OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
25/tcp    open     smtp         Postfix smtpd
80/tcp    open     http         Apache httpd 2.4.29 ((Ubuntu))
110/tcp   open     pop3         Dovecot pop3d
139/tcp   filtered netbios-ssn
143/tcp   open     imap         Dovecot imapd (Ubuntu)
445/tcp   filtered microsoft-ds
993/tcp   open     ssl/imap     Dovecot imapd (Ubuntu)
995/tcp   open     ssl/pop3     Dovecot pop3d
MAC Address: DE:AD:00:00:BE:EF (Intel Corporate)
Service Info: Host:  inlane; OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 91.73 seconds
```

What this does:

* `10.129.2.28` targets the host
* `-p-` scans all TCP ports
* `-sV` performs service version detection

Nmap mainly identifies services from banners.

If banners are incomplete, Nmap falls back to signature matching.

That can take longer, and it can still miss useful details.

### Trace how Nmap identifies a service

Use packet tracing when you want to see the raw interaction behind a detected service.

```bash
impale7@htb[/htb]$ sudo nmap 10.129.2.28 -p- -sV -Pn -n --disable-arp-ping --packet-trace

Starting Nmap 7.80 ( https://nmap.org ) at 2020-06-16 20:10 CEST
<SNIP>
NSOCK INFO [0.4200s] nsock_trace_handler_callback(): Callback: READ SUCCESS for EID 18 [10.129.2.28:25] (35 bytes): 220 inlane ESMTP Postfix (Ubuntu)..
Service scan match (Probe NULL matched with NULL line 3104): 10.129.2.28:25 is smtp.  Version: |Postfix smtpd|||
NSOCK INFO [0.4200s] nsock_iod_delete(): nsock_iod_delete (IOD #1)
Nmap scan report for 10.129.2.28
Host is up (0.076s latency).

PORT   STATE SERVICE VERSION
25/tcp open  smtp    Postfix smtpd
MAC Address: DE:AD:00:00:BE:EF (Intel Corporate)
Service Info: Host:  inlane

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 0.47 seconds
```

Key flags:

* `-p-` scans all TCP ports
* `-sV` performs service version detection
* `-Pn` skips host discovery
* `-n` disables DNS resolution
* `--disable-arp-ping` disables ARP ping
* `--packet-trace` shows sent and received packets

The most useful line here is:

* `NSOCK INFO [0.4200s] nsock_trace_handler_callback(): Callback: READ SUCCESS for EID 18 [10.129.2.28:25] (35 bytes): 220 inlane ESMTP Postfix (Ubuntu)..`

That line shows extra banner data.

The SMTP service reveals `Ubuntu`, even though the final service summary only shows `Postfix smtpd`.

That happens because many services send an identifying banner right after the TCP three-way handshake.

At the packet level, that data often appears in a segment with the `PSH` flag set.

Some services do not send a banner immediately.

Some administrators also remove or modify banners on purpose.

### Grab the banner manually

If you want to verify what Nmap missed, connect to the service yourself and capture the traffic.

{% stepper %}
{% step %}
### Start a packet capture

```bash
impale7@htb[/htb]$ sudo tcpdump -i eth0 host 10.10.14.2 and 10.129.2.28

tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), capture size 262144 bytes
```
{% endstep %}

{% step %}
### Connect with `nc`

```bash
impale7@htb[/htb]$ nc -nv 10.129.2.28 25

Connection to 10.129.2.28 port 25 [tcp/*] succeeded!
220 inlane ESMTP Postfix (Ubuntu)
```
{% endstep %}

{% step %}
### Review the captured packets

```bash
18:28:07.128564 IP 10.10.14.2.59618 > 10.129.2.28.smtp: Flags [S], seq 1798872233, win 65535, options [mss 1460,nop,wscale 6,nop,nop,TS val 331260178 ecr 0,sackOK,eol], length 0
18:28:07.255151 IP 10.129.2.28.smtp > 10.10.14.2.59618: Flags [S.], seq 1130574379, ack 1798872234, win 65160, options [mss 1460,sackOK,TS val 1800383922 ecr 331260178,nop,wscale 7], length 0
18:28:07.255281 IP 10.10.14.2.59618 > 10.129.2.28.smtp: Flags [.], ack 1, win 2058, options [nop,nop,TS val 331260304 ecr 1800383922], length 0
18:28:07.319306 IP 10.129.2.28.smtp > 10.10.14.2.59618: Flags [P.], seq 1:36, ack 1, win 510, options [nop,nop,TS val 1800383985 ecr 331260304], length 35: SMTP: 220 inlane ESMTP Postfix (Ubuntu)
18:28:07.319426 IP 10.10.14.2.59618 > 10.129.2.28.smtp: Flags [.], ack 36, win 2058, options [nop,nop,TS val 331260368 ecr 1800383985], length 0
```
{% endstep %}
{% endstepper %}

### Understand the TCP exchange

The first three packets are the normal TCP three-way handshake:

1. `[SYN]` — `18:28:07.128564 IP 10.10.14.2.59618 > 10.129.2.28.smtp: Flags [S], <SNIP>`
2. `[SYN-ACK]` — `18:28:07.255151 IP 10.129.2.28.smtp > 10.10.14.2.59618: Flags [S.], <SNIP>`
3. `[ACK]` — `18:28:07.255281 IP 10.10.14.2.59618 > 10.129.2.28.smtp: Flags [.], <SNIP>`

Next, the server sends banner data:

4. `[PSH-ACK]` — `18:28:07.319306 IP 10.129.2.28.smtp > 10.10.14.2.59618: Flags [P.], <SNIP>`

Here, `PSH` means the server is pushing application data to the client.

At the same time, `ACK` confirms the earlier packets.

Finally, your client confirms receipt:

5. `[ACK]` — `18:28:07.319426 IP 10.10.14.2.59618 > 10.129.2.28.smtp: Flags [.], <SNIP>`

### Takeaways

Use service enumeration to do three things well:

* identify exact products and versions with `-sV`
* inspect packet traces when a result looks incomplete
* verify banners manually when you need higher confidence

### Further reading

See the official [Nmap version detection guide](https://nmap.org/book/man-version-detection.html).
