---
description: Hard lab attack path for recovering the `HTB` user password.
---

# Footprinting - hard

Use this page to track the hard footprinting lab path.

### Target

* OS: `Linux`
* Difficulty: `Hard`
* Role: `MX` and management server
* Goal: recover the password for user `HTB`

### Attack chain summary

```
Full TCP scan
    → SSH, POP3, IMAP, IMAPS, and POP3S
    → UDP scan reveals SNMP
    → Bruteforce SNMP community string
    → snmpwalk leaks mailbox credentials
    → Log in to IMAP
    → Read inbox
    → Recover SSH private key
    → SSH to the host
    → Enumerate local users and services
    → Pivot to MySQL-related account data
    → Recover HTB password
```

### Key steps

#### 1. Run TCP and UDP scans

```bash
nmap -p- -sV <target_IP>
sudo nmap -sU --top-ports 1000 <target_IP>
```

**Findings**

* The host exposes `22`, `110`, `143`, `993`, and `995`
* The stack points to a Linux mail server
* UDP scanning reveals `SNMP`, which becomes the best next lead

#### 2. Find the SNMP community string

```bash
onesixtyone -c /usr/share/seclists/Discovery/SNMP/snmp.txt <target_IP>
```

Once you have a valid string, walk the tree:

```bash
snmpwalk -v2c -c <community> <target_IP>
```

**Findings**

* SNMP leaks a credential pair
* The creds fit the mail services better than SSH

#### 3. Reuse the creds on IMAP

```bash
openssl s_client -connect <target_IP>:imaps
```

Or use an IMAP client with the recovered username and password.

**Findings**

* IMAP login succeeds
* The inbox contains an SSH private key
* The key gives a stronger access path than mail alone

#### 4. Reuse the key over SSH

```bash
chmod 600 id_rsa
ssh -i id_rsa <user>@<target_IP>
```

**Findings**

* SSH access succeeds
* Basic local enumeration does not expose the target password immediately
* `/etc/passwd` reveals a `mysql` user, which points to the next lead

#### 5. Pivot to local account backup data

**Findings**

* The server stores internal account data
* The MySQL path is the key lead after SSH access
* Querying or reading the relevant account data reveals the `HTB` password

### Why this worked

* TCP-only scans missed the real entry point
* SNMP exposed credentials that should never be public
* Mailbox contents exposed an SSH private key
* SSH access allowed local enumeration of stored account data

### Takeaways

* Add UDP scans when TCP results stall
* Treat SNMP as high-value early recon
* Check mailboxes for attached keys, creds, and internal notes
* Reuse every credential across mail, SSH, and database access
