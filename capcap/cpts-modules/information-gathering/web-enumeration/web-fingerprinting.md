# Web Fingerprinting

## Web fingerprinting

Use this page to identify the technologies behind a target website.

Focus on headers, redirects, WAF detection, and software clues.

### Why fingerprinting matters

Fingerprinting helps you:

* target likely vulnerabilities faster
* spot outdated or misconfigured software
* prioritize hosts that look more exposed
* build a better attack surface profile

### Common techniques

#### Banner grabbing

Read server banners and version strings.

Common sources include the `Server` header and redirect behavior.

#### HTTP header analysis

Inspect headers such as:

* `Server`
* `X-Powered-By`
* `Link`
* `Location`

These often reveal frameworks, CMSs, or proxy layers.

#### Response probing

Send specific requests and compare the response.

Errors, redirects, and unusual behavior can expose the stack.

#### Page content analysis

Look at source, scripts, paths, and metadata.

Patterns such as `wp-` or framework-specific assets are often enough.

### Common tools

| Tool         | Best use                                       |
| ------------ | ---------------------------------------------- |
| `Wappalyzer` | Browser-based technology profiling             |
| `BuiltWith`  | Detailed hosted stack reports                  |
| `WhatWeb`    | CLI fingerprinting for web technologies        |
| `Nmap`       | Service and OS fingerprinting with NSE support |
| `Netcraft`   | Hosting and security posture checks            |
| `wafw00f`    | WAF detection                                  |

### Example target: `inlanefreight.com`

This workflow uses both manual and automated checks.

Start with headers. Then check for a WAF. Then validate with `Nikto`.

### Step 1: Grab banners with `curl`

Use `curl -I` to request headers only.

#### Check the base domain

```bash
curl -I inlanefreight.com
```

Example output:

```shell
impale7@htb[/htb]$ curl -I inlanefreight.com

HTTP/1.1 301 Moved Permanently
Date: Fri, 31 May 2024 12:07:44 GMT
Server: Apache/2.4.41 (Ubuntu)
Location: https://inlanefreight.com/
Content-Type: text/html; charset=iso-8859-1
```

#### What to note

* `Server: Apache/2.4.41 (Ubuntu)` reveals the web server and OS family
* `Location` shows an HTTP to HTTPS redirect

Follow the redirect chain and keep checking headers.

#### Check the HTTPS endpoint

```bash
curl -I https://inlanefreight.com
```

Example output:

```shell
impale7@htb[/htb]$ curl -I https://inlanefreight.com

HTTP/1.1 301 Moved Permanently
Date: Fri, 31 May 2024 12:12:12 GMT
Server: Apache/2.4.41 (Ubuntu)
X-Redirect-By: WordPress
Location: https://www.inlanefreight.com/
Content-Type: text/html; charset=UTF-8
```

#### What to note

* `X-Redirect-By: WordPress` is a strong CMS clue
* the site redirects again to `https://www.inlanefreight.com/`

#### Check the final host

```bash
curl -I https://www.inlanefreight.com
```

Example output:

```shell
impale7@htb[/htb]$ curl -I https://www.inlanefreight.com

HTTP/1.1 200 OK
Date: Fri, 31 May 2024 12:12:26 GMT
Server: Apache/2.4.41 (Ubuntu)
Link: <https://www.inlanefreight.com/index.php/wp-json/>; rel="https://api.w.org/"
Link: <https://www.inlanefreight.com/index.php/wp-json/wp/v2/pages/7>; rel="alternate"; type="application/json"
Link: <https://www.inlanefreight.com/>; rel=shortlink
Content-Type: text/html; charset=UTF-8
```

#### Key takeaway

The `wp-json` path and `wp-` prefix strongly suggest `WordPress`.

### Step 2: Detect a WAF with `wafw00f`

Check for a web application firewall before deeper probing.

A WAF can change responses or block noisy requests.

#### Install `wafw00f`

```bash
pip3 install git+https://github.com/EnableSecurity/wafw00f
```

#### Run the check

```bash
wafw00f inlanefreight.com
```

Example output:

```shell
impale7@htb[/htb]$ wafw00f inlanefreight.com

                ______
               /      \
              (  W00f! )
               \  ____/
               ,,    __            404 Hack Not Found
           |`-.__   / /                      __     __
           /"  _/  /_/                       \ \   / /
          *===*    /                          \ \_/ /  405 Not Allowed
         /     )__//                           \   /
    /|  /     /---`                        403 Forbidden
    \\/`   \ |                                 / _ \
    `\    /_\\_              502 Bad Gateway  / / \ \  500 Internal Error
      `_____``-`                             /_/   \_\

                        ~ WAFW00F : v2.2.0 ~
        The Web Application Firewall Fingerprinting Toolkit
    
[*] Checking https://inlanefreight.com
[+] The site https://inlanefreight.com is behind Wordfence (Defiant) WAF.
[~] Number of requests: 2
```

