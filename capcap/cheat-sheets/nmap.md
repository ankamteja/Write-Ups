# Nmap

Use this page as a complete quick reference for the Nmap workflow covered in **Network Enumeration with Nmap**.

Use it when you need host discovery, port scanning, service enumeration, NSE checks, output handling, performance tuning, and firewall-aware scan options in one place.

### Basic syntax

Nmap uses this format:

```bash
nmap <scan types> <options> <target>
```

Common scan types:

```bash
nmap --help

<SNIP>
SCAN TECHNIQUES:
  -sS/sT/sA/sW/sM: TCP SYN/Connect()/ACK/Window/Maimon scans
  -sU: UDP Scan
  -sN/sF/sX: TCP Null, FIN, and Xmas scans
  --scanflags <flags>: Customize TCP scan flags
  -sI <zombie host[:probeport]>: Idle scan
  -sY/sZ: SCTP INIT/COOKIE-ECHO scans
  -sO: IP protocol scan
  -b <FTP relay host>: FTP bounce scan
<SNIP>
```

With `-sS`:

* `SYN-ACK` means `open`
* `RST` means `closed`
* no response often means `filtered`

Example:

```bash
sudo nmap -sS localhost

Starting Nmap 7.80 ( https://nmap.org ) at 2020-06-11 22:50 UTC
Nmap scan report for localhost (127.0.0.1)
Host is up (0.000010s latency).
Not shown: 996 closed ports
PORT     STATE SERVICE
22/tcp   open  ssh
80/tcp   open  http
5432/tcp open  postgresql
5901/tcp open  vnc-1

Nmap done: 1 IP address (1 host up) scanned in 0.18 seconds
```

### Host discovery

Start by finding which hosts are alive.

#### Scan a subnet

```bash
sudo nmap 10.129.2.0/24 -sn -oA tnet | grep for | cut -d" " -f5

10.129.2.4
10.129.2.10
10.129.2.11
10.129.2.18
10.129.2.19
10.129.2.20
10.129.2.28
```

What it does:

* `10.129.2.0/24` targets the whole subnet
* `-sn` skips the port scan
* `-oA tnet` saves `tnet.nmap`, `tnet.gnmap`, and `tnet.xml`

{% hint style="info" %}
This works best when hosts answer ICMP or other discovery probes.
{% endhint %}

If standard discovery fails, try direct TCP probes against likely ports such as `80` or `445`.

#### Scan an IP list

```bash
cat hosts.lst

10.129.2.4
10.129.2.10
10.129.2.11
10.129.2.18
10.129.2.19
10.129.2.20
10.129.2.28
```

```bash
sudo nmap -sn -oA tnet -iL hosts.lst | grep for | cut -d" " -f5

10.129.2.18
10.129.2.19
10.129.2.20
```

Use this when the scope is predefined.

If some hosts do not appear, they may be blocking or ignoring the default discovery traffic.

#### Scan multiple IPs directly

```bash
sudo nmap -sn -oA tnet 10.129.2.18 10.129.2.19 10.129.2.20 | grep for | cut -d" " -f5

10.129.2.18
10.129.2.19
10.129.2.20
```

If the IPs are consecutive, use a range:

```bash
sudo nmap -sn -oA tnet 10.129.2.18-20 | grep for | cut -d" " -f5

10.129.2.18
10.129.2.19
10.129.2.20
```

#### Scan one host

```bash
sudo nmap 10.129.2.18 -sn -oA host

Starting Nmap 7.80 ( https://nmap.org ) at 2020-06-14 23:59 CEST
Nmap scan report for 10.129.2.18
Host is up (0.087s latency).
MAC Address: DE:AD:00:00:BE:EF
Nmap done: 1 IP address (1 host up) scanned in 0.11 seconds
```

#### See why Nmap used ARP

On a local subnet, Nmap often prefers ARP over ICMP.

```bash
sudo nmap 10.129.2.18 -sn -oA host -PE --packet-trace

Starting Nmap 7.80 ( https://nmap.org ) at 2020-06-15 00:08 CEST
SENT (0.0074s) ARP who-has 10.129.2.18 tell 10.10.14.2
RCVD (0.0309s) ARP reply 10.129.2.18 is-at DE:AD:00:00:BE:EF
Nmap scan report for 10.129.2.18
Host is up (0.023s latency).
MAC Address: DE:AD:00:00:BE:EF
Nmap done: 1 IP address (1 host up) scanned in 0.05 seconds
```

Why this happens:

* hosts on the same local subnet must use ARP to resolve MAC addresses
* if the target answers ARP, Nmap already knows the host is alive
* there is no need for a separate ICMP probe

Why it matters:

* ARP-based discovery is very reliable on the local segment
* local firewalls do not usually stop ARP the way they stop ICMP
* results can change once traffic crosses a router
* ARP does not cross routers
* remote discovery falls back to ICMP or TCP-based probes

Flag breakdown:

* `--packet-trace` prints sent and received packets
* `-PE` uses ICMP echo requests for discovery

