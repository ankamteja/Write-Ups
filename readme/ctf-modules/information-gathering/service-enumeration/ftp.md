# FTP

Use this page to enumerate FTP, test access, and review risky settings.

### Key notes

* `nmap -sC` can reveal anonymous login, directory listing, and version details.
* The version matters when you look for vulnerabilities.
* `nc` and `telnet` can grab banners without an FTP client.
* `openssl` helps with FTPS and also shows certificate details.

### FTP basics

FTP opens two channels:

* a control channel between client and server over TCP `21`
* a data channel over TCP `20`

### Active and passive FTP

In active mode, the server connects back to the client.

This can fail if a firewall blocks the return connection.

In passive mode, the client connects to the server for data transfer.

This avoids many firewall issues.

### TFTP

Trivial File Transfer Protocol is simpler than FTP.

It uses UDP and does not provide user authentication.

Unlike FTP, TFTP does not support directory listings.

### Default `vsftpd` configuration

| **Setting**                                                   | **Description**                                                                                          |
| ------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------- |
| `listen=NO`                                                   | Run from inetd or as a standalone daemon?                                                                |
| `listen_ipv6=YES`                                             | Listen on IPv6 ?                                                                                         |
| `anonymous_enable=NO`                                         | Enable Anonymous access?                                                                                 |
| `local_enable=YES`                                            | Allow local users to login?                                                                              |
| `dirmessage_enable=YES`                                       | Display active directory messages when users go into certain directories?                                |
| `use_localtime=YES`                                           | Use local time?                                                                                          |
| `xferlog_enable=YES`                                          | Activate logging of uploads/downloads?                                                                   |
| `connect_from_port_20=YES`                                    | Connect from port 20?                                                                                    |
| `secure_chroot_dir=/var/run/vsftpd/empty`                     | Name of an empty directory                                                                               |
| `pam_service_name=vsftpd`                                     | This string is the name of the PAM service vsftpd will use.                                              |
| `rsa_cert_file=/etc/ssl/certs/ssl-cert-snakeoil.pem`          | The last three options specify the location of the RSA certificate to use for SSL encrypted connections. |
| `rsa_private_key_file=/etc/ssl/private/ssl-cert-snakeoil.key` |                                                                                                          |
| `ssl_enable=NO`                                               |                                                                                                          |

In addition, `/etc/ftpusers` denies certain users access to the FTP service.

### Dangerous settings

| **Setting**                    | **Description**                                                                    |
| ------------------------------ | ---------------------------------------------------------------------------------- |
| `anonymous_enable=YES`         | Allowing anonymous login?                                                          |
| `anon_upload_enable=YES`       | Allowing anonymous to upload files?                                                |
| `anon_mkdir_write_enable=YES`  | Allowing anonymous to create new directories?                                      |
| `no_anon_password=YES`         | Do not ask anonymous for password?                                                 |
| `anon_root=/home/username/ftp` | Directory for anonymous.                                                           |
| `write_enable=YES`             | Allow the usage of FTP commands: STOR, DELE, RNFR, RNTO, MKD, RMD, APPE, and SITE? |

### Anonymous login

When you connect to a `vsftpd` server, response code `220` usually shows the FTP banner.

That can reveal the service and version.

Example:

```bash
ftp <IP>

Connected to 10.129.14.136.
220 "Welcome to the HTB Academy vsFTP service."
Name (10.129.14.136:cry0l1t3): anonymous

230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
```

Use this to get an overview of server settings:

```bash
ftp> status

//debug and trace can also be used to get more info out of it
```

`debug` and `trace` can also reveal more information.

### More settings

| **Setting**               | **Description**                                                        |
| ------------------------- | ---------------------------------------------------------------------- |
| `dirmessage_enable=YES`   | Show a message when they first enter a new directory?                  |
| `chown_uploads=YES`       | Change ownership of anonymously uploaded files?                        |
| `chown_username=username` | User who is given ownership of anonymously uploaded files.             |
| `local_enable=YES`        | Enable local users to login?                                           |
| `chroot_local_user=YES`   | Place local users into their home directory?                           |
| `chroot_list_enable=YES`  | Use a list of local users that will be placed in their home directory? |

