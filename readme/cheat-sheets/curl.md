---
description: Quick header checks for web servers.
---

# cURL

Use this page as a quick command reference for `curl` during web enumeration.

Use this when you need to inspect headers, redirects, or basic server behavior.

### Read response headers

```bash
curl -IL https://<TARGET_HOST>
```

### Flag reminder

* `-I` requests headers only.
* `-L` follows redirects.

### What to note

* server type and version
* technology stack hints
* auth prompts and redirects
