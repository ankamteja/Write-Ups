# Footprinting - easy

#### Target

* Access provided: `ceil:qwer1234`
* Key services: `21/tcp FTP`, `22/tcp SSH`, `53/tcp DNS`, `2121/tcp ProFTPD`
* Goal: recover `flag.txt`

### Attack chain summary

```
Provided creds
    → Nmap finds FTP, SSH, DNS, and ProFTPD on 2121
    → Test creds on FTP 2121
    → FTP login as ceil succeeds
    → Enumerate files
    → Hidden .ssh directory appears
    → Download id_rsa
    → chmod 600 id_rsa
    → SSH with key as ceil
    → Read flag.txt
```

### Key steps

#### 1. Scan the target

```bash
nmap -sC -sV <target_IP>
```

**Findings**

* `2121/tcp` runs `ProFTPD`
* The provided creds are worth testing on FTP first

#### 2. Log in to FTP on the custom port

```bash
ftp <target_IP> 2121
# Name: ceil
# Password: qwer1234
```

**Findings**

* Login succeeds as `ceil`
* Standard FTP does not give the flag directly

#### 3. Enumerate the FTP files

```bash
ls -la
cd .ssh
ls -la
get id_rsa
```

**Findings**

* FTP exposes a hidden `.ssh` directory
* The directory contains `id_rsa`
* The private key belongs to `ceil`

#### 4. Reuse the key over SSH

```bash
chmod 600 id_rsa
ssh -i id_rsa ceil@<target_IP>
```

**Findings**

* The key gives direct SSH access
* The provided password is only needed to reach the key

#### 5. Read the flag

```bash
find / -name flag.txt 2>/dev/null
cat <path_to_flag>
```

### Why this worked

* The provided creds worked on `ProFTPD`
* FTP exposed sensitive SSH material
* The SSH key turned file access into shell access

### Takeaways

* Test supplied creds on every exposed auth service
* Check hidden files after FTP access
* Treat exposed private keys as instant escalation paths

### Related references

* [FTP (21)](../ftp-21.md)
* [SSH (22)](../linux-remote-management-protocols/ssh-22.md)

