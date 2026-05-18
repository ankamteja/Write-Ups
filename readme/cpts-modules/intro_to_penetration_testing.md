---
description: Core phases, testing models, and engagement rules for a standard pentest.
---

# Introduction to the penetration tester path

## Introduction to the penetration tester path

Use this page as a quick map of a standard pentest engagement.

{% hint style="info" %}
Penetration testing is a controlled exercise. Each phase builds on the last.
{% endhint %}

### Engagement lifecycle

{% stepper %}
{% step %}
### Plan

Define scope, targets, rules, success criteria, and reporting lines.

This is the pre-engagement phase.
{% endstep %}

{% step %}
### Map

Collect OSINT, enumerate infrastructure, and profile services and hosts.

Then review the attack surface for likely weaknesses.
{% endstep %}

{% step %}
### Validate

Test the best attack paths first.

Prioritize options with high success, low complexity, and low damage risk.
{% endstep %}

{% step %}
### Expand

After access, gather internal context, escalate privileges, and move laterally when approved.
{% endstep %}

{% step %}
### Prove

Capture evidence, show impact, and document the full attack chain.
{% endstep %}

{% step %}
### Close

Clean up, report clearly, retest fixes, and remove retained data on schedule.
{% endstep %}
{% endstepper %}

### What a pentester covers

* **Network services** — exposed services, weak configurations, and outdated software.
* **Web applications** — application flaws, insecure logic, and access control issues.
* **Internal systems** — identity abuse, trust paths, segmentation gaps, and privilege escalation.
* **Modern targets** — APIs, cloud assets, mobile apps, thick clients, and IoT where in scope.

### Testing models

{% tabs %}
{% tab title="Assessment style" %}
| Topic    | Vulnerability assessment          | Penetration test                    |
| -------- | --------------------------------- | ----------------------------------- |
| Method   | Mostly automated scanning         | Automated plus manual testing       |
| Strength | Fast coverage of known issues     | Finds chained and contextual issues |
| Weakness | Misses logic and business context | Takes more time and skill           |
| Output   | Potential findings                | Validated findings with impact      |
{% endtab %}

{% tab title="Starting position" %}
| Type     | Starting point           | Typical targets                      | Goal                 |
| -------- | ------------------------ | ------------------------------------ | -------------------- |
| External | Outside the organization | Public sites, VPNs, mail, APIs       | Break the perimeter  |
| Internal | Inside the network       | AD, shares, servers, endpoints, apps | Measure blast radius |

Typical flow:

1. Run an external test.
2. Gain a foothold.
3. Continue from an internal position.
{% endtab %}

{% tab title="Knowledge level" %}
| Type        | What you know                     | Notes                                   |
| ----------- | --------------------------------- | --------------------------------------- |
| Black box   | Almost nothing                    | Most realistic. Usually hardest.        |
| Grey box    | Partial knowledge                 | Common for client work.                 |
| White box   | Full knowledge                    | Fastest route to deep coverage.         |
| Red team    | Full attack simulation            | May include phishing or physical paths. |
| Purple team | Shared attacker and defender work | Improves detection and response.        |
{% endtab %}
{% endtabs %}

### Operating rules

* **Risk management** uses four responses: accept, transfer, avoid, or mitigate.
* **Safe exploitation** starts in a matching lab when a public proof is not trusted yet.
* **Client approval** is required before disruptive testing, persistence, or real data exfiltration.
* **Evidence quality** matters as much as initial access. Every finding needs impact and reproduction.

{% hint style="warning" %}
Never guess on production systems. If a test may cause damage, stop and get approval first.
{% endhint %}

### After access

Once you have a foothold, priorities change.

* Map local network structure and trust relationships.
* Pillage approved sources such as configs, tokens, keys, and internal documents.
* Escalate privileges to `root`, `SYSTEM`, or higher-value domain access.
* Test lateral movement paths through pivoting, credential reuse, and trust abuse.
* Preserve clean evidence with commands, screenshots, and logs.

<details>

<summary>Examples of post-access activity</summary>

Useful follow-on actions often include:

* Reviewing shares, printers, routing paths, and management channels.
* Checking VPN configs, ARP data, interface data, and saved credentials.
* Using techniques such as pass-the-hash or NTLM capture workflows when they are in scope.

Password spraying also appears in real engagements. It tests one password against many accounts to reduce lockout risk, but it extends the detection window.

</details>

### Reporting and closeout

Good output is clear, reproducible, and tied to root cause.

* Show the full attack chain, not just a one-off exploit.
* Explain why the issue exists and how to fix it permanently.
* Include an executive summary, findings, evidence, and remediation guidance.
* Retest fixed issues and confirm what is resolved.
* Keep client data encrypted and retain it only as long as needed.

{% hint style="success" %}
The best proof of concept explains the weakness, the impact, and the durable fix.
{% endhint %}
