# MSSQL

Use this page to enumerate MSSQL, validate access, and identify risky settings.

### Key notes

* **What MSSQL is:** A closed-source relational database management system developed by Microsoft.
* **Key port:** TCP `1433`
* **Query language:** T-SQL
* **Main integration point:** Windows authentication and the wider Microsoft ecosystem

### Core concepts and architecture

MSSQL is tightly integrated with the Windows ecosystem and the `.NET` framework.

Unlike MySQL, MSSQL relies on T-SQL and includes deeper operating system integration through system databases and built-in procedural modules.

### Default system databases to know

When you land an initial connection on an MSSQL server, you will usually see these default administrative databases:

* `master` — tracks system-level configuration metadata for the SQL Server instance.
* `model` — acts as the template database for every new database created on the server.
* `msdb` — stores SQL Server Agent schedules, alerts, jobs, and task history.
* `tempdb` — holds temporary objects and transient data created during queries.
* `resource` — a hidden, read-only system database used internally by the SQL engine.

### Dangerous configurations

* **Windows Authentication only mode:** If access relies only on Windows NTLM or Kerberos tokens, a compromised domain session can provide immediate database access.
* **The `sa` account:** The built-in `sa` account has full administrative rights and is often left enabled or protected with weak credentials.
* **Cleartext config stores:** Database credentials often appear in config files, web roots, or saved SSMS profiles.

### Interactive tooling and footprinting

#### Targeted script recon (`nmap`)

Because MSSQL handles NTLM challenge-response natively, targeted Nmap scripts can reveal the internal hostname and domain details without logging in:

{% code title="nmap MSSQL scan" %}
```bash
sudo nmap -sV -p 1433 --script ms-sql-info,ms-sql-ntlm-info,ms-sql-empty-password <TARGET_IP>
```
{% endcode %}

{% hint style="info" %}
Review the `ms-sql-ntlm-info` output for the NetBIOS computer name, DNS domain name, and host OS product version.
{% endhint %}

#### Metasploit passive discovery (`mssql_ping`)

To discover SQL Server instances across an internal network without heavy port scanning, use the UDP-based broadcast ping scanner:

{% code title="Metasploit mssql_ping" %}
```
msf6 > use auxiliary/scanner/mssql/mssql_ping
msf6 auxiliary(scanner/mssql/mssql_ping) > set rhosts <TARGET_IP>
msf6 auxiliary(scanner/mssql/mssql_ping) > run
```
{% endcode %}

#### Linux-to-Windows remote connection (`Impacket`)

When attacking from Linux, use Impacket’s `mssqlclient.py` to handle the Windows authentication handshake cleanly:

{% code title="Connect with mssqlclient.py" %}
```bash
# Connect using Windows Domain Authentication (-windows-auth)
python3 mssqlclient.py Administrator@<TARGET_IP> -windows-auth
```
{% endcode %}

### Active session database navigation (T-SQL)

Once you establish an open connection prompt (`SQL>`), use commands like these to map the environment:

{% code title="MSSQL enumeration queries" %}
```sql
/* 1. Environment Survey */
SELECT @@version;                    -- Returns the precise SQL Server version and build details

/* 2. Listing Existing Databases */
SELECT name FROM sys.databases;      -- Maps out all user and default system database targets

/* 3. Querying Master Tables */
SELECT * FROM sys.tables;            -- Displays tables available inside the active context

/* 4. Reading Column Schemas */
SELECT column_name, data_type FROM information_schema.columns WHERE table_name = 'users';
```
{% endcode %}

### Post-enumeration priorities

* **Look for `xp_cmdshell`:** This extended stored procedure can execute native OS commands. If it is enabled, or if you can enable it with admin rights, it can turn database access into remote code execution.
* **Harvest domain hashes:** Windows-authenticated MSSQL services can sometimes be coerced into authenticating to a listener you control, which may expose crackable NetBIOS or NTLM hashes.
