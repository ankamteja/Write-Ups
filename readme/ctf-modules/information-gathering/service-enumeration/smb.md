# SMB

### Key notes

* **What SMB is:** A file-sharing protocol for shares, printers, and other network resources.
* OS Target: Primarily Windows, but runs on Linux via Samba.
* **Key ports:** TCP `139` for NetBIOS SMB and TCP `445` for direct SMB/CIFS.
* **Enumeration flow:** Start with `smbclient`, pivot to `rpcclient`, then validate findings with `smbmap`, `crackmapexec`, or `enum4linux-ng`.
* **What each tool gives you:**
  * `smbclient` — lists shares, connects to them, and pulls files.
  * `rpcclient` — queries users, groups, domains, and share details.
  * `smbmap` — shows shares and effective permissions fast.
  * `crackmapexec` — verifies access and highlights SMB security details.
  * `enum4linux-ng` — automates broad SMB, RPC, and NetBIOS enumeration.
* **Dangerous settings:** `guest ok = yes` allows anonymous access, `writable = yes` allows file changes, and `browseable = yes` makes shares easy to discover.

## SMB <a href="#smb" id="smb"></a>

***

SMB exposes shared files, folders, and printers across a network.

Use it to find readable shares, writable paths, usernames, and domain details.

### Ports

* TCP `139` — SMB over NetBIOS
* TCP `445` — direct SMB

### Quick workflow

1. Check whether SMB is exposed.
2. List shares with `smbclient`.
3. Query users and domain info with `rpcclient`.
4. Confirm permissions with `smbmap` or `crackmapexec`.
5. Run `enum4linux-ng` for broader automated coverage.

### Common commands

#### Detect SMB

```bash
nmap -sV -sC -p139,445 <TARGET_IP>
```

#### List shares anonymously

```bash
smbclient -N -L //<TARGET_IP>
```

#### Connect to a share

```bash
smbclient //<TARGET_IP>/<SHARE_NAME>
```

#### Pull useful RPC data

```bash
rpcclient -U "" <TARGET_IP>
```

Useful `rpcclient` queries:

* `srvinfo`
* `enumdomains`
* `querydominfo`
* `netshareenumall`
* `enumdomusers`

#### Check permissions fast

```bash
smbmap -H <TARGET_IP>
```

```bash
crackmapexec smb <TARGET_IP> --shares -u '' -p ''
```

#### Run broad enumeration

```bash
enum4linux-ng -A <TARGET_IP>
```

### What each tool is best for

* `smbclient` — share listing, browsing, and file download.
* `rpcclient` — users, groups, domain info, and share metadata.
* `smbmap` — fast view of read and write access.
* `crackmapexec` — share access plus signing and SMB version checks.
* `enum4linux-ng` — one-pass SMB, RPC, and NetBIOS enumeration.

### High-value findings

* Anonymous access
* Readable shares
* Writable shares
* Usernames and RIDs
* SMB signing not required
* SMBv1 enabled

### Dangerous settings

* `guest ok = yes` — anonymous users can connect.
* `browseable = yes` — shares are easier to discover.
* `writable = yes` — users can change or drop files.
* `read only = no` — write access is allowed.
* `create mask = 0777` — new files may be too permissive.
* `directory mask = 0777` — new folders may be too permissive.

Essential Commands(again):

{% code title="Footprint ports and check if Message Signing is required" %}
```bash
sudo nmap -p 139,445 -sV -sC <IP>
```
{% endcode %}

{% code title="List shares using a Null Session (-N)" %}
```bash
smbclient -N -L //<IP>
```
{% endcode %}

{% code title="Connect directly to an accessible share" %}
```bash
smbclient //<IP>/<share_name>

# Inside smbprompt: use 'ls' to list, 'get <file>' to download, '!' to run local commands
```
{% endcode %}

{% code title="Interact with the MS-RPC backend using rpcclient (Null Session)" %}
```bash
rpcclient -U "" <IP>

# Inside rpcinfo prompt:
#   srvinfo         -> Get server OS and banner information
#   enumdomusers    -> Enumerate all system usernames
#   netshareenumall -> List all available shares
#   netsharegetinfo <share_name> -> Find absolute system paths and permissions
```
{% endcode %}

{% code title="Bash For-Loop to brute-force User RIDs (useful if enumdomusers is restricted)" %}
```bash
for i in $(seq 500 1100); do rpcclient -N -U "" <IP> -c "queryuser 0x$(printf '%x\n' $i)" | grep "User Name\|user_rid" && echo ""; done
```
{% endcode %}

Automated Swiss-Army Tools:

{% code title="Check share permissions instantly" %}
```bash
smbmap -H <IP>
crackmapexec smb <IP> --shares -u '' -p ''
```
{% endcode %}

{% code title="Dump the entire SAM database / users via Impacket" %}
```bash
samrdump.py <IP>
```
{% endcode %}

{% code title="Run a complete, automated scan of names, OS info, policies, users, and shares" %}
```bash
./enum4linux-ng.py <IP> -A
```
{% endcode %}

### What to do next

If you find a readable share, inspect files for creds, configs, keys, and scripts.

If you find a writable share, test whether it can influence an app, login script, or web root.

### `rpcclient` queries

| **Query**                 | **Description**                                                    |
| ------------------------- | ------------------------------------------------------------------ |
| `srvinfo`                 | Server information.                                                |
| `enumdomains`             | Enumerate all domains that are deployed in the network.            |
| `querydominfo`            | Provides domain, server, and user information of deployed domains. |
| `netshareenumall`         | Enumerates all available shares.                                   |
| `netsharegetinfo <share>` | Provides information about a specific share.                       |
| `enumdomusers`            | Enumerates all domain users.                                       |
| `queryuser <RID>`         | Provides information about a specific user.                        |

