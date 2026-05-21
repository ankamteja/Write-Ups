---
description: Step-by-step workflow for CTF and exam-style targets.
---

# Workflow-Service Enumeration

Use this page when you need a repeatable flow for the <mark style="color:$success;">topics covered in</mark> <mark style="color:$success;"></mark><mark style="color:$success;">**Getting started**</mark><mark style="color:$success;">.</mark>

### Quick loop

1. Scan.
2. Fingerprint.
3. Enumerate the best target.
4. Reuse what you find.
5. Move to the next service.

{% stepper %}
{% step %}
### Start with a full scan

Run a quick scan first.

Then run a full TCP scan with service detection.

```bash
nmap <TARGET_IP>
nmap -sV -sC -p- <TARGET_IP>
```

Write down:

* open ports
* service versions
* hostnames and banners
* anything unusual
{% endstep %}

{% step %}
### Pick the best attack surface

Do not attack everything at once.

Pick the service most likely to give you credentials, file access, or code execution.

Good early targets:

* web on `80` or `443`
* SMB on `445`
* FTP on `21`
* SNMP on `161`
{% endstep %}

{% step %}
### Enumerate one service deeply

Go deep before you move wide.

Examples:

* web — directories, vhosts, login pages, uploads, parameters
* SMB — shares, readable files, usernames, configs
* FTP — anonymous access, backups, web content
* SNMP — hostname, users, routes, software info

Goal:

* credentials
* sensitive files
* version-specific weakness
* remote code execution
{% endstep %}

{% step %}
### Reuse every credential

When you find a username or password, test it everywhere it makes sense.

Common reuse paths:

* web to SSH
* SMB to SSH
* config file creds to admin panels or file shares

Keep a small list of:

* usernames
* passwords
* hashes
* hostnames
{% endstep %}

{% step %}
### Move to the next service

If one path stalls, return to the port list.

Pick the next service with the best chance of giving you:

* credentials
* readable files
* version details
* a clear next step
{% endstep %}
{% endstepper %}

### Fast decision rules

* If web looks custom, stay on web longer.
* If SMB gives files, mine them before scanning harder.
* If SNMP leaks users, try them on auth services.
* If one path stalls, return to the port list and pick the next best surface.

### Useful references

* [Nmap](../../cheat-sheets/nmap.md)
* [SMB](../../cheat-sheets/smb.md)
* [SNMP](../../cheat-sheets/snmp.md)
* [Common ports](../../cheat-sheets/page-3.md)
