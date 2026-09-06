---
description: Run Nmap scripts to enumerate services, pull banners, and check known issues.
---

# NSE

Use this page to run Nmap scripts against exposed services and pull deeper findings fast.

NSE stands for **Nmap Scripting Engine**.

It lets Nmap run Lua scripts for enumeration, detection, and targeted checks.

Use it when a basic port scan is not enough.

It helps you:

* grab banners and metadata
* enumerate service features
* run targeted checks by category
* spot likely vulnerabilities faster

{% hint style="warning" %}
Scripts in categories like `intrusive`, `exploit`, `brute`, `dos`, and some `vuln` checks can be noisy or risky. Use them only when the target and scope allow it.
{% endhint %}

### NSE script categories

NSE scripts are grouped into categories.

Each category serves a different purpose.

| Category    | Use                                                    |
| ----------- | ------------------------------------------------------ |
| `auth`      | Checks authentication and credential-related behavior. |
| `broadcast` | Discovers hosts by broadcast and can feed later scans. |
| `brute`     | Tries credential brute-force attacks against services. |
| `default`   | Runs the default safe script set behind `-sC`.         |
| `discovery` | Gathers service and host information.                  |
| `dos`       | Tests denial-of-service conditions.                    |
| `exploit`   | Tries known exploit paths.                             |
| `external`  | Uses external services for added processing.           |
| `fuzzer`    | Sends unusual input to trigger unexpected behavior.    |
| `intrusive` | Runs checks that may affect the target.                |
| `malware`   | Looks for malware indicators.                          |
| `safe`      | Runs lower-risk defensive checks.                      |
| `version`   | Extends service version detection.                     |
| `vuln`      | Checks for known vulnerabilities.                      |

### Run NSE scripts

You can run NSE in three common ways.

#### Run default scripts

Use `-sC` to run the default script set.

```bash
impale7@htb[/htb]$ sudo nmap <target> -sC
```

#### Run a script category

Use a category when you want a broader type of check.

```bash
impale7@htb[/htb]$ sudo nmap <target> --script <category>
```

#### Run specific scripts

Use exact script names when you already know what you want.

```bash
impale7@htb[/htb]$ sudo nmap <target> --script <script-name>,<script-name>
```

### Example — banner and SMTP commands

Target the SMTP service directly and run two useful scripts.

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

What this does:

* `10.129.2.28` targets the host
* `-p 25` scans only SMTP on TCP `25`
* `--script banner,smtp-commands` runs two specific NSE scripts

What you learn:

* `banner` reveals `Postfix` and the `Ubuntu` clue
* `smtp-commands` shows supported SMTP commands
* supported commands can help you test user enumeration paths

### Use aggressive mode carefully

Nmap also supports aggressive mode with `-A`.

This combines:

* service detection with `-sV`
* OS detection with `-O`
* traceroute with `--traceroute`
* default scripts with `-sC`

#### Example

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

What this does:

* `10.129.2.28` targets the host
* `-p 80` scans only HTTP on TCP `80`
* `-A` runs an aggressive multi-feature scan

What you learn:

* the web server is `Apache 2.4.29`
* the site uses `WordPress 5.3.4`
* the page title is `blog.inlanefreight.com`
* the host is likely running `Linux`

{% hint style="info" %}
`-A` is useful for quick triage. It is also louder than a focused scan. Use it when you want speed over stealth.
{% endhint %}

### Use the `vuln` category for targeted checks

Once you identify a web service, the `vuln` category can surface extra findings fast.

This works best after you already know the service and version.

#### Example

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

What this does:

* `10.129.2.28` targets the host
* `-p 80` scans only HTTP on TCP `80`
* `-sV` confirms the service version first
* `--script vuln` runs scripts from the `vuln` category

What you learn:

* likely WordPress paths and version clues
* a discovered username such as `admin`
* possible CVEs tied to the detected server software

### Takeaways

Use NSE when you need more than open ports.

The fastest pattern is:

1. identify the port
2. confirm the service with `-sV`
3. run focused scripts or categories
4. validate the result manually when it matters

Good starting points:

* `-sC` for default script coverage
* `--script banner` for quick banner checks
* `--script vuln` for focused vulnerability triage

### Further reading

* Official [NSE script reference](https://nmap.org/nsedoc/index.html)
* [Service Enumeration](service-enumeration-using-nmap.md)