{% hint style="info" %}
On a local subnet, Nmap may still prefer ARP even when `-PE` is set.
{% endhint %}

#### Use `--reason`

```bash
sudo nmap 10.129.2.18 -sn -oA host -PE --reason

Starting Nmap 7.80 ( https://nmap.org ) at 2020-06-15 00:10 CEST
SENT (0.0074s) ARP who-has 10.129.2.18 tell 10.10.14.2
RCVD (0.0309s) ARP reply 10.129.2.18 is-at DE:AD:00:00:BE:EF
Nmap scan report for 10.129.2.18
Host is up, received arp-response (0.028s latency).
MAC Address: DE:AD:00:00:BE:EF
Nmap done: 1 IP address (1 host up) scanned in 0.03 seconds
```

Key line:

* `Host is up, received arp-response`

#### Force ICMP instead of ARP

```bash
sudo nmap 10.129.2.18 -sn -oA host -PE --packet-trace --disable-arp-ping

Starting Nmap 7.80 ( https://nmap.org ) at 2020-06-15 00:12 CEST
SENT (0.0107s) ICMP [10.10.14.2 > 10.129.2.18 Echo request (type=8/code=0) id=13607 seq=0] IP [ttl=255 id=23541 iplen=28 ]
RCVD (0.0152s) ICMP [10.129.2.18 > 10.10.14.2 Echo reply (type=0/code=0) id=13607 seq=0] IP [ttl=128 id=40622 iplen=28 ]
Nmap scan report for 10.129.2.18
Host is up (0.086s latency).
MAC Address: DE:AD:00:00:BE:EF
Nmap done: 1 IP address (1 host up) scanned in 0.11 seconds
```

This proves the host answered ICMP instead of ARP.

#### Use TTL as a clue

Common starting points:

* `64` for many Linux and Unix systems
* `128` for many Windows systems
* `255` for some network devices

{% hint style="warning" %}
Treat TTL as a clue, not proof. Routing hops and host configuration can change it.
{% endhint %}

#### Alternate host discovery strategies

When firewalls drop ICMP, switch strategies:

* TCP SYN Ping with `-PS`
* TCP ACK Ping with `-PA`
* UDP Ping with `-PU`

Why they help:

* `-PS` can trigger a TCP response even when ICMP is blocked
* `-PA` can trigger a `RST` and prove the host is alive
* `-PU` can trigger ICMP `Port Unreachable`

For large scopes, run host discovery first and scan only the live hosts you confirm.

### Host and port scanning

Once the host is up, map its exposed services.

Focus on:

* open ports and protocols
* service names and versions
* banners and metadata
* operating system clues

Nmap reports six possible port states.

By default, Nmap scans the top `1000` TCP ports.

If you run as root, Nmap usually uses `-sS`.

Without raw socket privileges, Nmap falls back to `-sT`.

Common scope options:

* `-p 22,25,80,139,445`
* `-p 22-445`
* `--top-ports=10`
* `-p-`
* `-F`

{% hint style="info" %}
Use `-p-` when you do not want to miss high-numbered ports.
{% endhint %}

#### Trace a SYN scan

```bash
impale7@htb[/htb]$ sudo nmap 10.129.2.28 -p 21 --packet-trace -Pn -n --disable-arp-ping

Starting Nmap 7.80 ( https://nmap.org ) at 2020-06-15 15:39 CEST
SENT (0.0429s) TCP 10.10.14.2:63090 > 10.129.2.28:21 S ttl=56 id=57322 iplen=44  seq=1699105818 win=1024 <mss 1460>
RCVD (0.0573s) TCP 10.129.2.28:21 > 10.10.14.2:63090 RA ttl=64 id=0 iplen=40  seq=0 win=0
Nmap scan report for 10.129.2.28
Host is up (0.014s latency).

PORT   STATE  SERVICE
21/tcp closed ftp
MAC Address: DE:AD:00:00:BE:EF (Intel Corporate)

Nmap done: 1 IP address (1 host up) scanned in 0.07 seconds
```

Here, the target returns `RST,ACK`.

That means the port is reachable but closed.

#### Use a TCP connect scan

The connect scan with `-sT` completes the full handshake.

It is accurate, but noisy.

If the target returns:

* `SYN-ACK`, the port is `open`
* `RST`, the port is `closed`

```bash
impale7@htb[/htb]$ sudo nmap 10.129.2.28 -p 443 --packet-trace --disable-arp-ping -Pn -n --reason -sT

Starting Nmap 7.80 ( https://nmap.org ) at 2020-06-15 16:26 CET
CONN (0.0385s) TCP localhost > 10.129.2.28:443 => Operation now in progress
CONN (0.0396s) TCP localhost > 10.129.2.28:443 => Connected
Nmap scan report for 10.129.2.28
Host is up, received user-set (0.013s latency).

PORT    STATE SERVICE REASON
443/tcp open  https   syn-ack

Nmap done: 1 IP address (1 host up) scanned in 0.04 seconds
```

