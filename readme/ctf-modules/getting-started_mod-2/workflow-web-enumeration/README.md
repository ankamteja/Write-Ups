---
description: Step-by-step workflow for web enumeration.
---

# Workflow-Web Enumeration

Use this page when a web service is the best first target.

### Quick loop

1. Confirm the site loads.
2. Fingerprint the stack.
3. Find hidden content.
4. Expand the surface.
5. Prioritize the best lead.

{% stepper %}
{% step %}
### Start in the browser

Browse to `http://<TARGET_IP>` and `https://<TARGET_IP>`.

Write down:

* page title
* redirects
* login pages
* anything unusual
{% endstep %}

{% step %}
### Pull headers and fingerprint

Check headers first.

Then fingerprint the stack.

```bash
curl -IL https://<TARGET_HOST>
whatweb <TARGET_IP>
```

Write down:

* server type
* version numbers
* frameworks
* auth methods
{% endstep %}

{% step %}
### Check easy browser wins

Review `robots.txt`.

Review page source.

Check the certificate on HTTPS.

Goal:

* hidden paths
* comments and keys
* names and domains
{% endstep %}

{% step %}
### Brute-force content

Start with directories.

If you have a domain, expand to subdomains.

```bash
gobuster dir -u http://<TARGET_IP>/ -w /usr/share/seclists/Discovery/Web-Content/common.txt
gobuster dns -d <DOMAIN> -w /usr/share/SecLists/Discovery/DNS/namelist.txt
```

Track:

* `200` pages
* `301` redirects
* `403` forbidden paths
{% endstep %}

{% step %}
### Pick the best lead and go deeper

Stay on the finding most likely to give you access.

Good next steps:

* WordPress or CMS paths
* admin panels
* uploads
* dev or staging subdomains
{% endstep %}
{% endstepper %}

### Fast decision rules

* If the site redirects, follow the redirect first.
* If `robots.txt` exposes paths, browse them before brute-forcing harder.
* If Gobuster finds a CMS path, switch to targeted enumeration.
* If HTTPS exposes names or domains, reuse them in vhost and subdomain checks.

### Useful references

* [Gobuster](../../../cheat-sheets/gobuster.md)
* [cURL](../../../cheat-sheets/curl.md)
* [WhatWeb](../../../cheat-sheets/whatweb.md)
* [robots.txt](../../../cheat-sheets/robots.txt.md)
* [HTTP status codes](../../../cheat-sheets/http-status-codes.md)
