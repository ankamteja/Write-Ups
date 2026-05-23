# Network Enumeration with Nmap

Use this page to discover live hosts and understand how Nmap performs host discovery on internal networks.

```
┌────────────────────────┐
│ THE PUBLIC INTERNET    │
│ (External Pentest)     │
└───────────┬────────────┘
            │
       [ Public IP ]
            │
┌───────────┴────────────┐
│ Router and firewall    │
│ NAT blocks inbound     │
└───────────┬────────────┘
            │
      [ Private subnet ]
            │
┌──────────────────────────────────────────────────────────────┐
│ Internal web host │ Domain controller │ Attacker via VPN    │
│ 10.129.2.18       │ 10.129.2.10       │ 10.10.14.5          │
└──────────────────────────────────────────────────────────────┘
```

<figure><img src="../../../../.gitbook/assets/image (61).png" alt=""><figcaption></figcaption></figure>

```
[ HIGH LAYER: Human & Application Data ]
       7. Application  <-- HTTP, SSH, DNS, FTP
       6. Presentation <-- Encryption (SSL/TLS), Compression
       5. Session      <-- Sockets, Tunneling (RPC, NetBIOS)
       
       [ MIDDLE LAYER: Routing & Transport Rules ]
       4. Transport    <-- Port Numbers, Handshakes (TCP, UDP)
       3. Network      <-- IP Addresses, Routing, Errors (IP, ICMP)
       
       [ LOW LAYER: Physical Wires & Hardware ]
       2. Data Link    <-- MAC Addresses, Local Switching (ARP)
       1. Physical     <-- Cables, RJ45, Bits, Electrical Signals
```

## Syntax

Nmap uses this basic format:

```bash
nmap <scan types> <options> <target>
```

`-sS` is one of the most common scan types.

It sends a SYN packet and stops before the full TCP handshake completes.

That makes it fast and relatively quiet.

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

With a SYN scan:

* `SYN-ACK` means the port is `open`.
* `RST` means the port is `closed`.
* No response often means the port is `filtered`.

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

### Child pages

Use these pages for the full walkthroughs:

* [Host Discovery](host-discovery.md)
* [Host and Port Scanning](host-and-port-scanning.md)
