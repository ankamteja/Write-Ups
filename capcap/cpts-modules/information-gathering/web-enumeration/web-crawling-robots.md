---
description: Crawl websites, inspect robots.txt, and use .well-known URIs during web recon.
---

# Web Crawling/Robots

## Web crawling and robots

Use this page to find hidden paths, crawl websites, and collect recon clues.

Focus on `robots.txt`, `/.well-known/`, and crawl output.

### Quick workflow

1. Fetch `robots.txt`.
2. Check useful `/.well-known/` endpoints.
3. Crawl the site from a seed URL.
4. Review links, files, metadata, and comments.
5. Manually inspect anything high value.

### Web crawling

#### What crawling is

`Crawling`, or `spidering`, is automated browsing.

A crawler starts from a seed URL.

It fetches the page, extracts links, and follows them.

This differs from fuzzing.

Fuzzing guesses paths.

Crawling follows discovered paths.

#### How crawling works

1. Start from a homepage or seed URL.
2. Extract links from that page.
3. Add those links to a queue.
4. Visit each link and repeat.

Example:

```
Homepage
├── link1
├── link2
└── link3
```

If `link1` exposes more paths, the crawler keeps expanding:

```
link1 Page
├── Homepage
├── link2
├── link4
└── link5
```

#### Crawl strategies

**Breadth-first crawling**

