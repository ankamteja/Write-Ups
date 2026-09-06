# SNMP

### Key notes

* **What SNMP is:** An application-layer protocol for monitoring and managing network devices.
* **Key ports:** UDP `161` for queries and UDP `162` for traps.
* **Auth model:** SNMPv1 and SNMPv2c rely on community strings.
* **Main risk:** Older versions expose data in plaintext.
* **Main tools:** `onesixtyone`, `snmpwalk`, and `braa`.

#### Core Concepts & Architecture

* What it is: An application-layer protocol used to monitor, manage, and remotely alter configurations on network hardware (routers, switches, servers, IoT devices).
* Key Ports:
  * `UDP 161`: Core service port where the client actively queries or sends control commands to the SNMP agent.
  * `UDP 162`: Used for SNMP Traps (Unsolicited automated alerts sent _from_ the server _to_ the client when a critical error or event triggers).

**Decoding the Structural Components (MIB vs. OID)**

* MIB (Management Information Base): A plain-text reference map layout file. It contains no data; it simply acts as the schematic index defining what can be monitored on a device.
* OID (Object Identifier): A specific, numeric node address mapped inside a standardized, dot-notated tree hierarchy (e.g., `.1.3.6.1.2.1.1.1.0`). The deeper the numeric chain goes, the more specific the system item it queries (like a specific software package version, uptime status, or system admin email address).

#### Protocol Version Vulnerabilities

* SNMPv1: Outdated and highly insecure. It passes all information in plaintext and lacks robust security blocks.
* SNMPv2c: Community-Based SNMP. The absolute standard version found in production networks today. It adds performance improvements but retains the major flaw of transmitting the access password (Community String) over the network wire in raw plaintext.
* SNMPv3: Secure implementation. Introduces complete transmission encryption (via pre-shared keys) and cryptographic username/password credential authentication. It is highly secure but much more complex to configure.

#### Dangerous Configurations (`snmpd.conf`)

* `rwcommunity <string>` or `rwuser noauth`: Granting Read-Write access blocks instead of traditional Read-Only (`rocommunity`) status. This enables unauthenticated or guest networks to execute remote control commands, altering system settings or wiping device configurations.

#### Tooling & Footprinting Reference

**1. Password Brute-Forcing (`onesixtyone`)**

Because SNMPv1 and v2c do not use standard usernames, access relies entirely on guessing the Community String (passwords like `public`, `private`, `internal`).

{% code title="onesixtyone" %}
```bash
# High-speed parallel community string brute-forcing using SecLists wordlists
onesixtyone -c /usr/share/seclists/Discovery/SNMP/snmp.txt <TARGET_IP>
```
{% endcode %}

* High-Value Discovery: If this tool returns `[public]` or another active identifier string alongside the system banner, you have successfully unlocked data extraction access.

**2. Mass Data Extraction (`snmpwalk`)**

Once you find a working community string, use `snmpwalk` to traverse the tree hierarchy and dump information:

{% code title="snmpwalk" %}
```bash
# Bulk-query the entire OID tree using SNMPv2c (-v2c) and community string (-c)
snmpwalk -v2c -c public <TARGET_IP>
```
{% endcode %}

* Loot Targets Inside the Snmpwalk Dump:
  * `.1.3.6.1.2.1.1.4.0` ──> System Administrator Contact Info (Email addresses).
  * `.1.3.6.1.2.1.1.5.0` ──> System Hostname.
  * `.1.3.6.1.2.1.25.4.2.1.2` ──> Running System Processes.
  * `.1.3.6.1.2.1.25.6.3.1.2` ──> Comprehensive list of installed software packages (reveals exact software versions vulnerable to exploitation).

**3. Mass OID Querying (`braa`)**

`snmpwalk` queries nodes one by one, which can be slow over large networks. `braa` allows you to target massive chunks of the OID namespace simultaneously:

{% code title="braa" %}
```bash
# Rapidly query a specific range structure recursively using wildcards (*)
braa public@<TARGET_IP>:.1.3.6.*
```
{% endcode %}

#### Post-Enumeration Priorities

* Identify Exploit Vectors: Carefully inspect the installed software versions leaked by the dump (`iso.3.6.1.2.1.25.6...`). Look for outdated applications (like old web servers, FTP, or SSH builds) running on the machine that you can exploit later.
* Harvest Credentials: Review the boot image configurations, environment variables, or host descriptions inside the data dump for hardcoded passwords or sensitive network architecture layout hints.
