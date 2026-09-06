---
description: >-
  Collect subdomains, DNS records, and external exposure without touching the
  target directly.
---

# Passive Recon

Use passive recon to collect useful data without sending traffic to the target directly.

Start with these sources:

* `crt.sh` — find subdomains from certificate transparency logs
* `host` — resolve subdomains to IP addresses
* `Shodan` — review exposed services already indexed on the public internet
* `dig` — inspect DNS records such as `A`, `MX`, `NS`, and `TXT`

### Basic passive recon flow

1. Find subdomains with `crt.sh`.
2. Resolve them with `host`.
3. Check the resolved IPs in Shodan.
4. Review DNS records with `dig`.

### Common DNS record types

The most useful records here are:

* `A` — maps a hostname to an IPv4 address
* `MX` — identifies the mail servers for the domain
* `NS` — shows the authoritative name servers
* `TXT` — often reveals third-party services, email security settings, or ownership data

### `crt.sh`

`crt.sh` helps you find subdomains that appear in public TLS certificates.

This is useful because many organizations issue certificates for internal tools, staging apps, VPN portals, and other internet-facing hosts.

#### Query certificate transparency logs

```bash
curl -s "https://crt.sh/?q=inlanefreight.com&output=json" | jq .
```

What this does:

* `curl -s` fetches the response silently
* `?q=inlanefreight.com` searches for certificates tied to the domain
* `&output=json` returns structured JSON output
* `jq .` formats the JSON so it is easier to read

#### Extract unique subdomains

```bash
curl -s "https://crt.sh/?q=inlanefreight.com&output=json" | jq . | grep name | cut -d":" -f2 | grep -v "CN=" | cut -d'"' -f2 | awk '{gsub(/\\n/,"\n");}1;' | sort -u
```

This filters the output down to unique subdomains.

### `host`

Use `host` to resolve discovered subdomains to IP addresses.

That tells you which assets are live and gives you targets to inspect further.

#### Resolve subdomains to IP addresses

```bash
for i in $(cat subdomainlist); do host $i | grep "has address" | grep inlanefreight.com | cut -d" " -f1,4; done
```

This prints each subdomain with its resolved IP address.

### Shodan

Shodan shows internet-exposed services it has already scanned and indexed.

This helps you identify high-value hosts before you start active testing.

Focus on:

* open ports
* service banners
* product names and versions

#### Build an IP list

```bash
for i in $(cat subdomainlist); do host $i | grep "has address" | grep inlanefreight.com | cut -d" " -f4 >> ip-addresses.txt; done
```

This writes resolved IP addresses to `ip-addresses.txt`.

#### Query each IP in Shodan

```bash
for i in $(cat ip-addresses.txt); do shodan host $i; done
```

This checks which services Shodan has already indexed for each host.

### `dig`

Use `dig` to inspect individual DNS records and confirm what the domain exposes publicly.

`ANY` queries are not always fully supported, so query specific record types when needed.

#### Query DNS records

```bash
dig any inlanefreight.com
```

Use targeted lookups when you need more reliable results:

```bash
dig A inlanefreight.com
dig MX inlanefreight.com
dig NS inlanefreight.com
dig TXT inlanefreight.com
```
