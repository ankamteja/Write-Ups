---
description: Step-by-step workflow for gaining and validating a bind shell.
---

# WF-Bind Shell

Use this page when you already have code execution and the target can expose a listening port.

### Quick loop

1. Pick a port.
2. Start the bind shell.
3. Connect to the port.
4. Connect to the shell.
5. Decide whether to keep using it.

{% stepper %}
{% step %}
### Pick a reachable port

With a bind shell, the target opens a listening port.

In the example, the target listens on `0.0.0.0:1234`.
{% endstep %}

{% step %}
### Start the bind shell

Choose a bind shell command that fits the target OS and available tools.

The goal is simple:

* make the target listen
* expose a shell on that port
* connect to it from your machine
{% endstep %}

{% step %}
### Connect to the port

After the target starts listening, connect to that port with `netcat`.
{% endstep %}

{% step %}
### Connect to the shell

Use `netcat` to connect.

```bash
nc -nv <TARGET_IP> 1234
```

If the shell responds, you have direct access to the target shell.
{% endstep %}

{% step %}
### Decide whether to keep using it

Bind shells can be convenient when they stay running.

If your connection drops, you may reconnect without rerunning the exploit.

If the process stops or the target reboots, you still lose access.
{% endstep %}
{% endstepper %}

### Fast decision rules

* If inbound firewall rules block the port, use a reverse shell instead.
* If the bind shell stays running, reconnect instead of rerunning the exploit.
* If access is unstable, use the bind shell only as a short bridge.

### Useful references

* [Getting started](./)
* [Payload All The Things bind shell cheat sheet](https://swisskyrepo.github.io/InternalAllTheThings/cheatsheets/shell-bind-cheatsheet/)
