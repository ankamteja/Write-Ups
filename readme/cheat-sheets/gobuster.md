---
description: Quick directory and DNS brute-force commands.
---

# Gobuster

Use this page as a quick command reference for Gobuster.

Use this when you need to brute-force directories or find subdomains.

### Directory enumeration

```bash
gobuster dir -u http://<TARGET_IP>/ -w /usr/share/seclists/Discovery/Web-Content/common.txt
```

### DNS subdomain enumeration

```bash
gobuster dns -d <DOMAIN> -w /usr/share/SecLists/Discovery/DNS/namelist.txt
```

### Watch for

* `200` content
* `301` redirects
* `403` forbidden paths
