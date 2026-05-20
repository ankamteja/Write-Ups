# Introduction to the penetration tester path

This page outlines the standard penetration testing workflow.

<figure><img src="../../.gitbook/assets/image (13).png" alt=""><figcaption></figcaption></figure>

### Core phases

* **Pre-engagement** — define scope, rules, and contracts.
* **Information gathering** — collect recon and map targets.
* **Vulnerability assessment** — identify weaknesses and gaps.
* **Exploitation** — use weaknesses to gain access.
* **Post-exploitation** — escalate privileges and deepen access.
* **Lateral movement** — move through the internal network.
* **Proof of concept** — document and prove each issue.
* **Post-engagement** — clean up and report findings.

### Exploitation focus

Exploitation usually falls into two tracks:

* **Network services** — find misconfigurations and exploit them.
* **Web applications** — find vulnerabilities and exploit them.

### What each phase means

* **Pre-engagement** — understand the scope and required documentation.
* **Information gathering** — collect the context needed to find weaknesses.
* **Vulnerability assessment** — identify issues that support exploitation or later stages.
* **Exploitation** — gain access to the target system.
* **Post-exploitation** — escalate privileges after initial access.
* **Lateral movement** — move across connected systems.
* **Proof of concept** — document the issue and show impact.
* **Post-engagement** — deliver the report to the client.

### Risk management

Risk management follows a simple flow:

Find risks → evaluate impact and likelihood → reduce them where possible.

Common responses:

* **Accept** — live with the risk when impact and likelihood are low.
* **Transfer** — shift the risk through insurance or outsourcing.
* **Avoid** — stop doing the risky activity.
* **Mitigate** — reduce the risk with security controls.

### Pentester responsibilities

#### What pentesters do

* Find vulnerabilities.
* Document them clearly.
* Show how to reproduce them.
* Recommend how to fix them.

#### What pentesters do not do

* Fix the vulnerabilities.
* Apply patches.
* Change the client's code.
* Monitor systems long term.

### Vulnerability assessment vs penetration test

#### Vulnerability assessment

* Uses automated tools such as Nessus or OpenVAS.
* Focuses on known vulnerabilities.
* Runs quickly.
* Misses custom or chained issues.

#### Penetration test

* Combines automated and manual testing.
* Uses human judgment and creativity.
* Adapts to the target environment.
* Finds issues tools alone often miss.

### External vs internal pentest

#### External pentest

* Performed from outside the company.
* Targets internet-facing assets.
* Common targets include websites, VPNs, login portals, mail servers, and public APIs.

#### Internal pentest

* Performed from inside the company network.
* May start on-site or through VPN access after compromise.
* Targets internal servers, Active Directory, databases, file shares, internal apps, and employee devices.

In simple terms:

* **External** means you start as an outsider.
* **Internal** means you already have a foothold and test how far you can go.

Typical sequence:

1. Break through the perimeter.
2. Gain access to the network.
3. Test internal reach and impact.

### Types of penetration testing

* **Black box** — no prior knowledge. Most realistic. Often starts with only an IP or domain.
* **Grey box** — limited knowledge. Common in real engagements.
* **White box** — full knowledge, including code, credentials, and configs.
* **Red team** — full attack simulation. May include physical intrusion and social engineering.
* **Purple team** — attackers and defenders work together to improve detection and response.

#### Common real-world mapping

* **Bug bounty** — usually black box.
* **Most pentests** — usually grey box.
* **Code audit** — usually white box.
* **Advanced simulation** — usually red team.

### Types of testing environments

* **Network** — routers, switches, and infrastructure.
* **Web app** — websites and web applications.
* **Mobile** — Android and iOS apps.
* **API** — backend endpoints.
* **Thick client** — desktop applications.
* **IoT** — smart devices and cameras.
* **Cloud** — AWS, Azure, or GCP environments.
* **Source code** — code review for vulnerabilities.
* **Physical** — locks, access cards, and servers.
* **Employees** — phishing and social engineering tests.
* **Hosts and servers** — individual machines.
* **Firewalls and IDS** — security device misconfigurations.

### Password spraying

Password spraying means trying one password against many usernames.

The goal is to find one valid login without triggering lockouts.

Example:

* Try `Winter2026!` against many users.
* Avoid repeated attempts on one account.
* Look for one successful authentication.

### Structure of information gathering

#### OSINT

Gather public information before touching the target.

Common sources:

* Google
* GitHub
* LinkedIn
* DNS lookups

Goal:

* Find leaked credentials.
* Discover subdomains.
* Identify employees and public assets.

#### Infrastructure enumeration

Map the company’s online presence.

Examples:

* DNS servers
* Mail servers
* Web servers
* Cloud assets

Goal:

* Understand the full attack surface.

#### Service enumeration

For each discovered server, identify:

* Running services such as FTP, SSH, HTTP, SMB, or RDP.
* Service versions.
* Weak or outdated components.

Goal:

* Find exploitable services.

#### Host enumeration

For each host, identify:

* Operating system
* Open ports
* Role in the network
* Connected systems

Goal:

* Find vulnerable or misconfigured hosts.

#### Pillaging

Pillaging happens after exploitation.

You already have access. Now collect useful data such as:

