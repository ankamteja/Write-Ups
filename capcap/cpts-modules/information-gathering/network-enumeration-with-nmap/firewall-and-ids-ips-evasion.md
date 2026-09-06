# Firewall and IDS/IPS Evasion

Use this page when a standard scan returns `filtered`, gets dropped, or starts hitting monitoring controls.

These techniques help you tell whether traffic is blocked, ignored, or selectively trusted.

### Understand the controls

#### Firewalls

A firewall monitors traffic between networks and applies rules to each connection attempt.

It decides whether to pass, ignore, or block individual packets.

That is how it prevents unwanted or dangerous connections from external networks.

#### IDS and IPS

`IDS` and `IPS` are also software-based controls.

`IDS` inspects traffic, detects suspicious patterns, and reports them.

`IPS` builds on that and can block or disrupt traffic automatically.

Pattern matching matters here.

If a scan looks like service detection or active reconnaissance, an `IPS` may stop it before you finish.

### Determine firewall behavior

When Nmap shows a port as `filtered`, the firewall usually does one of two things:

* `drop` the packet and send nothing back
* `reject` the packet and send an explicit response

Rejected traffic often returns:

* a TCP `RST`
* or an ICMP error such as:
  * `Net Unreachable`
  * `Net Prohibited`
  * `Host Unreachable`
  * `Host Prohibited`
  * `Port Unreachable`
  * `Proto Unreachable`

#### Compare SYN and ACK scans

An ACK scan with `-sA` is often harder to filter than a normal SYN scan with `-sS` or a connect scan with `-sT`.

That is because it sends only an `ACK` packet.

If the target port is open or closed, the host must answer with `RST`.

Many firewalls block new inbound `SYN` packets by default.

They often pass `ACK` packets because they cannot easily tell whether the connection started externally or internally.

**SYN scan**

```bash
impale7@htb[/htb]$ sudo nmap 10.129.2.28 -p 21,22,25 -sS -Pn -n --disable-arp-ping --packet-trace

Starting Nmap 7.80 ( https://nmap.org ) at 2020-06-21 14:56 CEST
SENT (0.0278s) TCP 10.10.14.2:57347 > 10.129.2.28:22 S ttl=53 id=22412 iplen=44  seq=4092255222 win=1024 <mss 1460>
SENT (0.0278s) TCP 10.10.14.2:57347 > 10.129.2.28:25 S ttl=50 id=62291 iplen=44  seq=4092255222 win=1024 <mss 1460>
SENT (0.0278s) TCP 10.10.14.2:57347 > 10.129.2.28:21 S ttl=58 id=38696 iplen=44  seq=4092255222 win=1024 <mss 1460>
RCVD (0.0329s) ICMP [10.129.2.28 > 10.10.14.2 Port 21 unreachable (type=3/code=3) ] IP [ttl=64 id=40884 iplen=72 ]
RCVD (0.0341s) TCP 10.129.2.28:22 > 10.10.14.2:57347 SA ttl=64 id=0 iplen=44  seq=1153454414 win=64240 <mss 1460>
RCVD (1.0386s) TCP 10.129.2.28:22 > 10.10.14.2:57347 SA ttl=64 id=0 iplen=44  seq=1153454414 win=64240 <mss 1460>
SENT (1.1366s) TCP 10.10.14.2:57348 > 10.129.2.28:25 S ttl=44 id=6796 iplen=44  seq=4092320759 win=1024 <mss 1460>
Nmap scan report for 10.129.2.28
Host is up (0.0053s latency).

PORT   STATE    SERVICE
21/tcp filtered ftp
22/tcp open     ssh
25/tcp filtered smtp
MAC Address: DE:AD:00:00:BE:EF (Intel Corporate)

Nmap done: 1 IP address (1 host up) scanned in 0.07 seconds
```

**ACK scan**

```bash
impale7@htb[/htb]$ sudo nmap 10.129.2.28 -p 21,22,25 -sA -Pn -n --disable-arp-ping --packet-trace

Starting Nmap 7.80 ( https://nmap.org ) at 2020-06-21 14:57 CEST
SENT (0.0422s) TCP 10.10.14.2:49343 > 10.129.2.28:21 A ttl=49 id=12381 iplen=40  seq=0 win=1024
SENT (0.0423s) TCP 10.10.14.2:49343 > 10.129.2.28:22 A ttl=41 id=5146 iplen=40  seq=0 win=1024
SENT (0.0423s) TCP 10.10.14.2:49343 > 10.129.2.28:25 A ttl=49 id=5800 iplen=40  seq=0 win=1024
RCVD (0.1252s) ICMP [10.129.2.28 > 10.10.14.2 Port 21 unreachable (type=3/code=3) ] IP [ttl=64 id=55628 iplen=68 ]
RCVD (0.1268s) TCP 10.129.2.28:22 > 10.10.14.2:49343 R ttl=64 id=0 iplen=40  seq=1660784500 win=0
SENT (1.3837s) TCP 10.10.14.2:49344 > 10.129.2.28:25 A ttl=59 id=21915 iplen=40  seq=0 win=1024
Nmap scan report for 10.129.2.28
Host is up (0.083s latency).

PORT   STATE      SERVICE
21/tcp filtered   ftp
22/tcp unfiltered ssh
25/tcp filtered   smtp
MAC Address: DE:AD:00:00:BE:EF (Intel Corporate)

Nmap done: 1 IP address (1 host up) scanned in 0.15 seconds
```

