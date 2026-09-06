---
description: Tune Nmap timing, retries, and packet rates without losing useful results.
---

# Performance

Use this page when you need faster Nmap scans without throwing away useful results.

Performance tuning matters most when:

* the target range is large
* bandwidth is limited
* latency is high
* you need faster feedback during enumeration

Nmap lets you tune scan behavior with options like:

* timing templates with `-T <0-5>`
* retry limits with `--max-retries`
* RTT timeouts with `--initial-rtt-timeout` and `--max-rtt-timeout`
* packet rate with `--min-rate`

{% hint style="warning" %}
Faster scans are not always better. Aggressive tuning can miss hosts, ports, or service data.
{% endhint %}

### Tune RTT timeouts

Nmap waits for replies based on the round-trip time, or `RTT`.

If you lower RTT values too much, slow hosts may look dead.

#### Default scan

```bash
impale7@htb[/htb]$ sudo nmap 10.129.2.0/24 -F

<SNIP>
Nmap done: 256 IP addresses (10 hosts up) scanned in 39.44 seconds
```

#### Reduced RTT values

```bash
impale7@htb[/htb]$ sudo nmap 10.129.2.0/24 -F --initial-rtt-timeout 50ms --max-rtt-timeout 100ms

<SNIP>
Nmap done: 256 IP addresses (8 hosts up) scanned in 12.29 seconds
```

What this does:

* `10.129.2.0/24` scans the target subnet
* `-F` scans the top `100` ports
* `--initial-rtt-timeout 50ms` lowers the starting RTT wait time
* `--max-rtt-timeout 100ms` caps the maximum RTT wait time

What to notice:

* the scan finishes much faster
* two hosts are missed
* overly short RTT values can hide slower systems

### Reduce max retries

Nmap retries probes when a target does not answer.

Lower retries make scans faster.

They also reduce confidence in unstable networks.

#### Default retry behavior

```bash
impale7@htb[/htb]$ sudo nmap 10.129.2.0/24 -F | grep "/tcp" | wc -l

23
```

#### Retry limit set to zero

```bash
impale7@htb[/htb]$ sudo nmap 10.129.2.0/24 -F --max-retries 0 | grep "/tcp" | wc -l

21
```

What this does:

* `10.129.2.0/24` scans the target subnet
* `-F` scans the top `100` ports
* `--max-retries 0` stops Nmap from retrying unanswered probes

What to notice:

* the scan becomes more aggressive
* two open ports are missed in this example
* low retries work best on stable, low-loss networks

### Increase packet rate

If the environment allows it, raise the packet rate to speed up discovery.

This is most useful when you know the network can handle the traffic.

That is common in controlled or whitelisted tests.

#### Default scan

```bash
impale7@htb[/htb]$ sudo nmap 10.129.2.0/24 -F -oN tnet.default

<SNIP>
Nmap done: 256 IP addresses (10 hosts up) scanned in 29.83 seconds
```

#### Higher minimum rate

```bash
impale7@htb[/htb]$ sudo nmap 10.129.2.0/24 -F -oN tnet.minrate300 --min-rate 300

<SNIP>
Nmap done: 256 IP addresses (10 hosts up) scanned in 8.67 seconds
```

What this does:

* `10.129.2.0/24` scans the target subnet
* `-F` scans the top `100` ports
* `-oN tnet.minrate300` saves normal output to a file
* `--min-rate 300` tells Nmap to send at least `300` packets per second

Compare the results:

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
* `--min-rate` can give a strong speed boost without losing coverage
* this works best when the network is stable and not rate-limiting you

### Use timing templates

When you do not want to tune each setting manually, use a timing template.

Nmap supports six templates:

* `-T 0` or `-T paranoid`
* `-T 1` or `-T sneaky`
* `-T 2` or `-T polite`
* `-T 3` or `-T normal`
* `-T 4` or `-T aggressive`
* `-T 5` or `-T insane`

`-T 3` is the default.

Higher values push speed harder.

They also increase noise and the chance of dropped results.

#### Default timing

```bash
impale7@htb[/htb]$ sudo nmap 10.129.2.0/24 -F -oN tnet.default

<SNIP>
Nmap done: 256 IP addresses (10 hosts up) scanned in 32.44 seconds
```

#### `-T 5` timing

```bash
impale7@htb[/htb]$ sudo nmap 10.129.2.0/24 -F -oN tnet.T5 -T 5

<SNIP>
Nmap done: 256 IP addresses (10 hosts up) scanned in 18.07 seconds
```

What this does:

* `10.129.2.0/24` scans the target subnet
* `-F` scans the top `100` ports
* `-oN tnet.T5` saves normal output to a file
* `-T 5` uses the most aggressive timing template

Compare the results:

```bash
impale7@htb[/htb]$ cat tnet.default | grep "/tcp" | wc -l

23
```

```bash
impale7@htb[/htb]$ cat tnet.T5 | grep "/tcp" | wc -l

23
```

What to notice:

* the aggressive template finishes faster
* this example keeps the same open-port count
* aggressive templates are louder and easier to detect

{% hint style="info" %}
Start with `-T 3` or `-T 4` unless you have a strong reason to go faster. Use `-T 5` carefully.
{% endhint %}

### Fast decision rules

* If the network is slow, tune RTT carefully before pushing rate.
* If packets drop often, avoid setting retries too low.
* If you are whitelisted, `--min-rate` can save time fast.
* If you need a simple speed boost, try `-T 4` before manual tuning.

### Further reading

* Official [Nmap performance guide](https://nmap.org/book/man-performance.html)
* Official [Nmap timing template reference](https://nmap.org/book/performance-timing-templates.html)
* [Host and Port Scanning](host-and-port-scanning.md)
