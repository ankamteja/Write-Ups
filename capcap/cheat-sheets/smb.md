---
description: Quick SMB enumeration and share access commands.
---

# SMB

Use this page as a quick command reference for SMB checks.

Use this when you need to detect SMB, list shares, or test access quickly.

### Service detection

```bash
nmap -A -p<TARGET_PORT> <TARGET_IP>
```

### OS discovery

```bash
nmap --script smb-os-discovery -p445 <TARGET_IP>
```

### List shares anonymously

```bash
smbclient -N -L \\\\<TARGET_IP>
```

### Connect to a share

```bash
smbclient \\\\<TARGET_IP>\\<SHARE_NAME>
```

### Connect with credentials

```bash
smbclient -U <USERNAME> \\\\<TARGET_IP>\\<SHARE_NAME>
```

### Common next step

```bash
get passwords.txt
```
