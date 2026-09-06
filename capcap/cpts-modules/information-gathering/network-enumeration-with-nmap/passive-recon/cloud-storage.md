---
description: Find exposed cloud storage targets and leaked objects during passive recon.
---

# Cloud Storage

Use passive recon to find cloud-hosted storage tied to the target.

Common targets include:

* Amazon S3
* Azure Blob Storage
* Google Cloud Storage

Misconfigurations sometimes expose data without authentication.

High-value findings include:

* public file listings
* backups
* configuration files
* private SSH keys

### Tools

Start with:

* Google dorks
* `domain.glass`
* GrayHatWarfare
* page source review

### Resolve subdomains and look for cloud-hosted endpoints

Use `host` to resolve discovered subdomains.

```bash
for i in $(cat subdomainlist); do host $i | grep "has address" | grep inlanefreight.com | cut -d" " -f1,4; done
```

Example output:

```
blog.inlanefreight.com 10.129.24.93
inlanefreight.com 10.129.27.33
matomo.inlanefreight.com 10.129.127.22
www.inlanefreight.com 10.129.127.33
s3-website-us-west-2.amazonaws.com 10.129.95.250
```

This helps you spot infrastructure that points to cloud-backed services.

An S3 website endpoint is a strong clue that public storage may be in use.

### Find cloud storage in public search results

Google dorks often reveal exposed buckets, file listings, and indexed objects.

#### AWS

<figure><img src="../../../../../.gitbook/assets/image (96).png" alt=""><figcaption></figcaption></figure>

#### Azure

<figure><img src="../../../../../.gitbook/assets/image (97).png" alt=""><figcaption></figcaption></figure>

### Check page source and client-side files

Developers sometimes leave storage URLs in source code, JavaScript files, or configuration snippets.

<figure><img src="../../../../../.gitbook/assets/image (98).png" alt=""><figcaption></figcaption></figure>

### Review DNS and edge exposure

Use `domain.glass` to review DNS, hosting, and edge protection details.

<figure><img src="../../../../../.gitbook/assets/image (99).png" alt=""><figcaption></figcaption></figure>

If a gateway such as Cloudflare is present, note it and continue mapping exposed assets behind it.

### Search known public bucket indexes

GrayHatWarfare can reveal publicly indexed storage objects.

Try:

* the full company name
* common abbreviations
* product names
* internal naming patterns

<figure><img src="../../../../../.gitbook/assets/image (100).png" alt=""><figcaption></figcaption></figure>

### Watch for leaked credentials and keys

Cloud storage leaks become critical when they expose secrets.

Private SSH keys, API keys, and backups can turn simple recon into immediate access.

<figure><img src="../../../../../.gitbook/assets/image (101).png" alt=""><figcaption></figcaption></figure>

{% hint style="warning" %}
Treat exposed keys and credentials as critical findings. Validate access carefully and stay within scope.
{% endhint %}
