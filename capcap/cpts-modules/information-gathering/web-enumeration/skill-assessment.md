---
description: Structured walkthrough for the web enumeration skill assessment.
---

# Skill Assessment

## Skill assessment

Use this walkthrough to move from the first host entry to the final answers.

### Before you start

Add the target host:

```bash
sudo sh -c "echo '154.57.164.67 inlanefreight.htb' >> /etc/hosts"
```

Use `sudo sh -c` so the redirection runs as root.

Then browse to:

`http://inlanefreight.htb:32311`

### Question 1

#### Goal

Find the registrar IANA ID for `inlanefreight.com`.

#### Command

```bash
whois inlanefreight.com
```

#### What to note

Look for the `Registrar IANA ID` field.

#### Answer

`468`

### Question 2

#### Goal

Identify the HTTP server software.

#### Command

```bash
curl -I http://inlanefreight.htb:32311
```

#### What to note

Focus on the `Server` header.

Example:

```
Server: nginx/1.26.1
```

#### Answer

`nginx`

### Question 3

#### Goal

Find the API key in the hidden admin directory.

#### Enumerate virtual hosts

Use a large subdomain wordlist:

```bash
ffuf -u http://inlanefreight.htb:32311 -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt -mc 200,403 -t 60 -H "Host: FUZZ.inlanefreight.htb" -ac
```

Use `-ac` for auto calibration.

This finds:

```
web1337                 [Status: 200, Size: 104, Words: 4, Lines: 2, Duration: 24ms]
```

You can also use `gobuster`:

```bash
gobuster vhost -u http://inlanefreight.htb:32311 -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt -t 60 --append-domain
```

#### Add the discovered host

```bash
sudo sh -c "echo '154.57.164.67 web1337.inlanefreight.htb' >> /etc/hosts"
```

#### Enumerate paths on the new host

```bash
ffuf -u http://web1337.inlanefreight.htb:32311/FUZZ -w /usr/share/seclists/Discovery/Web-Content/common.txt -t 60
```

This reveals `robots.txt`.

#### Inspect `robots.txt`

```bash
curl http://web1337.inlanefreight.htb:35684/robots.txt
```

Key output:

```
User-agent: *
Allow: /index.html
Allow: /index-2.html
Allow: /index-3.html
Disallow: /admin_h1dd3n
```

#### Check the hidden path

```bash
curl -I http://web1337.inlanefreight.htb:32311/admin_h1dd3n
```

You should see a redirect:

```
HTTP/1.1 301 Moved Permanently
Server: nginx/1.26.1
Location: http://web1337.inlanefreight.htb/admin_h1dd3n/
```

Then open:

`http://web1337.inlanefreight.htb:35684/admin_h1dd3n/`

#### Answer

The API key is shown on that page.

### Question 4

#### Goal

Find the email address exposed during crawling.

#### Enumerate deeper virtual hosts

Check for subdomains under `web1337.inlanefreight.htb`:

```bash
ffuf -u http://web1337.inlanefreight.htb:35684 -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt -mc 200,403 -t 60 -H "Host: FUZZ.web1337.inlanefreight.htb" -ac
```

This reveals `dev`.

#### Add the development host

```bash
sudo sh -c "echo '94.237.61.84 dev.web1337.inlanefreight.htb' >> /etc/hosts"
```

#### Crawl the dev host

```bash
python3 ReconSpider.py http://dev.web1337.inlanefreight.htb:35684
```

This writes the crawl results to `results.json`.

#### Answer

Open `results.json` and read the full email address from the output.

### Question 5

#### Goal

Find the API key the developers plan to change to.

#### Answer

Use the same `results.json` file from Question 4.

The updated API key is listed there.

### Quick recap

Use `WHOIS` for domain ownership data.

Use `curl -I` for server headers.

Use virtual host fuzzing to find `web1337` and `dev`.

Use `robots.txt` and `ReconSpider` to uncover the final answers.
