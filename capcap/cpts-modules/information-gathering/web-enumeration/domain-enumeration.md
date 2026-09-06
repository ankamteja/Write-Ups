# Domain Enumeration

Use it when you need fast DNS recon, subdomain clues, and infrastructure context.

### Why DNS matters

DNS helps you map the target before active testing.

Good DNS enumeration can reveal:

* subdomains
* mail servers
* name servers
* third-party services
* internal naming patterns
* cloud or edge infrastructure

Examples:

* `dev.example.com` may expose a staging app
* `vpn.example.com` may reveal a remote access portal
* `TXT` records may expose SPF, DKIM, verification tokens, or vendor usage

### DNS resolution flow

Keep this order in mind:

1. Your system checks local cache.
2. The resolver checks its cache.
3. The resolver asks a root server.
4. The root server points to the TLD server.
5. The TLD server points to the authoritative server.
6. The authoritative server returns the answer.
7. The resolver caches it and returns it to you.

### Key terms

* **Resolver** — the server that performs lookups for you
* **Authoritative server** — the server that holds the target zone data
* **Zone** — the DNS namespace controlled by one admin
* **Zone file** — the records stored for that zone
* **TTL** — how long a result can be cached

### Record types to know

Focus on these for CPTS:

* `A` — maps a host to an IPv4 address
* `AAAA` — maps a host to an IPv6 address
* `CNAME` — aliases one host to another
* `MX` — lists mail servers
* `NS` — lists authoritative name servers
* `TXT` — stores text such as SPF, DKIM, and verification values
* `SOA` — shows zone authority details and serial values
* `PTR` — reverse lookup from IP to hostname
* `SRV` — shows service locations and ports

### What to look for during recon

Write down:

* subdomains with business value
* out-of-scope third-party providers
* old or unusual hostnames
* email security records
* CDN, WAF, or cloud platform hints
* naming patterns you can reuse for brute force

High-value clues:

* `admin`, `vpn`, `dev`, `stage`, `test`, `jira`, `git`, `mail`
* hosted mail providers
* external SaaS integrations
* TXT records that mention security tooling or vendors

### Fast workflow

{% stepper %}
{% step %}
### Identify core records

Start with `A`, `AAAA`, `MX`, `NS`, `TXT`, and `SOA`.

```bash
dig example.com A
dig example.com AAAA
dig example.com MX
dig example.com NS
dig example.com TXT
dig example.com SOA
```
{% endstep %}

{% step %}
### Check alternate resolvers

Compare answers and inspect the lookup path.

```bash
dig @1.1.1.1 example.com
dig @8.8.8.8 example.com
dig +trace example.com
```
{% endstep %}

{% step %}
### Resolve useful hosts

Confirm what each hostname points to.

```bash
host www.example.com
host mail.example.com
dig dev.example.com CNAME
```
{% endstep %}

{% step %}
### Enumerate subdomains

Use passive and wordlist-based methods where scope allows.

```bash
dnsenum --dnsserver 8.8.8.8 --enum example.com
dnsrecon -d example.com -t std
theHarvester -d example.com -b all
```
{% endstep %}

{% step %}
### Test for zone transfer

Try this only when scope allows.

```bash
dig AXFR example.com @ns1.example.com
host -l example.com ns1.example.com
```
{% endstep %}
{% endstepper %}

{% hint style="warning" %}
Many DNS servers block `AXFR` and ignore `ANY` queries. That is normal.
{% endhint %}

### Core commands

#### `dig`

Use `dig` when you need full DNS detail.

```bash
dig example.com
dig example.com A
dig example.com MX
dig example.com NS
dig example.com TXT
dig example.com CNAME
dig example.com SOA
dig +short example.com
dig +noall +answer example.com
dig -x 192.168.1.10
dig @1.1.1.1 example.com
dig +trace example.com
```

#### `host`

Use `host` for quick checks.

```bash
host example.com
host -t MX example.com
host -t TXT example.com
host 10.10.10.10
```

#### `nslookup`

Use `nslookup` for simple lookups.

```bash
nslookup example.com
nslookup -type=MX example.com
nslookup -type=TXT example.com
```

### How to read `dig` output

Focus on four parts:

1. **Header** — status and flags
2. **Question** — what record you asked for
3. **Answer** — the returned record values
4. **Footer** — resolver used, query time, and message size

Key fields:

* `status: NOERROR` means the lookup worked
* `ANSWER: 1` means one answer record returned
* `IN` means the Internet class
* `TTL` shows cache lifetime

Use this when you only want the result:

```bash
dig +short example.com
```

Use this when you want only the answer section:

```bash
dig +noall +answer example.com
```

### The hosts file

The hosts file bypasses DNS for local resolution.

Common paths:

* Linux and macOS — `/etc/hosts`
* Windows — `C:\Windows\System32\drivers\etc\hosts`

Format:

```txt
<IP_ADDRESS> <HOSTNAME> [ALIAS]
```

Examples:

```txt
127.0.0.1 localhost
192.168.1.10 devserver.local
10.10.10.15 admin.example.local
```

Use it for:

* local virtual host testing
* forced hostname resolution
* application testing after IP discovery

### Exam-focused notes

Remember these patterns:

* `MX` records can reveal mail hosts worth testing later
* `NS` records can reveal hosting providers or delegated zones
* `TXT` records often expose security tooling and tenant details
* `CNAME` records can show third-party services
* reverse lookups can reveal better hostnames than forward lookups

If you find multiple subdomains, group them by purpose:

* web
* mail
* VPN
* development
* admin

This helps you pick the best attack surface first.



Track these in your scratchpad:

* domain
* resolved IPs
* name servers
* mail servers
* TXT values
* subdomains
* possible vhosts
* likely high-value hosts

### Related pages

* [Passive Web Enumeration](passive-web-enumeration.md)
* [Passive Recon](../network-enumeration-with-nmap/passive-recon/)
* [WhatWeb](../../../cheat-sheets/whatweb.md)
