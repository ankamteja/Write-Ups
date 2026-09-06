# MySQL

Use this page to enumerate MySQL, validate access, and identify risky settings.

### Key notes

* **What MySQL is:** An open-source, client-server relational database management system that stores data in tables, columns, rows, and data types.
* **Key port:** TCP `3306`
* **Architecture:** `mysqld` handles structured queries from local or remote clients.
* **Common stacks:** LAMP and LEMP

### Core concepts and architecture

MySQL is frequently deployed as part of the LAMP (Linux, Apache, MySQL, PHP) or LEMP (with Nginx) stack.

The service usually runs through `mysqld`, which processes local and remote database connections.

### Dangerous configurations (`mysqld.cnf`)

When auditing `/etc/mysql/mysql.conf.d/mysqld.cnf`, inspect these parameters closely for visibility and security gaps:

* `bind-address = 0.0.0.0` — forces the database engine to listen on every available network interface and exposes an internal service to remote scanning.
* `secure_file_priv = ""` — if empty or misconfigured, it removes import and export restrictions and may let an attacker read local files or write a web shell through SQL injection.
* `debug` or `sql_warnings = 1` — enables verbose error handling and can leak deep database metadata into public-facing applications.

### Service footprinting and verification

#### Automated script recon (`nmap`)

Run a script scan to extract software versions, security policies, and initial authentication details:

{% code title="nmap MySQL scan" %}
```bash
sudo nmap -sV -sC -p3306 --script mysql* <TARGET_IP>
```
{% endcode %}

{% hint style="warning" %}
Nmap `mysql-brute` and `mysql-empty-password` can report false positives for `root` with an empty password. Always confirm access manually.
{% endhint %}

#### Local or remote manual authentication

Connect with discovered credentials such as `root:P4SSw0rd`.

Ensure there is no space between `-p` and the password.

{% code title="Connect to MySQL" %}
```bash
# Connect to a remote target instance securely
mysql -u root -pP4SSw0rd -h <TARGET_IP>
```
{% endcode %}

### Active session database navigation

Once inside the MySQL prompt (`MySQL [(none)]>`), use this layout to map and dump stored table data:

{% code title="MySQL enumeration queries" %}
```sql
/* 1. Environment & Server Recon */
select version();                   -- Drops the exact underlying software version
SHOW VARIABLES LIKE 'secure_file_priv'; -- Checks file system read/write privilege restrictions

/* 2. Mapping the Data Environment */
show databases;                      -- Lists all available target databases
use mysql;                           -- Switches the session view to focus on the 'mysql' database

/* 3. Extracting Structural Target Coordinates */
show tables;                         -- Lists all available tables inside the chosen database
show columns from user;              -- Reveals column structures (fields) inside the 'user' table

/* 4. Looting the Target Data */
select user, host, authentication_string from user; -- Dumps user credentials and password hashes
select * from host_summary;          -- Captures comprehensive connection statistics

/* 5. Filtering for Precise String Data */
select * from profiles where username = "admin";    -- Pulls specific administrative profiles
```
{% endcode %}

### Built-in metadata schemas to memorize

Every standard MySQL setup includes metadata schemas that help locate high-value developer-created tables:

* `information_schema` — the system metadata map. Use it to list table names and column names across every database without knowing them first.
* `sys` — the internal configuration catalog with performance views, host summaries, and diagnostic metrics.

### Post-enumeration priorities

* **Extract and crack hashes:** Dump the `authentication_string` column from `mysql.user`, then test recovered hashes with tools such as Hashcat or John the Ripper.
* **Check OS file read and write capability:** If `secure_file_priv` is disabled, test operations such as `LOAD_DATA()` or `SELECT ... INTO OUTFILE` to read files like `/etc/passwd` or write payloads to disk.
