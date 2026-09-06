---
description: Step-by-step workflow for local privilege escalation after initial access.
---

# WF-Privilege Escalation

Use this page when you already have low-privilege access and need to move to `root` or `SYSTEM`.

### Quick loop

1. Switch to local enumeration.
2. Collect host data.
3. Check the common vectors.
4. Validate the safest path.
5. Escalate and keep access.

{% stepper %}
{% step %}
### Switch from external to local enumeration

Your first shell is usually restricted.

Stop thinking like an external attacker.

Start looking for local misconfigurations and flaws.

Goal:

* find one strong escalation path
* move from low privilege to `root` or `SYSTEM`
{% endstep %}

{% step %}
### Collect host data carefully

Before you escalate privileges, collect data about the local operating system configuration.

You can do this manually or with focused enumeration scripts.

Common resources:

* `HackTricks`
* `PayloadsAllTheThings`
* `LinPEAS` or `WinPEAS`
* `LinEnum`, `linuxprivchecker`, `Seatbelt`, or `JAWS`

{% hint style="warning" %}
Automated enumeration scripts create heavy noise. Prefer manual enumeration when stealth matters.
{% endhint %}
{% endstep %}

{% step %}
### Check the common vectors first

Focus on the areas where misconfigurations appear most often.

Start with:

* outdated kernels
* `sudo` rights and user privileges
* scheduled tasks and cron jobs
* exposed credentials and password reuse
* readable or writable SSH keys

These checks usually give the fastest path to higher privilege.
{% endstep %}

{% step %}
### Validate the best path

Pick the lowest-risk vector with the clearest signal.

Use the proof already on the host.

Examples:

* run `sudo -l` to review allowed binaries
* verify whether `/etc/crontab` or `/etc/cron.d` is writable
* check whether a cron script runs from a writable path
* test whether a recovered password works with `su -` or `ssh`
* set key permissions with `chmod 600 id_rsa` before SSH use

{% hint style="warning" %}
Kernel exploits carry high risk. Use them only when safer paths fail and the risk is acceptable.
{% endhint %}
{% endstep %}

{% step %}
### Escalate and keep the better access

Use the confirmed vector to move into the higher-privileged context.

Common outcomes:

* run an allowed privileged binary
* hijack a scheduled task
* switch users with reused credentials
* authenticate with a recovered private key
* append a public key to `authorized_keys` when write access exists

If you use a private key, connect with:

```bash
chmod 600 id_rsa
ssh root@<TARGET_IP> -i id_rsa
```
{% endstep %}
{% endstepper %}

### Fast decision rules

* Prefer `sudo`, creds, or keys before kernel exploits.
* If stealth matters, enumerate manually first.
* If a cron job runs a writable script, use that before riskier paths.
* If you recover a password or key, test it immediately.

### Useful references

* [Getting started](../)