Flag breakdown:

* `10.129.2.28` targets the host
* `-p 21,22,25` scans only the listed ports
* `-sS` runs a SYN scan
* `-sA` runs an ACK scan
* `-Pn` skips ICMP echo discovery
* `-n` disables DNS resolution
* `--disable-arp-ping` disables ARP discovery
* `--packet-trace` shows sent and received packets

What to look for:

* In the SYN scan, TCP `22` replies with `SA`, so the port is `open`.
* In the ACK scan, TCP `22` replies with `RST`, so the port is `unfiltered`.
* TCP `25` sends no reply, which suggests packets are being dropped.

### Detect IDS and IPS

Firewalls are easier to spot than `IDS` or `IPS`.

`IDS` is passive.

It watches traffic and alerts an administrator when traffic matches a rule or signature.

`IPS` is active.

It can apply defensive actions automatically.

A practical way to test for these controls is to scan from multiple `VPS` IPs.

If one host gets blocked during the assessment, you have a strong sign that an administrator or `IPS` reacted to the activity.

Two working clues:

* `IDS` alone may trigger an operator response after aggressive scans against one port or service.
* `IPS` may block the scanning host entirely, which tells you the network is enforcing defensive actions automatically.

Once that happens, lower your noise and disguise your interactions more carefully.

### Use decoys

Decoys help when:

* specific regions or subnets are blocked
* an `IPS` is likely to block your source IP

With `-D`, Nmap inserts extra source IP addresses into the packet headers.

You can generate random decoys with `RND`.

For example, `RND:5` adds five random IP addresses.

Your real IP is placed randomly among them.

{% hint style="warning" %}
The decoys must be alive. Dead decoys can make the target service unreachable because of SYN-flood protections.
{% endhint %}

#### Scan with decoys

```bash
impale7@htb[/htb]$ sudo nmap 10.129.2.28 -p 80 -sS -Pn -n --disable-arp-ping --packet-trace -D RND:5

Starting Nmap 7.80 ( https://nmap.org ) at 2020-06-21 16:14 CEST
SENT (0.0378s) TCP 102.52.161.59:59289 > 10.129.2.28:80 S ttl=42 id=29822 iplen=44  seq=3687542010 win=1024 <mss 1460>
SENT (0.0378s) TCP 10.10.14.2:59289 > 10.129.2.28:80 S ttl=59 id=29822 iplen=44  seq=3687542010 win=1024 <mss 1460>
SENT (0.0379s) TCP 210.120.38.29:59289 > 10.129.2.28:80 S ttl=37 id=29822 iplen=44  seq=3687542010 win=1024 <mss 1460>
SENT (0.0379s) TCP 191.6.64.171:59289 > 10.129.2.28:80 S ttl=38 id=29822 iplen=44  seq=3687542010 win=1024 <mss 1460>
SENT (0.0379s) TCP 184.178.194.209:59289 > 10.129.2.28:80 S ttl=39 id=29822 iplen=44  seq=3687542010 win=1024 <mss 1460>
SENT (0.0379s) TCP 43.21.121.33:59289 > 10.129.2.28:80 S ttl=55 id=29822 iplen=44  seq=3687542010 win=1024 <mss 1460>
RCVD (0.1370s) TCP 10.129.2.28:80 > 10.10.14.2:59289 SA ttl=64 id=0 iplen=44  seq=4056111701 win=64240 <mss 1460>
Nmap scan report for 10.129.2.28
Host is up (0.099s latency).

PORT   STATE SERVICE
80/tcp open  http
MAC Address: DE:AD:00:00:BE:EF (Intel Corporate)

Nmap done: 1 IP address (1 host up) scanned in 0.15 seconds
```

Flag breakdown:

* `10.129.2.28` targets the host
* `-p 80` scans only TCP `80`
* `-sS` runs a SYN scan
* `-Pn` skips ICMP echo discovery
* `-n` disables DNS resolution
* `--disable-arp-ping` disables ARP discovery
* `--packet-trace` shows sent and received packets
* `-D RND:5` adds five random decoy IP addresses

Spoofed packets are often filtered by ISPs and routers.

If that happens, use known `VPS` IP addresses instead.

You can also test a different source IP with `-S`.

That is useful when only some subnets can reach the service.

Decoys work with SYN, ACK, ICMP, and OS detection scans.

#### Test a firewall rule