| **Setting**             | **Description**                                                                  |
| ----------------------- | -------------------------------------------------------------------------------- |
| `hide_ids=YES`          | All user and group information in directory listings will be displayed as "ftp". |
| `ls_recurse_enable=YES` | Allows the use of recurse listings                                               |

### `hide_ids=YES`

If `hide_ids=YES` is enabled, listings hide real usernames and groups:

```bash
-rw-rw-r--    1 ftp     ftp      8138592 Sep 14 16:54 Calender.pptx
drwxrwxr-x    2 ftp     ftp         4096 Sep 14 17:03 Clients
drwxrwxr-x    2 ftp     ftp         4096 Sep 14 16:50 Documents
drwxrwxr-x    2 ftp     ftp         4096 Sep 14 16:50 Employees
-rw-rw-r--    1 ftp     ftp           41 Sep 14 16:45 Important Notes.txt
-rw-------    1 ftp     ftp            0 Sep 15 14:57 testupload.txt
```

This hides usernames.

Usernames can help with brute-force attacks against services such as FTP or SSH.

In reality, [fail2ban](https://en.wikipedia.org/wiki/Fail2ban) is now common and can block access after repeated failed login attempts.

### `ls_recurse_enable=YES`

If `ls_recurse_enable=YES` is enabled, recursive listings work:

```bash
ftp> ls -R

---> PORT 10,10,14,4,222,149
200 PORT command successful. Consider using PASV.
---> LIST -R
150 Here comes the directory listing.
.:
-rw-rw-r--    1 ftp      ftp      8138592 Sep 14 16:54 Calender.pptx
drwxrwxr-x    2 ftp      ftp         4096 Sep 14 17:03 Clients
drwxrwxr-x    2 ftp      ftp         4096 Sep 14 16:50 Documents
drwxrwxr-x    2 ftp      ftp         4096 Sep 14 16:50 Employees
-rw-rw-r--    1 ftp      ftp           41 Sep 14 16:45 Important Notes.txt
-rw-------    1 ftp      ftp            0 Sep 15 14:57 testupload.txt

./Clients:
drwx------    2 ftp      ftp          4096 Sep 16 18:04 HackTheBox
drwxrwxrwx    2 ftp      ftp          4096 Sep 16 18:00 Inlanefreight

./Clients/HackTheBox:
-rw-r--r--    1 ftp      ftp         34872 Sep 16 18:04 appointments.xlsx
-rw-r--r--    1 ftp      ftp        498123 Sep 16 18:04 contract.docx
-rw-r--r--    1 ftp      ftp        478237 Sep 16 18:04 contract.pdf
-rw-r--r--    1 ftp      ftp           348 Sep 16 18:04 meetings.txt

./Clients/Inlanefreight:
-rw-r--r--    1 ftp      ftp         14211 Sep 16 18:00 appointments.xlsx
-rw-r--r--    1 ftp      ftp         37882 Sep 16 17:58 contract.docx
-rw-r--r--    1 ftp      ftp            89 Sep 16 17:58 meetings.txt
-rw-r--r--    1 ftp      ftp        483293 Sep 16 17:59 proposal.pptx

./Documents:
-rw-r--r--    1 ftp      ftp         23211 Sep 16 18:05 appointments-template.xlsx
-rw-r--r--    1 ftp      ftp         32521 Sep 16 18:05 contract-template.docx
-rw-r--r--    1 ftp      ftp        453312 Sep 16 18:05 contract-template.pdf

./Employees:
226 Directory send OK.
```

### Downloading files

Download a single file:

```bash
ftp> get <nameofthefile>
```

Download all available files:

```bash
impale7@htb[/htb]$ wget -m --no-passive ftp://anonymous:anonymous@10.129.14.136
```

After the download finishes, `wget` creates a directory named after the target IP and stores every file there.

Downloading files is one of the main FTP features, along with uploading files we create.

This can matter for exploitation.

For example, uploaded files may be used with LFI to make the host execute system commands.

FTP logs can also become an attack path and lead to `Remote Command Execution` (`RCE`).

This applies to the FTP service and to other services discovered during enumeration.

### Uploading files

Next, check whether you have permission to upload files to the FTP server.

This matters especially with web servers, where files are often synchronized for developer access.

FTP is commonly used for that workflow.

Many configuration errors appear on servers administrators think are not discoverable.

When internal systems are assumed unreachable, hardening often gets neglected.

That leads to misconfigurations.

If uploads reach a web server, the chance of direct access increases.

That can lead to a web shell, a reverse shell, internal command execution, and possibly privilege escalation.

```bash
ftp> put <nameofthefile>
```

### Footprinting the service

#### Using Nmap

```bash
impale7@htb[/htb]$ sudo nmap -sV -p21 -sC -A 10.129.14.136

Starting Nmap 7.80 ( https://nmap.org ) at 2021-09-16 18:12 CEST
Nmap scan report for 10.129.14.136
Host is up (0.00013s latency).

PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 2.0.8 or later
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
| -rwxrwxrwx    1 ftp      ftp       8138592 Sep 16 17:24 Calendar.pptx [NSE: writeable]
| drwxrwxrwx    4 ftp      ftp          4096 Sep 16 17:57 Clients [NSE: writeable]
| drwxrwxrwx    2 ftp      ftp          4096 Sep 16 18:05 Documents [NSE: writeable]
| drwxrwxrwx    2 ftp      ftp          4096 Sep 16 17:24 Employees [NSE: writeable]
| -rwxrwxrwx    1 ftp      ftp            41 Sep 16 17:24 Important Notes.txt [NSE: writeable]
|_-rwxrwxrwx    1 ftp      ftp             0 Sep 15 14:57 testupload.txt [NSE: writeable]
| ftp-syst:
|   STAT:
| FTP server status:
|      Connected to 10.10.14.4
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 2
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
```

The [ftp-anon](https://nmap.org/nsedoc/scripts/ftp-anon.html) NSE script checks whether the FTP server allows anonymous access.

The `ftp-syst` script runs the `STAT` command and shows FTP server status details.

#### Banner grab by interacting

```bash
impale7@htb[/htb]$ nc -nv 10.129.14.136 21
```

```bash
impale7@htb[/htb]$ telnet 10.129.14.136 21
```

#### FTPS with `openssl`

If the FTP server runs with TLS or SSL, use `openssl` to communicate with it.

This also reveals certificate details.

```bash
        shellsession
impale7@htb[/htb]$ openssl s_client -connect 10.129.14.136:21 -starttls ftp

CONNECTED(00000003)
Can't use SSL_get_servername
depth=0 C = US, ST = California, L = Sacramento, O = Inlanefreight, OU = Dev, CN = master.inlanefreight.htb, emailAddress = admin@inlanefreight.htb
verify error:num=18:self signed certificate
verify return:1

depth=0 C = US, ST = California, L = Sacramento, O = Inlanefreight, OU = Dev, CN = master.inlanefreight.htb, emailAddress = admin@inlanefreight.htb
verify return:1
---
Certificate chain
 0 s:C = US, ST = California, L = Sacramento, O = Inlanefreight, OU = Dev, CN = master.inlanefreight.htb, emailAddress = admin@inlanefreight.htb

 i:C = US, ST = California, L = Sacramento, O = Inlanefreight, OU = Dev, CN = master.inlanefreight.htb, emailAddress = admin@inlanefreight.htb
---

Server certificate

-----BEGIN CERTIFICATE-----

MIIENTCCAx2gAwIBAgIUD+SlFZAWzX5yLs2q3ZcfdsRQqMYwDQYJKoZIhvcNAQEL
...SNIP...
```

The SSL certificate can help identify the hostname and useful email addresses.
