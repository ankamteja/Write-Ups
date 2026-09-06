---
description: Step-by-step workflow for gaining and stabilizing a reverse shell.
---

# WF-Reverse Shell

Use this page when you already have code execution and the target can connect back to your machine.

### Quick loop

1. Start a listener.
2. Confirm your callback IP.
3. Trigger the reverse shell command.
4. Confirm the connection.
5. Upgrade the shell.

{% stepper %}
{% step %}
### Start a listener

Listen on a port you control.

```bash
nc -lvnp 4444
```

Your machine waits for the target to connect back.
{% endstep %}

{% step %}
### Confirm the callback IP

Use the IP the target can actually reach.

```bash
ip a
```

Use the VPN interface IP when the target is only reachable through the lab VPN.
{% endstep %}

{% step %}
### Trigger the reverse shell command

Run the reverse shell command through your exploit path.

Examples:

* remote code execution
* a Python exploit
* a Metasploit module

The exact command depends on the target OS and the tools available on the host.
{% endstep %}

{% step %}
### Confirm the connection

If the command works, your listener receives the shell.

At this stage, you have basic interactive access.
{% endstep %}

{% step %}
### Upgrade the shell

Many `netcat` shells are unstable and lack job control.

Upgrade the shell as soon as you get access.

```bash
python -c 'import pty; pty.spawn("/bin/bash")'
```

Then background the shell with `Ctrl+Z` and fix the terminal:

```bash
stty raw -echo; fg
export TERM=xterm
stty rows <ROWS> columns <COLS>
```
{% endstep %}
{% endstepper %}

### Fast decision rules

* If the target cannot reach you, use a bind shell or web shell.
* If the shell is unstable, upgrade it before doing more work.
* If the screen looks blank after `fg`, press `Enter` or run `reset`.

### Useful references

* [Getting started](./)
* [WF-Service Enumeration](wf-service-enumeration.md)
* [Payload All The Things reverse shell cheat sheet](https://swisskyrepo.github.io/InternalAllTheThings/cheatsheets/shell-reverse-cheatsheet/)