#### Understand filtered TCP ports

Two common cases:

* dropped packets
* rejected packets

**Dropped packets**

```bash
impale7@htb[/htb]$ sudo nmap 10.129.2.28 -p 139 --packet-trace -n --disable-arp-ping -Pn

Starting Nmap 7.80 ( https://nmap.org ) at 2020-06-15 15:45 CEST
SENT (0.0381s) TCP 10.10.14.2:60277 > 10.129.2.28:139 S ttl=47 id=14523 iplen=44  seq=4175236769 win=1024 <mss 1460>
SENT (1.0411s) TCP 10.10.14.2:60278 > 10.129.2.28:139 S ttl=45 id=7372 iplen=44  seq=4175171232 win=1024 <mss 1460>
Nmap scan report for 10.129.2.28
Host is up.

PORT    STATE    SERVICE
139/tcp filtered netbios-ssn
MAC Address: DE:AD:00:00:BE:EF (Intel Corporate)

Nmap done: 1 IP address (1 host up) scanned in 2.06 seconds
```

Key flags:

* `-p 139`
* `--packet-trace`
* `-n`
* `--disable-arp-ping`
* `-Pn`

**Rejected packets**

```bash
impale7@htb[/htb]$ sudo nmap 10.129.2.28 -p 445 --packet-trace -n --disable-arp-ping -Pn

Starting Nmap 7.80 ( https://nmap.org ) at 2020-06-15 15:55 CEST
SENT (0.0388s) TCP 10.129.2.28:52472 > 10.129.2.28:445 S ttl=49 id=21763 iplen=44  seq=1418633433 win=1024 <mss 1460>
RCVD (0.0487s) ICMP [10.129.2.28 > 10.129.2.28 Port 445 unreachable (type=3/code=3) ] IP [ttl=64 id=20998 iplen=72 ]
Nmap scan report for 10.129.2.28
Host is up (0.0099s latency).

PORT    STATE    SERVICE
445/tcp filtered microsoft-ds
MAC Address: DE:AD:00:00:BE:EF (Intel Corporate)

Nmap done: 1 IP address (1 host up) scanned in 0.05 seconds
```

An ICMP `type 3 code 3` response is a strong firewall clue when the host is already known to be alive.

#### Discover open UDP ports

UDP scanning uses `-sU`.

It is slower because UDP has no handshake.

Many services do not reply to empty probes.

That means Nmap often waits for timeouts.

```bash
impale7@htb[/htb]$ sudo nmap 10.129.2.28 -F -sU

Starting Nmap 7.80 ( https://nmap.org ) at 2020-06-15 16:01 CEST
Nmap scan report for 10.129.2.28
Host is up (0.059s latency).
Not shown: 95 closed ports
PORT     STATE         SERVICE
68/udp   open|filtered dhcpc
137/udp  open          netbios-ns
138/udp  open|filtered netbios-dgm
631/udp  open|filtered ipp
5353/udp open          zeroconf
MAC Address: DE:AD:00:00:BE:EF (Intel Corporate)

Nmap done: 1 IP address (1 host up) scanned in 98.07 seconds
```

Key flags:

* `-F` scans the top `100` ports
* `-sU` runs a UDP scan

{% hint style="warning" %}
UDP scans are often slow. Start small unless you have a reason to go broad.
{% endhint %}

**Open UDP port**

```bash
impale7@htb[/htb]$ sudo nmap 10.129.2.28 -sU -Pn -n --disable-arp-ping --packet-trace -p 137 --reason

Starting Nmap 7.80 ( https://nmap.org ) at 2020-06-15 16:15 CEST
SENT (0.0367s) UDP 10.10.14.2:55478 > 10.129.2.28:137 ttl=57 id=9122 iplen=78
RCVD (0.0398s) UDP 10.129.2.28:137 > 10.10.14.2:55478 ttl=64 id=13222 iplen=257
Nmap scan report for 10.129.2.28
Host is up, received user-set (0.0031s latency).

PORT    STATE SERVICE    REASON
137/udp open  netbios-ns udp-response ttl 64
MAC Address: DE:AD:00:00:BE:EF (Intel Corporate)

Nmap done: 1 IP address (1 host up) scanned in 0.04 seconds
```

**Closed UDP port**

```bash
impale7@htb[/htb]$ sudo nmap 10.129.2.28 -sU -Pn -n --disable-arp-ping --packet-trace -p 100 --reason

Starting Nmap 7.80 ( https://nmap.org ) at 2020-06-15 16:25 CEST
SENT (0.0445s) UDP 10.10.14.2:63825 > 10.129.2.28:100 ttl=57 id=29925 iplen=28
RCVD (0.1498s) ICMP [10.129.2.28 > 10.10.14.2 Port unreachable (type=3/code=3) ] IP [ttl=64 id=11903 iplen=56 ]
Nmap scan report for 10.129.2.28
Host is up, received user-set (0.11s latency).

PORT    STATE  SERVICE REASON
100/udp closed unknown port-unreach ttl 64
MAC Address: DE:AD:00:00:BE:EF (Intel Corporate)

Nmap done: 1 IP address (1 host up) scanned in  0.15 seconds
```

