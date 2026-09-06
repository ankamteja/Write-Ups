---
description: Medium lab attack path for recovering the `HTB` user password.
---

# Footprinting - medium

Use this page to track the medium footprinting lab path.

### Target

* OS: `Windows`
* Difficulty: `Medium`
* Goal: recover the password for user `HTB`
* Initial clue: internal server with broad user access

### Attack chain summary

```
Nmap scan
    → Windows host with RPC, SMB, NFS, RDP, and WinRM
    → showmount reveals /TechSupport
    → Mount NFS export
    → Read support files
    → Recover alex:l0l123!mD
    → RDP as alex
    → Find "important" file
    → Reuse recovered admin-style creds over RDP
    → Access SQL Server as admin
    → Read HTB user record
    → Recover HTB password
```

### Key steps

#### 1. Scan the target

```bash
nmap -sV <target_IP>
nmap -sC -sV <target_IP>
```

**Findings**

* The host is Windows
* Useful services include `111`, `135`, `139`, `445`, `2049`, `3389`, and `5985`
* `2049` points to NFS and becomes the best next lead

#### 2. Enumerate NFS

```bash
showmount -e <target_IP>
```

**Findings**

* The target exports `/TechSupport`
* The share is accessible to everyone

Mount it locally:

```bash
mkdir NFS
sudo mount -t nfs <target_IP>:/TechSupport ./NFS -o nolock
```

#### 3. Read the exported files

```bash
cd NFS
find . -type f -maxdepth 2
cat <interesting_file>
```

**Findings**

* Most files are empty
* One file contains credentials for `alex`
* Recovered creds: `alex:l0l123!mD`

#### 4. Use the creds over RDP

```bash
xfreerdp /v:<target_IP> /u:alex /p:'l0l123!mD'
```

**Findings**

* RDP access works as `alex`
* The desktop exposes more internal data
* A file named `important` contains another credential lead

#### 5. Reuse the second credential set

**Findings**

* The `important` file suggests admin-level access
* The creds do not work in SQL Server with `sa`
* The same value works for a higher-privileged RDP login

#### 6. Read the database and recover the target password

**Findings**

* The higher-privileged session gives access to SQL Server data
* The `HTB` user record is present in the database
* The relevant row uses ID `157`
* Copy the password value for the final answer

### Why this worked

* NFS exposed internal support data to all users
* Support files leaked reusable credentials
* RDP access exposed a second credential source
* Admin desktop access led to the database that stored the `HTB` password

### Takeaways

* Do not ignore NFS on mixed Windows environments
* Public support shares often leak credentials
* Reuse every credential across RDP, SMB, WinRM, and database access
* Desktop artifacts can be as valuable as service banners

<br>