#### What to note

* the target is behind `Wordfence`
* later probes may be filtered or rate-limited

{% hint style="info" %}
If a target uses a WAF, expect false negatives and blocked requests.
{% endhint %}

### Step 3: Validate software clues with `Nikto`

`Nikto` can confirm software, headers, and common disclosure issues.

#### Install `Nikto` if needed

```bash
sudo apt update && sudo apt install -y perl
git clone https://github.com/sullo/nikto
cd nikto/program
chmod +x ./nikto.pl
```

#### Run only the software identification modules

```bash
nikto -h inlanefreight.com -Tuning b
```

#### Flag reminder

* `-h` sets the target host
* `-Tuning b` runs the software identification checks

Example output:

```shell
impale7@htb[/htb]$ nikto -h inlanefreight.com -Tuning b

- Nikto v2.5.0
---------------------------------------------------------------------------
+ Multiple IPs found: 134.209.24.248, 2a03:b0c0:1:e0::32c:b001
+ Target IP:          134.209.24.248
+ Target Hostname:    www.inlanefreight.com
+ Target Port:        443
---------------------------------------------------------------------------
+ SSL Info:        Subject:  /CN=inlanefreight.com
                   Altnames: inlanefreight.com, www.inlanefreight.com
                   Ciphers:  TLS_AES_256_GCM_SHA384
                   Issuer:   /C=US/O=Let's Encrypt/CN=R3
+ Start Time:         2024-05-31 13:35:54 (GMT0)
---------------------------------------------------------------------------
+ Server: Apache/2.4.41 (Ubuntu)
+ /: Link header found with value: ARRAY(0x558e78790248). See: https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Link
+ /: The site uses TLS and the Strict-Transport-Security HTTP header is not defined. See: https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Strict-Transport-Security
+ /: The X-Content-Type-Options header is not set. This could allow the user agent to render the content of the site in a different fashion to the MIME type. See: https://www.netsparker.com/web-vulnerability-scanner/vulnerabilities/missing-content-type-header/
+ /index.php?: Uncommon header 'x-redirect-by' found, with contents: WordPress.
+ No CGI Directories found (use '-C all' to force check all possible dirs)
+ /: The Content-Encoding header is set to "deflate" which may mean that the server is vulnerable to the BREACH attack. See: http://breachattack.com/
+ Apache/2.4.41 appears to be outdated (current is at least 2.4.59). Apache 2.2.34 is the EOL for the 2.x branch.
+ /: Web Server returns a valid response with junk HTTP methods which may cause false positives.
+ /license.txt: License file found may identify site software.
+ /: A Wordpress installation was found.
+ /wp-login.php?action=register: Cookie wordpress_test_cookie created without the httponly flag. See: https://developer.mozilla.org/en-US/docs/Web/HTTP/Cookies
+ /wp-login.php:X-Frame-Options header is deprecated and has been replaced with the Content-Security-Policy HTTP header with the frame-ancestors directive instead. See: https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/X-Frame-Options
+ /wp-login.php: Wordpress login found.
+ 1316 requests: 0 error(s) and 12 item(s) reported on remote host
+ End Time:           2024-05-31 13:47:27 (GMT0) (693 seconds)
---------------------------------------------------------------------------
+ 1 host(s) tested
```

### Key findings from the `Nikto` scan

* the site resolves to both IPv4 and IPv6
* the server is `Apache/2.4.41 (Ubuntu)`
* a `WordPress` installation is present
* `/wp-login.php` is exposed
* `/license.txt` may disclose software details
* `Strict-Transport-Security` is missing
* `X-Content-Type-Options` is missing
* `x-redirect-by: WordPress` confirms CMS behavior
* the server version appears outdated

### Fast workflow

1. Use `curl -I` to inspect headers and redirects.
2. Follow redirects until you reach the final host.
3. Look for CMS or framework clues such as `wp-json`.
4. Run `wafw00f` before noisier probes.
5. Use `Nikto -Tuning b` to validate software findings.

### Quick summary

This target shows a clear stack:

* `Apache/2.4.41 (Ubuntu)`
* `WordPress`
* `Wordfence WAF`

That gives you a focused next step for further web enumeration.
