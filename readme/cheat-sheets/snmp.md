---
description: Quick SNMP enumeration and community string commands.
---

# SNMP

Use this page as a quick command reference for SNMP checks.

### Read a hostname with `snmpwalk`

```bash
snmpwalk -v 2c -c public 10.129.42.253 1.3.6.1.2.1.1.5.0
```

### Argument reminder

* `-v 2c` uses SNMP v2c.
* `-c public` sets the community string.
* `1.3.6.1.2.1.1.5.0` requests the hostname OID.

### Brute-force community strings

```bash
onesixtyone -c dict.txt 10.129.42.254
```