**Open or filtered UDP port**

```bash
impale7@htb[/htb]$ sudo nmap 10.129.2.28 -sU -Pn -n --disable-arp-ping --packet-trace -p 138 --reason

Starting Nmap 7.80 ( https://nmap.org ) at 2020-06-15 16:32 CEST
SENT (0.0380s) UDP 10.10.14.2:52341 > 10.129.2.28:138 ttl=50 id=65159 iplen=28
SENT (1.0392s) UDP 10.10.14.2:52342 > 10.129.2.28:138 ttl=40 id=24444 iplen=28
Nmap scan report for 10.129.2.28
Host is up, received user-set.

PORT    STATE         SERVICE     REASON
138/udp open|filtered netbios-dgm no-response
MAC Address: DE:AD:00:00:BE:EF (Intel Corporate)

Nmap done: 1 IP address (1 host up) scanned in 2.06 seconds
```

#### Detect service versions during scanning

```bash
impale7@htb[/htb]$ sudo nmap 10.129.2.28 -Pn -n --disable-arp-ping --packet-trace -p 445 --reason -sV

Starting Nmap 7.80 ( https://nmap.org ) at 2022-11-04 11:10 GMT
SENT (0.3426s) TCP 10.10.14.2:44641 > 10.129.2.28:445 S ttl=55 id=43401 iplen=44  seq=3589068008 win=1024 <mss 1460>
RCVD (0.3556s) TCP 10.129.2.28:445 > 10.10.14.2:44641 SA ttl=63 id=0 iplen=44  seq=2881527699 win=29200 <mss 1337>
NSOCK INFO [0.4980s] nsock_iod_new2(): nsock_iod_new (IOD #1)
NSOCK INFO [0.4980s] nsock_connect_tcp(): TCP connection requested to 10.129.2.28:445 (IOD #1) EID 8
NSOCK INFO [0.5130s] nsock_trace_handler_callback(): Callback: CONNECT SUCCESS for EID 8 [10.129.2.28:445]
Service scan sending probe NULL to 10.129.2.28:445 (tcp)
NSOCK INFO [0.5130s] nsock_read(): Read request from IOD #1 [10.129.2.28:445] (timeout: 6000ms) EID 18
NSOCK INFO [6.5190s] nsock_trace_handler_callback(): Callback: READ TIMEOUT for EID 18 [10.129.2.28:445]
Service scan sending probe SMBProgNeg to 10.129.2.28:445 (tcp)
NSOCK INFO [6.5190s] nsock_write(): Write request for 168 bytes to IOD #1 EID 27 [10.129.2.28:445]
NSOCK INFO [6.5190s] nsock_read(): Read request from IOD #1 [10.129.2.28:445] (timeout: 5000ms) EID 34
NSOCK INFO [6.5190s] nsock_trace_handler_callback(): Callback: WRITE SUCCESS for EID 27 [10.129.2.28:445]
NSOCK INFO [6.5320s] nsock_trace_handler_callback(): Callback: READ SUCCESS for EID 34 [10.129.2.28:445] (135 bytes)
Service scan match (Probe SMBProgNeg matched with SMBProgNeg line 13836): 10.129.2.28:445 is netbios-ssn.  Version: |Samba smbd|3.X - 4.X|workgroup: WORKGROUP|
NSOCK INFO [6.5320s] nsock_iod_delete(): nsock_iod_delete (IOD #1)
Nmap scan report for 10.129.2.28
Host is up, received user-set (0.013s latency).

PORT    STATE SERVICE     REASON         VERSION
445/tcp open  netbios-ssn syn-ack ttl 63 Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
Service Info: Host: Ubuntu

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 6.55 seconds
```

Key flags:

* `-Pn`
* `-n`
* `--disable-arp-ping`
* `--packet-trace`
* `-p 445`
* `--reason`
* `-sV`

### Save the results

Always save scan output.

Useful formats:

* `-oN` for normal output
* `-oG` for grepable output
* `-oX` for XML
* `-oA` for all three

#### Save all formats at once

```bash
impale7@htb[/htb]$ sudo nmap 10.129.2.28 -p- -oA target

Starting Nmap 7.80 ( https://nmap.org ) at 2020-06-16 12:14 CEST
Nmap scan report for 10.129.2.28
Host is up (0.0091s latency).
Not shown: 65525 closed ports
PORT      STATE SERVICE
22/tcp    open  ssh
25/tcp    open  smtp
80/tcp    open  http
MAC Address: DE:AD:00:00:BE:EF (Intel Corporate)

Nmap done: 1 IP address (1 host up) scanned in 10.22 seconds
```

What it does:

* `10.129.2.28` targets the host
* `-p-` scans all TCP ports
* `-oA target` writes `target.nmap`, `target.gnmap`, and `target.xml`

