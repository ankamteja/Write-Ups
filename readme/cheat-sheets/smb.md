---
description: Quick SMB enumeration and share access commands.
---

# SMB

Use this page as a quick command reference for SMB checks.

### Service detection

```bash
nmap -A -p{Port} IP
```

### OS discovery

```bash
nmap --script smb-os-discovery -p445 IP
```

### List shares anonymously

```bash
smbclient -N -L \\IP
```

### Connect to a share

```bash
smbclient \\IP\{non default share}
```

### Connect with credentials

```bash
smbclient -U <username> \\<Target_IP>\<Share_Name>
```

### Common next step

```bash
get passwords.txt
```