* Passwords
* Config files
* SSH keys
* Credentials
* Employee data
* Scripts

Goal:

* Gather data and material for the next attack step.

### Vulnerability assessment

Vulnerability assessment can be viewed through four lenses:

* **Descriptive** — what do I see?
* **Diagnostic** — why does this exist?
* **Predictive** — what could happen if it is exploited?
* **Prescriptive** — what should be done next?

### Exploitation

When choosing what to exploit first, consider:

1. **Probability of success** — how likely it is to work.
2. **Complexity** — how hard it is to execute.
3. **Probability of damage** — how likely it is to break something.

General rule:

* High success, low complexity, and low damage risk go first.
* Low success, high complexity, and high damage risk go last or not at all.

If no working proof of concept is available:

1. Build a local VM that mirrors the target.
2. Match the OS and service versions.
3. Test the exploit locally.
4. Confirm it works safely.
5. Use it on the real target only after validation.

If you are unsure about impact:

* Ask the client first.
* Never guess with production systems.

### Post-exploitation

Once you gain access, detection risk increases.

One careless command can trigger monitoring.

#### Evasive testing modes

* **Evasive** — stay fully hidden.
* **Non-evasive** — stealth is not a priority.
* **Hybrid** — start quietly, then increase activity as needed.

#### Internal information gathering

Internal visibility is very different from external recon.

Look for:

* Local network structure
* Connected machines
* Printers, databases, and file shares
* Communication paths to other systems

#### Pillaging targets

Collect useful artifacts such as:

* Passwords in config files
* SSH keys
* Scripts with hardcoded credentials
* Sensitive office files
* Emails
* Network interfaces and routes
* VPN configurations
* ARP tables

#### Persistence

**Persistence** in cybersecurity refers to the techniques and mechanisms employed by threat actors to maintain long-term access to a compromised system or network, ensuring their presence survives system reboots, user logoffs, and credential changes.

If your access drops, you may lose the foothold.

That is why persistence often comes first.

Why it matters:

* The original exploit may stop working.
* The service may crash or get restarted.
* Persistence gives you a way back in.

#### Privilege escalation

Initial access is often limited.

The next goal is higher privilege:

* **Linux** — get `root`.
* **Windows** — get `SYSTEM` or Domain Admin.

Common paths:

* Exploit a local vulnerability.
* Find stored credentials for a higher-privileged account.

#### Data exfiltration

The question is not only whether data can be stolen.

It is also whether security controls detect it.

Best practice:

* Avoid exfiltrating real sensitive data.
* Create fake data such as fake credit cards or fake SSNs.
* Attempt to exfiltrate the fake data.
* Measure whether DLP or EDR detects it.

Always get client approval before exfiltration.

Screenshots and recordings can support proof safely.

#### Typical post-exploitation flow

Get access → establish persistence → gather internal info → pillage useful data → assess internal weaknesses → escalate privileges → exfiltrate approved test data → move laterally.

### Lateral movement

Real attackers rarely stop at one machine.

They move across the environment and expand impact.

Typical activities:

1. **Pivoting** — use one compromised host to reach others.
2. **Evasive testing** — avoid IPS, IDS, and EDR alerts.
3. **Information gathering** — map the internal network.
4. **Vulnerability assessment** — identify internal weaknesses.
5. **Exploitation** — attack more systems.
6. **Post-exploitation** — repeat the process on each new host.

Common techniques:

* **Pass-the-Hash** — use a stolen password hash without cracking it.
* **Responder** — capture NTLMv2 hashes and reuse them where possible.

Example attack chain:

Compromise a web server → pivot into the internal network → reach a database server → reuse or crack credentials → find domain admin access → compromise Active Directory.

### Proof of concept

A proof of concept shows that the vulnerability is real.

It can include:

* Step-by-step documentation
* A script or code sample

A good proof of concept does more than prove exploitability.

It should also explain the root cause.

Why this matters:

* Fixing only the exact script path is not enough.
* The underlying weakness may still exist.

Example:

* **Issue** — Domain Admin uses `Password123`.
* **Wrong fix** — change one password.
* **Right fix** — improve the overall password policy.

A strong proof of concept should include:

* The full attack chain
* Root cause for each finding
* Clear remediation guidance
* The impact of fixing one step versus the whole issue

### Post-engagement

#### 1. Cleanup

* Delete tools and scripts from target systems.
* Revert any changes you made.
* Document all changes.

#### 2. Draft report

Include:

* Full attack chain
* Executive summary
* Findings with risk ratings
* Reproduction steps
* Appendices such as scope, compromised accounts, transferred files, and scan data

#### 3. Report review meeting

* Walk through the findings.
* Answer questions.
* Clarify anything unclear.

#### 4. Final report

* Collect client feedback on the draft.
* Update the report.
* Issue the final version.

#### 5. Post-remediation testing

* Retest the fixes.
* Confirm whether each issue is resolved.
* Deliver a post-remediation report.

#### 6. Data retention

* Keep client data encrypted.
* Wipe local copies after the engagement.
* Retain data only as long as required.

#### 7. Closeout

* Wipe engagement systems.
* Invoice the client.
* Send a satisfaction survey.