#### Check the generated files

```bash
impale7@htb[/htb]$ ls

target.gnmap  target.nmap  target.xml
```

#### Normal output

```bash
impale7@htb[/htb]$ cat target.nmap

# Nmap 7.80 scan initiated Tue Jun 16 12:14:53 2020 as: nmap -p- -oA target 10.129.2.28
Nmap scan report for 10.129.2.28
Host is up (0.053s latency).
Not shown: 4 closed ports
PORT   STATE SERVICE
22/tcp open  ssh
25/tcp open  smtp
80/tcp open  http
MAC Address: DE:AD:00:00:BE:EF (Intel Corporate)

# Nmap done at Tue Jun 16 12:15:03 2020 -- 1 IP address (1 host up) scanned in 10.22 seconds
```

#### Grepable output

```bash
impale7@htb[/htb]$ cat target.gnmap

# Nmap 7.80 scan initiated Tue Jun 16 12:14:53 2020 as: nmap -p- -oA target 10.129.2.28
Host: 10.129.2.28 ()    Status: Up
Host: 10.129.2.28 ()    Ports: 22/open/tcp//ssh///, 25/open/tcp//smtp///, 80/open/tcp//http///  Ignored State: closed (4)
# Nmap done at Tue Jun 16 12:14:53 2020 -- 1 IP address (1 host up) scanned in 10.22 seconds
```

{% hint style="info" %}
Grepable output is good for quick shell parsing. XML is better for structured automation.
{% endhint %}

#### XML output

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE nmaprun>
<?xml-stylesheet href="file:///usr/local/bin/../share/nmap/nmap.xsl" type="text/xsl"?>
<!-- Nmap 7.80 scan initiated Tue Jun 16 12:14:53 2020 as: nmap -p- -oA target 10.129.2.28 -->
<nmaprun scanner="nmap" args="nmap -p- -oA target 10.129.2.28" start="12145301719" startstr="Tue Jun 16 12:15:03 2020" version="7.80" xmloutputversion="1.04">
<scaninfo type="syn" protocol="tcp" numservices="65535" services="1-65535"/>
<verbose level="0"/>
<debugging level="0"/>
<host starttime="12145301719" endtime="12150323493"><status state="up" reason="arp-response" reason_ttl="0"/>
<address addr="10.129.2.28" addrtype="ipv4"/>
<address addr="DE:AD:00:00:BE:EF" addrtype="mac" vendor="Intel Corporate"/>
<hostnames>
</hostnames>
<ports><extraports state="closed" count="4">
<extrareasons reason="resets" count="4"/>
</extraports>
<port protocol="tcp" portid="22"><state state="open" reason="syn-ack" reason_ttl="64"/><service name="ssh" method="table" conf="3"/></port>
<port protocol="tcp" portid="25"><state state="open" reason="syn-ack" reason_ttl="64"/><service name="smtp" method="table" conf="3"/></port>
<port protocol="tcp" portid="80"><state state="open" reason="syn-ack" reason_ttl="64"/><service name="http" method="table" conf="3"/></port>
</ports>
<times srtt="52614" rttvar="75640" to="355174"/>
</host>
<runstats><finished time="12150323493" timestr="Tue Jun 16 12:14:53 2020" elapsed="10.22" summary="Nmap done at Tue Jun 16 12:15:03 2020; 1 IP address (1 host up) scanned in 10.22 seconds" exit="success"/><hosts up="1" down="0" total="1"/>
</runstats>
</nmaprun>
```

#### Convert XML to HTML

```bash
impale7@htb[/htb]$ xsltproc target.xml -o target.html
```

### Service enumeration

Use service enumeration to confirm what is actually running on a port.

It gives you:

* product names
* version numbers
* hostnames and OS clues
* better exploit search terms

#### Scan all TCP ports with version detection

```bash
impale7@htb[/htb]$ sudo nmap 10.129.2.28 -p- -sV

