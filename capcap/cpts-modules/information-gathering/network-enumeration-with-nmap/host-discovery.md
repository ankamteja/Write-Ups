---
description: Discover live hosts with Nmap before deeper enumeration.
---

# Host Discovery

Start by finding which hosts are online.

On internal tests, this is usually the first step.

### Scan a network range

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

<figure><img src="../../../../.gitbook/assets/image (55).png" alt=""><figcaption></figcaption></figure>

What this does:

* `10.129.2.0/24` targets the whole subnet.
* `-sn` skips the port scan and only checks whether hosts respond.
* `-oA tnet` saves output as `tnet.nmap`, `tnet.gnmap`, and `tnet.xml`.

{% hint style="info" %}
This works best when hosts reply to ICMP or other discovery probes.
{% endhint %}

If standard discovery fails, try direct TCP probes against likely ports such as `80` or `445`.

### Scan an IP list

Use a host list when the scope is predefined.

Example list:

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

Scan the list:

```bash
sudo nmap -sn -oA tnet -iL hosts.lst | grep for | cut -d" " -f5

10.129.2.18
10.129.2.19
10.129.2.20
```

<figure><img src="../../../../.gitbook/assets/image (56).png" alt=""><figcaption></figcaption></figure>

If some hosts do not appear, they may be blocking or ignoring the default discovery traffic.

### Scan multiple IPs directly

Use this when you only need a few targets.

```bash
sudo nmap -sn -oA tnet 10.129.2.18 10.129.2.19 10.129.2.20 | grep for | cut -d" " -f5

10.129.2.18
10.129.2.19
10.129.2.20
```

If the addresses are consecutive, use a range:

```bash
sudo nmap -sn -oA tnet 10.129.2.18-20 | grep for | cut -d" " -f5

10.129.2.18
10.129.2.19
10.129.2.20
```

### Scan a single IP

Before you enumerate services, confirm the host is alive.

```bash
sudo nmap 10.129.2.18 -sn -oA host

Starting Nmap 7.80 ( https://nmap.org ) at 2020-06-14 23:59 CEST
Nmap scan report for 10.129.2.18
Host is up (0.087s latency).
MAC Address: DE:AD:00:00:BE:EF
Nmap done: 1 IP address (1 host up) scanned in 0.11 seconds
```

### Why Nmap used ARP instead of ICMP

On a local subnet, Nmap often uses ARP discovery first.

That happens even if you expect an ICMP echo request.

Use `--packet-trace` to see it clearly:

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

<figure><img src="../../../../.gitbook/assets/image (59).png" alt=""><figcaption></figcaption></figure>

### Why this happens

Hosts on the same local subnet must use ARP to resolve MAC addresses.

If the target answers ARP, Nmap already knows the host is alive.

There is no need to send a separate ICMP probe.

### Why it matters

For internal testing:

* ARP-based discovery is very reliable on the local segment.
* Local firewalls do not usually stop ARP in the same way they stop ICMP.
* Results can look different once traffic crosses a router.

For remote networks:

* ARP does not cross routers.
* Nmap must fall back to ICMP or TCP-based discovery.
* Host discovery becomes easier to filter or hide.

### Flag breakdown

* `--packet-trace` prints packets that Nmap sends and receives.
* `-PE` tells Nmap to use ICMP echo requests for discovery.

{% hint style="info" %}
On a local subnet, Nmap may still prefer ARP even when `-PE` is set.
{% endhint %}

<figure><img src="../../../../.gitbook/assets/image (60).png" alt=""><figcaption></figcaption></figure>

### Use `--reason` to verify discovery

Use `--reason` when you want Nmap to explain why it marked a host as up.

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

The key line is:

* `Host is up, received arp-response`

That confirms Nmap treated the ARP reply as proof the host is alive.

<figure><img src="../../../../.gitbook/assets/image (62).png" alt=""><figcaption></figcaption></figure>

### Force ICMP instead of ARP

If you want to test ICMP echo requests specifically, disable ARP discovery first.

Use `--disable-arp-ping` with `-PE`.

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

This time the trace shows:

* an ICMP echo request
* an ICMP echo reply

That proves the host responded to ICMP, not just ARP.

### Use TTL as a clue

ICMP replies can also give you a rough TTL value.

TTL can help you estimate the target OS family.

Common starting points include:

* `64` for many Linux and Unix systems
* `128` for many Windows systems
* `255` for some network devices

{% hint style="warning" %}
Treat TTL as a hint, not proof. Routing hops and host configuration can change it.
{% endhint %}

<figure><img src="../../../../.gitbook/assets/image (63).png" alt=""><figcaption></figcaption></figure>

### 2. Evading Firewalls and IDS Systems

Standard host discovery fails if a firewall drops ICMP packets. If you only know one discovery strategy (ICMP Ping), you will report the host as dead and miss the target completely.

Knowing "more strategies" means you can cycle through alternate host discovery techniques when the front door is locked:

* TCP SYN Ping (`-PS`): Sends a silent connection initialization packet to a common port (like 80 or 443). Even if a firewall blocks ICMP, the target's operating system might automatically answer the TCP request, revealing that the host is alive.
* TCP ACK Ping (`-PA`): Sends a fake acknowledgment packet. The target machine gets confused because it doesn't remember an open connection, so it fires back a `RST` (Reset) packet, completely bypassing basic firewall rules and proving it is powered on.
* UDP Ping (`-PU`): Sends a UDP packet to a highly unlikely port. If the host is up, it returns an ICMP "Port Unreachable" error, exposing its active state.

### 3. Optimizing Your Pentest Window (Speed)

If you drop onto an enterprise network range containing 65,536 IP addresses (a `/16` subnet) and you try to run a full port scan on every single address blindly, your scan will take days, consume massive bandwidth, and alert every blue team alarm on the network.

Mastering host discovery allows you to execute a highly optimized multi-strategy "sweep" first, whittling down those 65,000 possibilities into a clean text file of 20 verified live targets in minutes. You then feed that specific scope list into your deeper service scans, keeping your execution clean, fast, and professional.

That is why the material stresses this phase: The more strategies you know, the less likely you are to be blinded by a simple firewall rule.

### Further reading

See the official [Nmap host discovery guide](https://nmap.org/book/host-discovery-strategies.html).
