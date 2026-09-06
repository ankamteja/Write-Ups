---
description: Quick local exploit search and copy commands.
---

# Searchsploit

Use this page as a quick command reference for Searchsploit.

Use this when you need to map a service version to public exploit code fast.

### Basic search

```bash
searchsploit openssh 7.2
searchsploit smb ms17-010
searchsploit vsftpd 3.0.3
```

### Title-only output

```bash
searchsploit -t openssh
```

### Exact match search

```bash
searchsploit -e "Windows SMB"
```

### View the exploit file

```bash
searchsploit -x linux/remote/45233.py
```

### Copy the exploit locally

```bash
searchsploit -m linux/remote/45233.py
```

### JSON output

```bash
searchsploit -j openssh 7.2
```

### Map Nmap XML to known exploits

```bash
searchsploit --nmap scan.xml
```

### Common workflow

```bash
nmap -sV -sC -p- <TARGET_IP>
searchsploit <SERVICE> <VERSION>
searchsploit -x <EDB_PATH>
searchsploit -m <EDB_PATH>
```

### Reminder

* Match the service and version first.
* Read the exploit before running it.
* Prefer `check` or safe verification when available.
