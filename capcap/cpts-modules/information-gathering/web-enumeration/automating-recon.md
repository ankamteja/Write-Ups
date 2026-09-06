---
description: Use recon frameworks to speed up web enumeration and scale repeatable checks.
---

# Automating Recon

## Automating recon

Use automation to speed up repeatable recon tasks.

This works best after you understand the manual checks first.

### Why automate reconnaissance

Automation helps when you need speed, scale, and repeatable output.

Key benefits:

* `Efficiency` — tools handle repetitive tasks faster than manual work
* `Scalability` — you can scan many targets or domains consistently
* `Consistency` — the same checks run the same way each time
* `Comprehensive coverage` — one workflow can include DNS, subdomains, crawling, ports, and more
* `Integration` — many tools plug into later assessment or exploitation steps

{% hint style="info" %}
Automation improves collection speed. It does not replace analysis.
{% endhint %}

### Reconnaissance frameworks

These tools aim to cover multiple recon tasks from one place.

#### Common options

* [FinalRecon](https://github.com/thewhiteh4t/FinalRecon) — Python-based web recon with modules for SSL, Whois, headers, and crawling
* [Recon-ng](https://github.com/lanmaster53/recon-ng) — modular Python framework for DNS, subdomains, ports, crawling, and more
* [theHarvester](https://github.com/laramies/theHarvester) — collects emails, subdomains, hosts, employee names, open ports, and banners from public sources
* [SpiderFoot](https://github.com/smicallef/spiderfoot) — OSINT automation across domains, IPs, emails, social profiles, DNS, web, and ports
* [OSINT Framework](https://osintframework.com/) — curated collection of OSINT tools and information sources

### FinalRecon

`FinalRecon` provides a broad set of recon modules in one tool.

#### What FinalRecon can collect

* `Header information` — server details, technologies, and possible misconfigurations
* `Whois lookup` — domain registration details and contact data
* `SSL certificate information` — certificate validity, issuer, and related details
* `Crawler`
  * HTML, CSS, and JavaScript resources
  * internal and external links
  * images, `robots.txt`, and `sitemap.xml`
  * links found in JavaScript
  * Wayback Machine data
* `DNS enumeration` — more than 40 record types, including DMARC
* `Subdomain enumeration` — sources such as `crt.sh`, `AnubisDB`, `ThreatMiner`, `CertSpotter`, `Facebook API`, `VirusTotal API`, `Shodan API`, and `BeVigil API`
* `Directory enumeration` — custom wordlists and file extensions
* `Wayback Machine` — URLs from the last five years for historical review

### Install FinalRecon

Run:

```bash
git clone https://github.com/thewhiteh4t/FinalRecon.git
cd FinalRecon
pip3 install -r requirements.txt
chmod +x ./finalrecon.py
./finalrecon.py --help
```

#### What each step does

1. `git clone` downloads the repository.
2. `cd FinalRecon` moves into the project folder.
3. `pip3 install -r requirements.txt` installs dependencies.
4. `chmod +x ./finalrecon.py` makes the script executable.
5. `./finalrecon.py --help` confirms the install and shows available options.

<details>

<summary>View the full help output</summary>

```
usage: finalrecon.py [-h] [--url URL] [--headers] [--sslinfo] [--whois]
                     [--crawl] [--dns] [--sub] [--dir] [--wayback] [--ps]
                     [--full] [-nb] [-dt DT] [-pt PT] [-T T] [-w W] [-r] [-s]
                     [-sp SP] [-d D] [-e E] [-o O] [-cd CD] [-k K]

FinalRecon - All in One Web Recon | v1.1.6

optional arguments:
  -h, --help  show this help message and exit
  --url URL   Target URL
  --headers   Header Information
  --sslinfo   SSL Certificate Information
  --whois     Whois Lookup
  --crawl     Crawl Target
  --dns       DNS Enumeration
  --sub       Sub-Domain Enumeration
  --dir       Directory Search
  --wayback   Wayback URLs
  --ps        Fast Port Scan
  --full      Full Recon

Extra Options:
  -nb         Hide Banner
  -dt DT      Number of threads for directory enum [ Default : 30 ]
  -pt PT      Number of threads for port scan [ Default : 50 ]
  -T T        Request Timeout [ Default : 30.0 ]
  -w W        Path to Wordlist [ Default : wordlists/dirb_common.txt ]
  -r          Allow Redirect [ Default : False ]
  -s          Toggle SSL Verification [ Default : True ]
  -sp SP      Specify SSL Port [ Default : 443 ]
  -d D        Custom DNS Servers [ Default : 1.1.1.1 ]
  -e E        File Extensions [ Example : txt, xml, php ]
  -o O        Export Format [ Default : txt ]
  -cd CD      Change export directory [ Default : ~/.local/share/finalrecon ]
  -k K        Add API key [ Example : shodan@key ]
```

</details>

### Core FinalRecon flags

Use these flags most often:

| Option         | Argument | Description                      |
| -------------- | -------- | -------------------------------- |
| `-h`, `--help` |          | Show the help message and exit   |
| `--url`        | `URL`    | Set the target URL               |
| `--headers`    |          | Retrieve header information      |
| `--sslinfo`    |          | Retrieve SSL certificate details |
| `--whois`      |          | Perform a Whois lookup           |
| `--crawl`      |          | Crawl the target site            |
| `--dns`        |          | Run DNS enumeration              |
| `--sub`        |          | Enumerate subdomains             |
| `--dir`        |          | Search for directories           |
| `--wayback`    |          | Retrieve Wayback URLs            |
| `--ps`         |          | Run a fast port scan             |
| `--full`       |          | Run the full recon workflow      |

#### Useful extra flags

| Option | Description                                        |
| ------ | -------------------------------------------------- |
| `-dt`  | Number of threads for directory enumeration        |
| `-pt`  | Number of threads for port scanning                |
| `-T`   | Request timeout                                    |
| `-w`   | Custom wordlist path                               |
| `-r`   | Allow redirects                                    |
| `-s`   | Toggle SSL verification                            |
| `-sp`  | Set a custom SSL port                              |
| `-d`   | Set custom DNS servers                             |
| `-e`   | Add file extensions such as `txt`, `xml`, or `php` |
| `-o`   | Set export format                                  |
| `-cd`  | Change the export directory                        |
| `-k`   | Add an API key such as `shodan@key`                |

### Example scan

This example collects headers and Whois data for `inlanefreight.com`.

```bash
./finalrecon.py --headers --whois --url http://inlanefreight.com
```

Example output:

```
 ______  __   __   __   ______   __
/\  ___\/\ \ /\ "-.\ \ /\  __ \ /\ \
\ \  __\\ \ \\ \ \-.  \\ \  __ \\ \ \____
 \ \_\   \ \_\\ \_\\"\_\\ \_\ \_\\ \_____\
  \/_/    \/_/ \/_/ \/_/ \/_/\/_/ \/_____/
 ______   ______   ______   ______   __   __
/\  == \ /\  ___\ /\  ___\ /\  __ \ /\ "-.\ \
\ \  __< \ \  __\ \ \ \____\ \ \/\ \\ \ \-.  \
 \ \_\ \_\\ \_____\\ \_____\\ \_____\\ \_\\"\_\
  \/_/ /_/ \/_____/ \/_____/ \/_____/ \/_/ \/_/

[>] Created By   : thewhiteh4t
 |---> Twitter   : https://twitter.com/thewhiteh4t
 |---> Community : https://twc1rcle.com/
[>] Version      : 1.1.6

[+] Target : http://inlanefreight.com

[+] IP Address : 134.209.24.248

[!] Headers :

Date : Tue, 11 Jun 2024 10:08:00 GMT
Server : Apache/2.4.41 (Ubuntu)
Link : <https://www.inlanefreight.com/index.php/wp-json/>; rel="https://api.w.org/", <https://www.inlanefreight.com/index.php/wp-json/wp/v2/pages/7>; rel="alternate"; type="application/json", <https://www.inlanefreight.com/>; rel=shortlink
Vary : Accept-Encoding
Content-Encoding : gzip
Content-Length : 5483
Keep-Alive : timeout=5, max=100
Connection : Keep-Alive
Content-Type : text/html; charset=UTF-8

[!] Whois Lookup :

   Domain Name: INLANEFREIGHT.COM
   Registry Domain ID: 2420436757_DOMAIN_COM-VRSN
   Registrar WHOIS Server: whois.registrar.amazon.com
   Registrar URL: http://registrar.amazon.com
   Updated Date: 2023-07-03T01:11:15Z
   Creation Date: 2019-08-05T22:43:09Z
   Registry Expiry Date: 2024-08-05T22:43:09Z
   Registrar: Amazon Registrar, Inc.
   Registrar IANA ID: 468
   Registrar Abuse Contact Email: abuse@amazonaws.com
   Registrar Abuse Contact Phone: +1.2024422253
   Domain Status: clientDeleteProhibited https://icann.org/epp#clientDeleteProhibited
   Domain Status: clientTransferProhibited https://icann.org/epp#clientTransferProhibited
   Domain Status: clientUpdateProhibited https://icann.org/epp#clientUpdateProhibited
   Name Server: NS-1303.AWSDNS-34.ORG
   Name Server: NS-1580.AWSDNS-05.CO.UK
   Name Server: NS-161.AWSDNS-20.COM
   Name Server: NS-671.AWSDNS-19.NET
   DNSSEC: unsigned
   URL of the ICANN Whois Inaccuracy Complaint Form: https://www.icann.org/wicf/

[+] Completed in 0:00:00.257780

[+] Exported : /home/htb-ac-643601/.local/share/finalrecon/dumps/fr_inlanefreight.com_11-06-2024_11:07:59
```

### What to look for in the output

From headers:

* server and platform clues
* redirect behavior
* CMS or framework indicators
* unusual headers such as API references

From Whois:

* registrar details
* creation and expiry dates
* abuse contacts
* name servers
* domain status values

### Quick summary

Automation makes recon faster and more repeatable.

Use frameworks when you need broad coverage across common tasks.

`FinalRecon` is a strong all-in-one option for headers, Whois, crawling, DNS, subdomains, directories, Wayback data, and fast port checks.