![Flowchart showing a Seed URL leading to Page 1, which branches to Page 2 and Page 3. Page 2 connects to Page 4 and Page 5, while Page 3 connects to Page 6 and Page 7.](https://cdn.services-k8s.prod.aws.htb.systems/content/modules/144/ig_crawling_1.png)

`Breadth-first` crawling explores wide first.

It visits all links on the current level.

Then it moves deeper.

Use this when you want a fast site map.

**Depth-first crawling**

![Flowchart showing a Seed URL leading to Page 1, then to Page 2. Page 2 connects to Page 3, which branches to Page 4 and Page 5.](https://mermaid.ink/svg/pako:eNo9zz0PgjAQBuC_0twsg18LgwlfGyYG4uQ5VHoC0RZS2sEQ_rsnTezU98mlvXeGZlAEMbRWjp0oKzSTf4RQEylxrUo0gk9yu8iWxPaOhoxCk4goOok06I41XSELsGfIVsgDHBjyFYoAR4YivCEEGtiAJqtlr3iZ-fclgutIE0LMVyXtCwHNwnPSu6H-mAZiZz1twA6-7SB-yvfEyY9KOsp7ySX0X0n1brDn0HWtvHwB2SFOww)

`Depth-first` crawling explores one branch first.

It keeps going until it must backtrack.

Use this when you want deep content quickly.

#### What to extract

Crawlers can collect:

* `Internal and external links`
* `Comments`
* `Metadata`
* `Sensitive files`

High-value file types include:

* backups such as `.bak` and `.old`
* configs such as `web.config` and `settings.php`
* logs such as `error_log` and `access_log`

These often expose credentials, keys, or source snippets.

#### Why context matters

Single findings rarely matter alone.

Clusters of findings do.

Examples:

* comments mention a software version
* metadata shows the version is old
* a backup file exposes related config

That pattern is often more useful than any one clue.

Another common example is a repeated path like `/files/`.

If many extracted links point there, inspect it manually.

Directory listing or exposed backups may sit behind it.

### robots.txt

`robots.txt` is a text file in the site root.

Common location:

```
https://example.com/robots.txt
```

It tells crawlers what they should or should not fetch.

It is guidance, not access control.

{% hint style="info" %}
`robots.txt` does not protect content. It only signals crawler behavior.
{% endhint %}

#### Basic example

```
User-agent: *
Disallow: /private/
```

This tells all crawlers to avoid `/private/`.

#### Structure

Each record has:

1. a `User-agent`
2. one or more directives

Common directives:

| Directive     | What it does                         | Example                                    |
| ------------- | ------------------------------------ | ------------------------------------------ |
| `Disallow`    | Blocks a path from crawling          | `Disallow: /admin/`                        |
| `Allow`       | Permits a path under a broader block | `Allow: /public/`                          |
| `Crawl-delay` | Slows repeated requests              | `Crawl-delay: 10`                          |
| `Sitemap`     | Points to an XML sitemap             | `Sitemap: https://example.com/sitemap.xml` |

#### Why it matters

Respecting `robots.txt` helps you:

* avoid unnecessary load
* stay within expected crawler behavior
* identify areas the owner considers sensitive

#### Recon value

For web recon, `robots.txt` often reveals:

* hidden directories
* admin routes
* backup or private paths
* trap paths for bot detection

#### Example analysis

```
User-agent: *
Disallow: /admin/
Disallow: /private/
Allow: /public/

User-agent: Googlebot
Crawl-delay: 10

Sitemap: https://www.example.com/sitemap.xml
```

What this suggests:

* `/admin/` may host an admin panel
* `/private/` may hold sensitive content
* `/public/` is intentionally exposed
* the sitemap may reveal more URLs fast

### Well-known URIs

`/.well-known/` is a standard root directory for metadata.

It is defined in [RFC 8615](https://datatracker.ietf.org/doc/html/rfc8615).

Clients use it to discover settings, policies, and service endpoints.

Example:

```
https://example.com/.well-known/security.txt
```

The IANA registry tracks standard well-known URIs:

[IANA well-known URI registry](https://www.iana.org/assignments/well-known-uris/well-known-uris.xhtml)

#### Useful examples

| URI                    | Why it matters                  |
| ---------------------- | ------------------------------- |
| `security.txt`         | Security contact details        |
| `change-password`      | Standard password change path   |
| `openid-configuration` | OpenID Connect discovery data   |
| `assetlinks.json`      | App and domain ownership checks |
| `mta-sts.txt`          | Mail transport security policy  |

#### Recon value

These endpoints can reveal:

* exposed service configuration
* authentication endpoints
* key locations
* supported protocols and features

#### Example: `openid-configuration`

This endpoint is often high value:

```
https://example.com/.well-known/openid-configuration
```

Example response:

```json
{
  "issuer": "https://example.com",
  "authorization_endpoint": "https://example.com/oauth2/authorize",
  "token_endpoint": "https://example.com/oauth2/token",
  "userinfo_endpoint": "https://example.com/oauth2/userinfo",
  "jwks_uri": "https://example.com/oauth2/jwks",
  "response_types_supported": ["code", "token", "id_token"],
  "subject_types_supported": ["public"],
  "id_token_signing_alg_values_supported": ["RS256"],
  "scopes_supported": ["openid", "profile", "email"]
}
```

What to review first:

1. `authorization_endpoint`
2. `token_endpoint`
3. `userinfo_endpoint`
4. `jwks_uri`

Also note supported scopes, response types, and signing algorithms.

### Crawling tools

#### Common options

* `Burp Suite Spider` for active application mapping
* `OWASP ZAP` for free crawling and proxy-based testing
* `Scrapy` for custom and scalable crawlers
* `Apache Nutch` for large crawls

{% hint style="warning" %}
Only crawl targets you are allowed to test.
{% endhint %}

### Scrapy

`Scrapy` is useful when you need a custom crawler.

It works well for structured recon tasks.

#### Install Scrapy

```bash
pip3 install scrapy
```

#### Download and extract `ReconSpider`

```bash
wget -O ReconSpider.zip https://academy.hackthebox.com/storage/modules/144/ReconSpider.v1.2.zip
unzip ReconSpider.zip
```

#### Run the spider

```bash
python3 ReconSpider.py http://inlanefreight.com
```

Replace `inlanefreight.com` with your target.

The spider crawls the site and writes `results.json`.

### Understanding `results.json`

Example structure:

```json
{
  "emails": [
    "lily.floid@inlanefreight.com",
    "cvs@inlanefreight.com"
  ],
  "links": [
    "https://www.themeansar.com",
    "https://www.inlanefreight.com/index.php/offices/"
  ],
  "external_files": [
    "https://www.inlanefreight.com/wp-content/uploads/2020/09/goals.pdf"
  ],
  "js_files": [
    "https://www.inlanefreight.com/wp-includes/js/jquery/jquery-migrate.min.js?ver=3.3.2"
  ],
  "form_fields": [],
  "images": [
    "https://www.inlanefreight.com/wp-content/uploads/2021/03/AboutUs_01-1024x810.png"
  ],
  "videos": [],
  "audio": [],
  "comments": [
    "<!-- #masthead -->"
  ]
}
```

Key fields:

| JSON key         | What it shows                   |
| ---------------- | ------------------------------- |
| `emails`         | Email addresses on the target   |
| `links`          | URLs discovered during crawling |
| `external_files` | Documents and other files       |
| `js_files`       | JavaScript assets               |
| `form_fields`    | Collected input fields          |
| `images`         | Image URLs                      |
| `videos`         | Video URLs                      |
| `audio`          | Audio URLs                      |
| `comments`       | HTML comments from source       |

#### What to inspect first

Prioritize:

* `links` for hidden routes
* `external_files` for documents and backups
* `js_files` for API paths and hardcoded values
* `comments` for hints and internal notes

### Quick summary

Start with `robots.txt` and `/.well-known/`.

Then crawl the site from a seed URL.

Use the results to spot hidden paths, files, and auth endpoints.
