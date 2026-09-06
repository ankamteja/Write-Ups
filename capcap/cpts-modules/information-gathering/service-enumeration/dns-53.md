# DNS

Use this page to enumerate DNS, review risky settings, and map target infrastructure.

### Key notes

* DNS is a distributed, hierarchical system that translates human-readable domain names, such as `academy.hackthebox.com`, into machine-routable IP addresses.
* The key port is UDP or TCP `53`.
* `dig` is the primary tool for manual DNS inspection.
* `dnsenum` helps automate nameserver checks, AXFR attempts, and subdomain brute forcing.

### Core concepts and architecture

* **What it is** — A distributed, hierarchical system that translates human-readable domain names (e.g., `academy.hackthebox.com`) into machine-routable IP addresses.
* **Key Port** — UDP / TCP `53`
* **The Infrastructure**
  * **Resolver** — Your local system or router making the initial name request.
  * **Root Servers** — 13 global clusters handling Top-Level Domains (TLDs like `.com`, `.htb`).
  * **Authoritative Server** — Holds the definitive blueprints (Zone Files) for a specific domain.

### Essential record types

When querying a server, you look for specific records to map the target infrastructure:

| **Record Type** | **Purpose**                                  | **High-Value Targeting Info**                                                                                                                                             |
| --------------- | -------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| A               | Maps hostname to IPv4 address.               | Finds public-facing or internal host targets.                                                                                                                             |
| AAAA            | Maps hostname to IPv6 address.               | Modern address targeting.                                                                                                                                                 |
| CNAME           | Acts as an Alias pointing to another domain. | Reveals underlying canonical naming structures.                                                                                                                           |
| MX              | Identifies responsible Mail Servers.         | Identifies corporate mail infrastructure targets.                                                                                                                         |
| NS              | Lists the domain's Nameservers.              | Tells you exactly which DNS servers to target with queries.                                                                                                               |
| TXT             | Arbitrary text strings.                      | Often leaks third-party integration data (Google, Atlassian, SPF/DMARC flags).                                                                                            |
| PTR             | Handles Reverse Lookups (IP ──> Name).       | Used to discover valid server names from a raw IP range.                                                                                                                  |
| SOA             | Start of Authority metadata.                 | Reveals administrative email. Note: The first dot (`.`) in the SOA email string translates to an `@` symbol (e.g., `hostmaster.domain.com.` ──> `hostmaster@domain.com`). |

### Dangerous configurations

Watch for these options when reviewing a BIND9 DNS server configuration file. Loose permissions expose the backend environment:

| **Setting**                    | **Description**                                                                                                                                           |
| ------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `allow-query { any; };`        | Allows any host on the network to request standard resolutions.                                                                                           |
| `allow-recursion { any; };`    | Allows external hosts to use your server for outside resolutions. This leaves the server vulnerable to becoming a tool in DNS Amplification DDoS attacks. |
| `allow-transfer { any; };`     | Allows unauthorized clients to download a complete copy of the internal zone files via an AXFR request.                                                   |
| `allow-transfer { subnets; };` | Allows clients in permitted subnets to download zone files via AXFR.                                                                                      |

### Footprinting the service

The `dig` tool is your primary asset for inspecting DNS behavior.

Target a specific DNS server using the `@<SERVER_IP>` syntax:

{% code title="Manual DNS queries with dig" %}
```bash
# 1. Query for the authoritative Nameservers of a domain
dig ns inlanefreight.htb @10.129.14.128

# 2. Extract software version data using a CHAOS class text query
dig CH TXT version.bind 10.129.120.85

# 3. Request a comprehensive dump of all public records the server discloses
dig any inlanefreight.htb @10.129.14.128

# 4. Attempt a Zone Transfer (AXFR) to steal the entire database
dig axfr inlanefreight.htb @10.129.14.128

# 5. Target a suspected internal domain path for a Zone Transfer
dig axfr internal.inlanefreight.htb @10.129.14.128
```
{% endcode %}

### Subdomain brute-forcing via Bash loop

When Zone Transfers (`AXFR`) are blocked, use a wordlist to guess subdomains sequentially:

{% code title="Subdomain brute-force loop" %}
```bash
for sub in $(cat subdomains-wordlist.txt); do dig $sub.inlanefreight.htb @10.129.14.128 | grep -v ';\|SOA' | sed -r '/^\s*$/d' | grep $sub; done
```
{% endcode %}

### Swiss-army automation tools

When manual verification is completed or you need high-speed enumeration against broad wordlists, pivot to specialized scripts:

#### `dnsenum`

Automates nameserver checks, AXFR attempts, and performs multithreaded subdomain brute forcing:

{% code title="dnsenum example" %}
```bash
dnsenum --dnsserver 10.129.14.128 --enum inlanefreight.htb -f /usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt
```
{% endcode %}

### Post-enumeration priorities

* If AXFR is successful: Read through the entire zone file layout. Look specifically for out-of-sight infrastructure elements, such as backup sites, pre-production/development spaces (`dev.target.htb`, `test.target.htb`), staging environments, Active Directory domain controllers (`dc1.internal...`), and VPN endpoints.
* If AXFR is blocked: Check for version information vulnerabilities, and immediately switch to dictionary brute-forcing via `dnsenum` to mapping out active network systems manually.