```bash
impale7@htb[/htb]$ sudo nmap 10.129.2.28 -n -Pn -p445 -O

Starting Nmap 7.80 ( https://nmap.org ) at 2020-06-22 01:23 CEST
Nmap scan report for 10.129.2.28
Host is up (0.032s latency).

PORT    STATE    SERVICE
445/tcp filtered microsoft-ds
MAC Address: DE:AD:00:00:BE:EF (Intel Corporate)
Too many fingerprints match this host to give specific OS details
Network Distance: 1 hop

OS detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 3.14 seconds
```

#### Scan with a different source IP

```bash
impale7@htb[/htb]$ sudo nmap 10.129.2.28 -n -Pn -p 445 -O -S 10.129.2.200 -e tun0

Starting Nmap 7.80 ( https://nmap.org ) at 2020-06-22 01:16 CEST
Nmap scan report for 10.129.2.28
Host is up (0.010s latency).

PORT    STATE SERVICE
445/tcp open  microsoft-ds
MAC Address: DE:AD:00:00:BE:EF (Intel Corporate)
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Aggressive OS guesses: Linux 2.6.32 (96%), Linux 3.2 - 4.9 (96%), Linux 2.6.32 - 3.10 (96%), Linux 3.4 - 3.10 (95%), Linux 3.1 (95%), Linux 3.2 (95%), AXIS 210A or 211 Network Camera (Linux 2.6.17) (94%), Synology DiskStation Manager 5.2-5644 (94%), Linux 2.6.32 - 2.6.35 (94%), Linux 2.6.32 - 3.5 (94%)
No exact OS matches for host (test conditions non-ideal).
Network Distance: 1 hop

OS detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 4.11 seconds
```

Flag breakdown:

* `10.129.2.28` targets the host
* `-n` disables DNS resolution
* `-Pn` skips ICMP echo discovery
* `-p 445` scans only TCP `445`
* `-O` runs OS detection
* `-S 10.129.2.200` sets the source IP
* `-e tun0` sends traffic through the specified interface

### Use DNS proxying and trusted source ports

By default, Nmap performs reverse DNS lookups unless you disable them.

Those requests usually go over `UDP 53`.

Historically, `TCP 53` was used mainly for zone transfers or responses larger than `512` bytes.

That is changing with IPv6 and DNSSEC, so more DNS traffic now uses `TCP 53` as well.

Nmap also lets you specify DNS servers directly with `--dns-server <ns>,<ns>`.

That can matter inside a `DMZ`.

Internal company DNS servers are often trusted more than public Internet resolvers.

Another useful trick is `--source-port 53`.

If a firewall trusts traffic from `TCP 53` and `IDS` or `IPS` filters are weak there, your scan may pass where normal probes fail.

#### SYN scan of a filtered port

```bash
impale7@htb[/htb]$ sudo nmap 10.129.2.28 -p50000 -sS -Pn -n --disable-arp-ping --packet-trace

Starting Nmap 7.80 ( https://nmap.org ) at 2020-06-21 22:50 CEST
SENT (0.0417s) TCP 10.10.14.2:33436 > 10.129.2.28:50000 S ttl=41 id=21939 iplen=44  seq=736533153 win=1024 <mss 1460>
SENT (1.0481s) TCP 10.10.14.2:33437 > 10.129.2.28:50000 S ttl=46 id=6446 iplen=44  seq=736598688 win=1024 <mss 1460>
Nmap scan report for 10.129.2.28
Host is up.

PORT      STATE    SERVICE
50000/tcp filtered ibm-db2

Nmap done: 1 IP address (1 host up) scanned in 2.06 seconds
```

#### SYN scan from source port 53

```bash
impale7@htb[/htb]$ sudo nmap 10.129.2.28 -p50000 -sS -Pn -n --disable-arp-ping --packet-trace --source-port 53

SENT (0.0482s) TCP 10.10.14.2:53 > 10.129.2.28:50000 S ttl=58 id=27470 iplen=44  seq=4003923435 win=1024 <mss 1460>
RCVD (0.0608s) TCP 10.129.2.28:50000 > 10.10.14.2:53 SA ttl=64 id=0 iplen=44  seq=540635485 win=64240 <mss 1460>
Nmap scan report for 10.129.2.28
Host is up (0.013s latency).

PORT      STATE SERVICE
50000/tcp open  ibm-db2
MAC Address: DE:AD:00:00:BE:EF (Intel Corporate)

Nmap done: 1 IP address (1 host up) scanned in 0.08 seconds
```

Flag breakdown:

* `10.129.2.28` targets the host
* `-p 50000` scans only TCP `50000`
* `-sS` runs a SYN scan
* `-Pn` skips ICMP echo discovery
* `-n` disables DNS resolution
* `--disable-arp-ping` disables ARP discovery
* `--packet-trace` shows sent and received packets
* `--source-port 53` sends the scan from source port `53`

If `TCP 53` is accepted by the firewall, `IDS` and `IPS` filters may also be weaker there.

You can verify access with `Netcat`.

#### Connect to the filtered port

```bash
impale7@htb[/htb]$ ncat -nv --source-port 53 10.129.2.28 50000

Ncat: Version 7.80 ( https://nmap.org/ncat )
Ncat: Connected to 10.129.2.28:50000.
220 ProFTPd
```
