---
description: Use WHOIS, search engines, and web archives to gather passive web recon data.
---

# Passive Web Enumeration

## Passive web enumeration

Use this page to collect web intelligence without touching the target host.

Focus on `WHOIS`, search engines, and web archives.

### Quick workflow

1. Run a `WHOIS` lookup.
2. Record ownership, registrar, and name server data.
3. Use search operators to find exposed content.
4. Check archived versions of the target.
5. Compare old and current content for hidden paths.

### WHOIS lookup

```bash
whois facebook.com
```

### What to note

Focus on:

* domain name and registrar
* creation, update, and expiration dates
* registrant, admin, and technical contacts
* name servers and other DNS clues

### Why it matters

* identifies people, teams, or vendors tied to the domain
* reveals infrastructure details such as name servers
* helps track ownership or configuration changes over time

### Historical WHOIS

Use [WhoisFreaks](https://whoisfreaks.com/) when you need older WHOIS records.

Historical records help you spot:

* ownership changes
* registrar changes
* contact changes
* DNS or name server changes

### Search engine discovery

Search engines are a powerful OSINT source.

They help you navigate a huge amount of public information fast.

They also expose data that is easy to miss during direct browsing.

Use them to uncover:

* login portals
* exposed files
* employee references
* cached content
* hidden or forgotten pages
* sensitive documents
* exposed credentials

Search engine discovery is passive.

It relies on already indexed content rather than direct interaction with the target.

#### Why it matters

Search engine discovery is useful because it is:

* `Open source`
* `Broad`
* `Easy to use`
* `Free`

It can support:

* security assessment
* threat intelligence
* competitive research
* investigative work

{% hint style="info" %}
Search engines do not index everything. Some content may be hidden, blocked, or removed.
{% endhint %}

Search results also vary by engine.

Syntax differs slightly between providers, but the underlying search logic stays similar.

### Search operators

Use operators to narrow results fast.

#### Operator reference

| Operator               | Description                                  | Example                                             | Example use                                               |
| ---------------------- | -------------------------------------------- | --------------------------------------------------- | --------------------------------------------------------- |
| `site:`                | Limits results to a specific domain or site. | `site:example.com`                                  | Find indexed pages on `example.com`.                      |
| `inurl:`               | Finds terms in the URL.                      | `inurl:login`                                       | Find login pages.                                         |
| `filetype:`            | Finds files by extension.                    | `filetype:pdf`                                      | Find downloadable PDF files.                              |
| `intitle:`             | Finds terms in the page title.               | `intitle:"confidential report"`                     | Find pages with a sensitive title.                        |
| `intext:` or `inbody:` | Finds terms in the page body.                | `intext:"password reset"`                           | Find pages containing reset flows or references.          |
| `cache:`               | Displays a cached page if available.         | `cache:example.com`                                 | Review older indexed content.                             |
| `link:`                | Finds pages that link to a target.           | `link:example.com`                                  | Identify backlinks.                                       |
| `related:`             | Finds similar sites.                         | `related:example.com`                               | Discover related organizations or services.               |
| `info:`                | Shows basic information about a page.        | `info:example.com`                                  | Review title and summary details.                         |
| `define:`              | Returns definitions for terms.               | `define:phishing`                                   | Quick term lookup.                                        |
| `numrange:`            | Finds numbers in a range.                    | `site:example.com numrange:1000-2000`               | Search for values, IDs, or prices in range.               |
| `allintext:`           | Requires all terms in the page body.         | `allintext:admin password reset`                    | Narrow body text results.                                 |
| `allinurl:`            | Requires all terms in the URL.               | `allinurl:admin panel`                              | Find likely admin panels.                                 |
| `allintitle:`          | Requires all terms in the title.             | `allintitle:confidential report 2023`               | Narrow title-based searches.                              |
| `AND`                  | Requires all terms to appear.                | `site:example.com AND (inurl:admin OR inurl:login)` | Combine filters tightly.                                  |
| `OR`                   | Matches either term.                         | `"linux" OR "ubuntu" OR "debian"`                   | Broaden technology searches.                              |
| `NOT`                  | Excludes results with a term.                | `site:bank.com NOT inurl:login`                     | Filter out unwanted pages.                                |
| `*`                    | Wildcard for a word or character.            | `site:socialnetwork.com filetype:pdf user* manual`  | Match variations such as `user guide` or `user handbook`. |
| `..`                   | Numeric range search.                        | `site:ecommerce.com "price" 100..500`               | Find values between two numbers.                          |
| `" "`                  | Exact phrase matching.                       | `"information security policy"`                     | Find exact wording.                                       |
| `-`                    | Excludes terms quickly.                      | `site:news.com -inurl:sports`                       | Remove noisy results.                                     |

#### How to use them well

Start with a domain filter.

Then add one operator at a time.

That keeps results specific and easy to review.

#### High-value examples

Find login pages:

```
site:example.com inurl:login
site:example.com (inurl:login OR inurl:admin)
```

Find exposed documents:

```
site:example.com filetype:pdf
site:example.com (filetype:xls OR filetype:docx)
```

Find config files:

```
site:example.com inurl:config.php
site:example.com (ext:conf OR ext:cnf)
```

Find likely backups:

```
site:example.com inurl:backup
site:example.com filetype:sql
```

#### Google dorking

Google dorking uses search operators to uncover sensitive or useful content.

It is also called Google hacking.

Use it to look for:

* exposed documents
* admin paths
* config files
* old backups

Common patterns include:

* finding login pages
* identifying exposed files
* uncovering configuration files
* locating database backups

For more examples, use the [Google Hacking Database](https://www.exploit-db.com/google-hacking-database).

### Web archives

Archived pages help you inspect older versions of a site.

The main source is the [Wayback Machine](https://web.archive.org/).

In practice, archived snapshots let you revisit how a site looked in the past.

That includes older content, structure, technologies, and exposed resources.

#### What it gives you

Archived snapshots can reveal:

* old pages
* removed files
* retired subdomains
* previous site structure
* older technologies and branding

![Internet Archive Wayback Machine homepage with search bar for web pages, tools like browser extensions, and options for subscription service, collection search, and saving pages.](https://cdn.services-k8s.prod.aws.htb.systems/content/modules/144/wayback.png)

#### What the Wayback Machine is

The Wayback Machine is a digital archive of the web.

It is operated by the Internet Archive.

It has been collecting snapshots of websites since 1996.

Each archived copy is tied to a date and time.

That lets you inspect how a site changed over time.

#### How it works

The archive process has three stages:

1. `Crawling` — bots collect pages and linked resources.
2. `Archiving` — captures are stored with a timestamp.
3. `Accessing` — you browse snapshots by date.

![Flowchart with three steps: Crawling, Archiving, Accessing.](https://cdn.services-k8s.prod.aws.htb.systems/content/modules/144/ig_webarchives_1.png)

Archived content can include:

* HTML
* CSS
* JavaScript
* images
* linked resources

Archive coverage is not uniform.

Some sites are captured often.

Others only have a few snapshots.

Capture frequency depends on:

* site popularity
* how often the site changes
* archive resources

The archive also does not include every page on the internet.

Some content is never captured.

Some site owners also try to exclude their content.

#### Why it matters in recon

Use archives to:

* uncover hidden assets and old endpoints
* track changes in structure or content
* gather older OSINT on staff, products, or services
* inspect targets passively without touching live infrastructure

#### What to compare

When reviewing snapshots, check for:

* old admin or login paths
* removed downloads or PDFs
* legacy JavaScript files
* old subdomains or hostnames
* technology changes over time

#### Why archived content is valuable

Archived content can help you:

1. uncover hidden assets and vulnerabilities
2. track changes and identify patterns
3. gather historical OSINT
4. perform stealthier reconnaissance

An older snapshot may expose:

* retired directories
* removed downloads
* legacy subdomains
* outdated frameworks
* content no longer linked publicly

### Quick summary

Passive web enumeration gives you strong early recon with low noise.

Start with `WHOIS`.

Then use search operators to find indexed content.

Finish with web archives to recover old paths and forgotten assets.