Starting Nmap 7.80 ( https://nmap.org ) at 2020-06-15 19:44 CEST
[Space Bar]
Stats: 0:00:03 elapsed; 0 hosts completed (1 up), 1 undergoing SYN Stealth Scan
SYN Stealth Scan Timing: About 3.64% done; ETC: 19:45 (0:00:53 remaining)
```

What it does:

* `10.129.2.28` targets the host
* `-p-` scans all TCP ports
* `-sV` runs service version detection

{% hint style="info" %}
Press the space bar during a running scan to print progress without stopping it.
{% endhint %}

#### Show regular status updates

```bash
impale7@htb[/htb]$ sudo nmap 10.129.2.28 -p- -sV --stats-every=5s

Starting Nmap 7.80 ( https://nmap.org ) at 2020-06-15 19:46 CEST
Stats: 0:00:05 elapsed; 0 hosts completed (1 up), 1 undergoing SYN Stealth Scan
SYN Stealth Scan Timing: About 13.91% done; ETC: 19:49 (0:00:31 remaining)
Stats: 0:00:10 elapsed; 0 hosts completed (1 up), 1 undergoing SYN Stealth Scan
SYN Stealth Scan Timing: About 39.57% done; ETC: 19:48 (0:00:15 remaining)
```

#### Increase verbosity

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

#### Review the final service results

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

Nmap mainly identifies services from banners.

If banners are incomplete, it falls back to signature matching.

#### Trace version detection

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

* `-p-`
* `-sV`
* `-Pn`
* `-n`
* `--disable-arp-ping`
* `--packet-trace`

Most useful line:

* `220 inlane ESMTP Postfix (Ubuntu)`

That extra banner data reveals `Ubuntu`, even when the summary only shows `Postfix smtpd`.

#### Grab a banner manually

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
### Review the packets

```bash
18:28:07.128564 IP 10.10.14.2.59618 > 10.129.2.28.smtp: Flags [S], seq 1798872233, win 65535, options [mss 1460,nop,wscale 6,nop,nop,TS val 331260178 ecr 0,sackOK,eol], length 0
18:28:07.255151 IP 10.129.2.28.smtp > 10.10.14.2.59618: Flags [S.], seq 1130574379, ack 1798872234, win 65160, options [mss 1460,sackOK,TS val 1800383922 ecr 331260178,nop,wscale 7], length 0
18:28:07.255281 IP 10.10.14.2.59618 > 10.129.2.28.smtp: Flags [.], ack 1, win 2058, options [nop,nop,TS val 331260304 ecr 1800383922], length 0
18:28:07.319306 IP 10.129.2.28.smtp > 10.10.14.2.59618: Flags [P.], seq 1:36, ack 1, win 510, options [nop,nop,TS val 1800383985 ecr 331260304], length 35: SMTP: 220 inlane ESMTP Postfix (Ubuntu)
18:28:07.319426 IP 10.10.14.2.59618 > 10.129.2.28.smtp: Flags [.], ack 36, win 2058, options [nop,nop,TS val 331260368 ecr 1800383985], length 0
```
{% endstep %}
{% endstepper %}

Handshake summary:

1. `[SYN]`
2. `[SYN-ACK]`
3. `[ACK]`
4. `[PSH-ACK]` with banner data
5. `[ACK]`

### NSE

NSE stands for **Nmap Scripting Engine**.

Use it when a basic port scan is not enough.

It helps you:

* grab banners and metadata
* enumerate service features
* run targeted checks by category
* spot likely vulnerabilities faster

{% hint style="warning" %}
Categories like `intrusive`, `exploit`, `brute`, `dos`, and some `vuln` checks can be noisy or risky. Use them only when scope allows it.
{% endhint %}

#### NSE categories

* `auth` — authentication and credential behavior
* `broadcast` — host discovery by broadcast
* `brute` — credential brute force
* `default` — the safe default set behind `-sC`
* `discovery` — host and service information gathering
* `dos` — denial-of-service testing
* `exploit` — known exploit paths
* `external` — external-service processing
* `fuzzer` — unusual input testing
* `intrusive` — checks that may affect the target
* `malware` — malware indicators
* `safe` — lower-risk checks
* `version` — deeper version detection
* `vuln` — known vulnerability checks

#### Run NSE

Run default scripts:

```bash
impale7@htb[/htb]$ sudo nmap <target> -sC
```

Run a category:

```bash
impale7@htb[/htb]$ sudo nmap <target> --script <category>
```

Run specific scripts:

```bash
impale7@htb[/htb]$ sudo nmap <target> --script <script-name>,<script-name>
```

#### Example — banner and SMTP commands

```bash
impale7@htb[/htb]$ sudo nmap 10.129.2.28 -p 25 --script banner,smtp-commands

Starting Nmap 7.80 ( https://nmap.org ) at 2020-06-16 23:21 CEST
Nmap scan report for 10.129.2.28
Host is up (0.050s latency).

PORT   STATE SERVICE
25/tcp open  smtp
|_banner: 220 inlane ESMTP Postfix (Ubuntu)
|_smtp-commands: inlane, PIPELINING, SIZE 10240000, VRFY, ETRN, STARTTLS, ENHANCEDSTATUSCODES, 8BITMIME, DSN, SMTPUTF8,
MAC Address: DE:AD:00:00:BE:EF (Intel Corporate)
```

What you learn:

* `banner` reveals `Postfix` and the `Ubuntu` clue
* `smtp-commands` shows supported SMTP commands
* supported commands can help with user enumeration paths

#### Use aggressive mode carefully

`-A` combines:

* `-sV`
* `-O`
* `--traceroute`
* `-sC`

```bash
impale7@htb[/htb]$ sudo nmap 10.129.2.28 -p 80 -A

Starting Nmap 7.80 ( https://nmap.org ) at 2020-06-17 01:38 CEST
Nmap scan report for 10.129.2.28
Host is up (0.012s latency).

PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-generator: WordPress 5.3.4
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: blog.inlanefreight.com
MAC Address: DE:AD:00:00:BE:EF (Intel Corporate)
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Aggressive OS guesses: Linux 2.6.32 (96%), Linux 3.2 - 4.9 (96%), Linux 2.6.32 - 3.10 (96%), Linux 3.4 - 3.10 (95%), Linux 3.1 (95%), Linux 3.2 (95%),
AXIS 210A or 211 Network Camera (Linux 2.6.17) (94%), Synology DiskStation Manager 5.2-5644 (94%), Netgear RAIDiator 4.2.28 (94%),
Linux 2.6.32 - 2.6.35 (94%)
No exact OS matches for host (test conditions non-ideal).
Network Distance: 1 hop

TRACEROUTE
HOP RTT      ADDRESS
1   11.91 ms 10.129.2.28

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 11.36 seconds
```

What you learn:

* Apache `2.4.29`
* WordPress `5.3.4`
* page title `blog.inlanefreight.com`
* likely Linux host

{% hint style="info" %}
`-A` is useful for quick triage. It is louder than a focused scan.
{% endhint %}

#### Use the `vuln` category

```bash
impale7@htb[/htb]$ sudo nmap 10.129.2.28 -p 80 -sV --script vuln

Nmap scan report for 10.129.2.28
Host is up (0.036s latency).

PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
| http-enum:
|   /wp-login.php: Possible admin folder
|   /readme.html: Wordpress version: 2
|   /: WordPress version: 5.3.4
|   /wp-includes/images/rss.png: Wordpress version 2.2 found.
|   /wp-includes/js/jquery/suggest.js: Wordpress version 2.5 found.
|   /wp-includes/images/blank.gif: Wordpress version 2.6 found.
|   /wp-includes/js/comment-reply.js: Wordpress version 2.7 found.
|   /wp-login.php: Wordpress login page.
|   /wp-admin/upgrade.php: Wordpress login page.
|_  /readme.html: Interesting, a readme.
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-stored-xss: Couldn't find any stored XSS vulnerabilities.
| http-wordpress-users:
| Username found: admin
|_Search stopped at ID #25. Increase the upper limit if necessary with 'http-wordpress-users.limit'
| vulners:
|   cpe:/a:apache:http_server:2.4.29:
|       CVE-2019-0211   7.2 https://vulners.com/cve/CVE-2019-0211
|       CVE-2018-1312   6.8 https://vulners.com/cve/CVE-2018-1312
|       CVE-2017-15715  6.8 https://vulners.com/cve/CVE-2017-15715
<SNIP>
```

What you learn:

* likely WordPress paths and version clues
* usernames such as `admin`
* possible CVEs tied to the detected service

### Performance tuning

Tune performance when:

* the target range is large
* bandwidth is limited
* latency is high
* you need faster feedback

Useful controls:

* `-T <0-5>`
* `--max-retries`
* `--initial-rtt-timeout`
* `--max-rtt-timeout`
* `--min-rate`

{% hint style="warning" %}
Faster scans are not always better. Aggressive tuning can miss hosts, ports, or version data.
{% endhint %}

#### Tune RTT timeouts

Default:

```bash
impale7@htb[/htb]$ sudo nmap 10.129.2.0/24 -F

<SNIP>
Nmap done: 256 IP addresses (10 hosts up) scanned in 39.44 seconds
```

Reduced RTT values:

```bash
impale7@htb[/htb]$ sudo nmap 10.129.2.0/24 -F --initial-rtt-timeout 50ms --max-rtt-timeout 100ms

<SNIP>
Nmap done: 256 IP addresses (8 hosts up) scanned in 12.29 seconds
```

What to notice:

* the scan finishes much faster
* two hosts are missed
* short RTT values can hide slower systems

#### Reduce max retries

Default:

```bash
impale7@htb[/htb]$ sudo nmap 10.129.2.0/24 -F | grep "/tcp" | wc -l

23
```

Retry limit `0`:

```bash
impale7@htb[/htb]$ sudo nmap 10.129.2.0/24 -F --max-retries 0 | grep "/tcp" | wc -l

21
```

What to notice:

* the scan is faster
* two open ports are missed here
* low retries fit stable, low-loss networks best

#### Increase packet rate

Default:

```bash
impale7@htb[/htb]$ sudo nmap 10.129.2.0/24 -F -oN tnet.default

<SNIP>
Nmap done: 256 IP addresses (10 hosts up) scanned in 29.83 seconds
```

Higher minimum rate:

```bash
impale7@htb[/htb]$ sudo nmap 10.129.2.0/24 -F -oN tnet.minrate300 --min-rate 300

<SNIP>
Nmap done: 256 IP addresses (10 hosts up) scanned in 8.67 seconds
```

Compare counts:

```bash
impale7@htb[/htb]$ cat tnet.default | grep "/tcp" | wc -l

23
```

```bash
impale7@htb[/htb]$ cat tnet.minrate300 | grep "/tcp" | wc -l

23
```

What to notice:

* the faster scan still finds the same open ports
* `--min-rate` can speed things up without losing coverage
* it works best on stable networks that are not rate-limiting you

#### Use timing templates

Templates:

* `-T 0` or `-T paranoid`
* `-T 1` or `-T sneaky`
* `-T 2` or `-T polite`
* `-T 3` or `-T normal`
* `-T 4` or `-T aggressive`
* `-T 5` or `-T insane`

`-T 3` is the default.

Higher values push speed harder and increase noise.

Default timing:

```bash
impale7@htb[/htb]$ sudo nmap 10.129.2.0/24 -F -oN tnet.default

<SNIP>
Nmap done: 256 IP addresses (10 hosts up) scanned in 32.44 seconds
```

`-T 5` timing:

```bash
impale7@htb[/htb]$ sudo nmap 10.129.2.0/24 -F -oN tnet.T5 -T 5

<SNIP>
Nmap done: 256 IP addresses (10 hosts up) scanned in 18.07 seconds
```

Compare counts:

```bash
impale7@htb[/htb]$ cat tnet.default | grep "/tcp" | wc -l

23
```

```bash
impale7@htb[/htb]$ cat tnet.T5 | grep "/tcp" | wc -l

23
```

{% hint style="info" %}
Start with `-T 3` or `-T 4` unless you have a strong reason to go faster. Use `-T 5` carefully.
{% endhint %}

Fast decision rules:

* If the network is slow, tune RTT before pushing rate.
* If packets drop often, avoid very low retries.
* If you are whitelisted, `--min-rate` can save time quickly.
* If you need a simple speed boost, try `-T 4` first.

### Firewall and IDS/IPS evasion

Use these options when normal scans return `filtered` or hit monitoring controls.

#### Determine firewall behavior

Filtered traffic is usually:

* dropped
* rejected

Common ICMP errors:

* `Net Unreachable`
* `Net Prohibited`
* `Host Unreachable`
* `Host Prohibited`
* `Port Unreachable`
* `Proto Unreachable`

#### Compare SYN and ACK scans

SYN scan:

```bash
impale7@htb[/htb]$ sudo nmap 10.129.2.28 -p 21,22,25 -sS -Pn -n --disable-arp-ping --packet-trace
```

ACK scan:

```bash
impale7@htb[/htb]$ sudo nmap 10.129.2.28 -p 21,22,25 -sA -Pn -n --disable-arp-ping --packet-trace
```

Use `-sA` when you want to test whether a firewall is filtering traffic rather than prove a port is open.

#### Detect IDS and IPS

Useful clues:

* aggressive scans against one service may trigger an operator response
* one `VPS` IP getting blocked suggests active controls
* switching to another `VPS` can confirm the behavior

#### Use decoys

```bash
impale7@htb[/htb]$ sudo nmap 10.129.2.28 -p 80 -sS -Pn -n --disable-arp-ping --packet-trace -D RND:5
```

Use `-D RND:5` to add five random decoys.

Decoys must be alive.

Spoofed packets are often filtered by ISPs and routers.

#### Test a different source IP

Test the rule:

```bash
impale7@htb[/htb]$ sudo nmap 10.129.2.28 -n -Pn -p445 -O
```

Use a different source:

```bash
impale7@htb[/htb]$ sudo nmap 10.129.2.28 -n -Pn -p 445 -O -S 10.129.2.200 -e tun0
```

Useful flags:

* `-O`
* `-S`
* `-e tun0`

#### Use DNS servers and source port `53`

Useful options:

* `--dns-server <ns>,<ns>`
* `--source-port 53`

Filtered port:

```bash
impale7@htb[/htb]$ sudo nmap 10.129.2.28 -p50000 -sS -Pn -n --disable-arp-ping --packet-trace
```

Same scan from source port `53`:

```bash
impale7@htb[/htb]$ sudo nmap 10.129.2.28 -p50000 -sS -Pn -n --disable-arp-ping --packet-trace --source-port 53
```

Test the access with `ncat`:

```bash
impale7@htb[/htb]$ ncat -nv --source-port 53 10.129.2.28 50000

Ncat: Version 7.80 ( https://nmap.org/ncat )
Ncat: Connected to 10.129.2.28:50000.
220 ProFTPd
```

### Practice prompts

* Find all TCP ports on your target. Submit the total number of found TCP ports.
* Enumerate the hostname of your target and submit it exactly.

### Further reading

* [Nmap host discovery guide](https://nmap.org/book/host-discovery-strategies.html)
* [Nmap port scanning guide](https://nmap.org/book/man-port-scanning-techniques.html)
* [Nmap output guide](https://nmap.org/book/output.html)
* [Nmap version detection guide](https://nmap.org/book/man-version-detection.html)
* [NSE script reference](https://nmap.org/nsedoc/index.html)
* [Nmap performance guide](https://nmap.org/book/man-performance.html)
* [Nmap timing template reference](https://nmap.org/book/performance-timing-templates.html)
* [Service Enumeration](../cpts-modules/information-gathering/network-enumeration-with-nmap/service-enumeration-using-nmap.md)