### Example `rpcclient` session

```bash
rpcclient $> srvinfo

        DEVSMB         Wk Sv PrQ Unx NT SNT DEVSM
        platform_id     :       500
        os version      :       6.1
        server type     :       0x809a03
        
        
rpcclient $> enumdomains

name:[DEVSMB] idx:[0x0]
name:[Builtin] idx:[0x1]


rpcclient $> querydominfo

Domain:         DEVOPS
Server:         DEVSMB
Comment:        DEVSM
Total Users:    2
Total Groups:   0
Total Aliases:  0
Sequence No:    1632361158
Force Logoff:   -1
Domain Server State:    0x1
Server Role:    ROLE_DOMAIN_PDC
Unknown 3:      0x1


rpcclient $> netshareenumall

netname: print$
        remark: Printer Drivers
        path:   C:\var\lib\samba\printers
        password:
netname: home
        remark: INFREIGHT Samba
        path:   C:\home\
        password:
netname: dev
        remark: DEVenv
        path:   C:\home\sambauser\dev\
        password:
netname: notes
        remark: CheckIT
        path:   C:\mnt\notes\
        password:
netname: IPC$
        remark: IPC Service (DEVSM)
        path:   C:\tmp
        password:
        
        
rpcclient $> netsharegetinfo notes

netname: notes
        remark: CheckIT
        path:   C:\mnt\notes\
        password:
        type:   0x0
        perms:  0
        max_uses:       -1
        num_uses:       1
revision: 1
type: 0x8004: SEC_DESC_DACL_PRESENT SEC_DESC_SELF_RELATIVE 
DACL
        ACL     Num ACEs:       1       revision:       2
        ---
        ACE
                type: ACCESS ALLOWED (0) flags: 0x00 
                Specific bits: 0x1ff
                Permissions: 0x101f01ff: Generic all access SYNCHRONIZE_ACCESS WRITE_OWNER_ACCESS WRITE_DAC_ACCESS READ_CONTROL_ACCESS DELETE_ACCESS 
                SID: S-1-1-0
```

### User enumeration

{% code title="User enumeration" %}
```bash
rpcclient $> enumdomusers

user:[mrb3n] rid:[0x3e8]
user:[cry0l1t3] rid:[0x3e9]


rpcclient $> queryuser 0x3e9

        User Name   :   cry0l1t3
        Full Name   :   cry0l1t3
        Home Drive  :   \\devsmb\cry0l1t3
        Dir Drive   :
        Profile Path:   \\devsmb\cry0l1t3\profile
        Logon Script:
        Description :
        Workstations:
        Comment     :
        Remote Dial :
        Logon Time               :      Do, 01 Jan 1970 01:00:00 CET
        Logoff Time              :      Mi, 06 Feb 2036 16:06:39 CET
        Kickoff Time             :      Mi, 06 Feb 2036 16:06:39 CET
        Password last set Time   :      Mi, 22 Sep 2021 17:50:56 CEST
        Password can change Time :      Mi, 22 Sep 2021 17:50:56 CEST
        Password must change Time:      Do, 14 Sep 30828 04:48:05 CEST
        unknown_2[0..31]...
        user_rid :      0x3e9
        group_rid:      0x201
        acb_info :      0x00000014
        fields_present: 0x00ffffff
        logon_divs:     168
        bad_password_count:     0x00000000
        logon_count:    0x00000000
        padding1[0..7]...
        logon_hrs[0..21]...


rpcclient $> queryuser 0x3e8

        User Name   :   mrb3n
        Full Name   :
        Home Drive  :   \\devsmb\mrb3n
        Dir Drive   :
        Profile Path:   \\devsmb\mrb3n\profile
        Logon Script:
        Description :
        Workstations:
        Comment     :
        Remote Dial :
        Logon Time               :      Do, 01 Jan 1970 01:00:00 CET
        Logoff Time              :      Mi, 06 Feb 2036 16:06:39 CET
        Kickoff Time             :      Mi, 06 Feb 2036 16:06:39 CET
        Password last set Time   :      Mi, 22 Sep 2021 17:47:59 CEST
        Password can change Time :      Mi, 22 Sep 2021 17:47:59 CEST
        Password must change Time:      Do, 14 Sep 30828 04:48:05 CEST
        unknown_2[0..31]...
        user_rid :      0x3e8
        group_rid:      0x201
        acb_info :      0x00000010
        fields_present: 0x00ffffff
        logon_divs:     168
        bad_password_count:     0x00000000
        logon_count:    0x00000000
        padding1[0..7]...
        logon_hrs[0..21]...

```
{% endcode %}

We can then use the results to identify the group's RID, which we can then use to retrieve information from the entire group.

### Group information

```bash
  
rpcclient $> querygroup 0x201

        Group Name:     None
        Description:    Ordinary Users
        Group Attribute:7
        Num Members:2

```

### Bruteforcing user IDs

{% code title="Bruteforcing user IDs" %}
```bash
impale7@htb[/htb]$ for i in $(seq 500 1100);do rpcclient -N -U "" 10.129.14.128 -c "queryuser 0x$(printf '%x\n' $i)" | grep "User Name\|user_rid\|group_rid" && echo "";done
```
{% endcode %}

An alternative to this would be a Python script from [Impacket](https://github.com/SecureAuthCorp/impacket) called [samrdump.py](https://github.com/SecureAuthCorp/impacket/blob/master/examples/samrdump.py).
