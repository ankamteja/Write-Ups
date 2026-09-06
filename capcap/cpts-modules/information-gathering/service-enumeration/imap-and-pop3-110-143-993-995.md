# IMAP & POP3

Use this page to enumerate IMAP and POP3, validate access, and review risky settings.

### Key notes

* **What they are:** Mail retrieval protocols used by clients to fetch messages from a remote mail server.
* **Core difference:** POP3 is download-and-delete by default. IMAP keeps mail on the server and syncs state across devices.
* **Key ports:** POP3 uses `TCP 110` and `TCP 995`. IMAP uses `TCP 143` and `TCP 993`.
* **Manual interaction:** `openssl s_client` gives direct access to secure POP3 and IMAP sessions.
* **Pre-auth checks:** `CAPABILITY` and `CAPA` reveal supported auth methods and extensions before login.
* **Fast validation:** `curl` can confirm mailbox access quickly when credentials work.
* **Dangerous settings:** `auth_debug_passwords = yes` and related Dovecot options can leak credentials into logs.

## IMAP & POP3

IMAP and POP3 are mail retrieval protocols used by clients to fetch messages sitting on a remote mail server.

Use them to identify mail access methods, validate credentials, inspect mailbox contents, and collect internal intelligence.

### Core concepts and differences

* What they are: Mail retrieval protocols used by clients to fetch messages sitting on a remote mail server.

| **Protocol** | **Core mechanics and sync behavior**                                                                                                                        | **Best for**                                  |
| ------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------- |
| POP3         | Stateless / Download-and-Delete: Downloads all messages onto the local machine and drops them from the remote server by default.                            | Single-device access; minimal server storage. |
| IMAP         | Stateful / Cloud Sync: Keeps messages permanently on the server. Changes (read status, folder trees) synchronize in real-time across all connected devices. | Multi-device access; collaborative mailboxes. |

### Port mapping and traffic encryption

Standard connections pass ASCII commands and user passwords in plaintext.

Secure auditing requires mapping out standard SSL/TLS wrappers.

| **Protocol** | **Default port (plaintext)** | **Secure port (SSL/TLS encrypted)** |
| ------------ | ---------------------------- | ----------------------------------- |
| POP3         | `TCP 110`                    | `TCP 995` (`pop3s`)                 |
| IMAP         | `TCP 143`                    | `TCP 993` (`imaps`)                 |

### Dangerous configurations (`dovecot.conf`)

Loose administrative parameters or debugging flags can leak corporate intelligence straight into system log directories.

* `auth_debug = yes`: Logs step-by-step internal state changes during authentication.
* `auth_debug_passwords = yes`: (Critical Leak) Logs the exact plaintext passwords typed by connecting users into the server log files.
* `auth_verbose_passwords = yes`: Captures failed login details, exposing credentials when users accidentally type passwords in the username field.
* `auth_anonymous_username = guest`: Paired with the `anonymous` SASL mechanism, this lets clients bypass password blocks completely and authenticates them automatically under a specific system profile.

### Interactive tooling reference

#### Passive recon with certificate scraping

Running an Nmap script scan against mail ports extracts the target's underlying SSL/TLS certificates.

{% code title="Enumerate POP3 and IMAP services" %}
```bash
sudo nmap 10.129.14.128 -sV -sC -p110,143,993,995
```
{% endcode %}

Look directly inside the `ssl-cert: Subject:` lines to harvest:

* the Organization Name (`O=`)
* state/location fields
* Admin Email Address (`emailAddress=`)
* the official server FQDN (`CN=`)

#### Prompt-speed credential validation (`curl`)

If you find a password reuse vulnerability (e.g., `robin:robin`), check folder visibility rapidly without an interactive shell.

{% code title="Validate IMAPS access quickly" %}
```bash
# Safely ignore certificate warning (-k) and view primary directories
curl -k 'imaps://<TARGET_IP>' --user <user>:<pass>

#-v for verbose output
```
{% endcode %}

#### Raw secure shell interaction (`openssl`)

Because modern servers block unencrypted credential exchanges, establish your interactive socket directly inside the SSL/TLS tunnel.

{% code title="Connect securely to POP3 and IMAP" %}
```bash
# Connect securely to POP3
openssl s_client -connect <TARGET_IP>:995

# Connect securely to IMAP
openssl s_client -connect <TARGET_IP>:993
```
{% endcode %}

#### Plaintext sockets with `ncat`

If the target exposes plaintext mail services, use `ncat` instead of `telnet and can also use telnet`.

{% code title="Connect to plaintext POP3 and IMAP" %}
```bash
# Connect to plaintext POP3
ncat <TARGET_IP> 110

# Connect to plaintext IMAP
ncat <TARGET_IP> 143
```
{% endcode %}

### Manual command streams

#### Capability checks before authentication

Check server capabilities before sending credentials.

This reveals supported auth methods, `AUTH=ANONYMOUS`, and whether `STARTTLS` is available or required.

**IMAP capability probe**

```
A001 CAPABILITY
```

**POP3 capability probe**

```
CAPA
```

#### POP3 active session

```
USER robin            # Transmit the targeted username account
PASS robin            # Transmit the password
STAT                  # Request mail counts and total byte size
LIST                  # Display mail Index numbers alongside sizes
RETR 1                # Fetch and read the entire body of mail ID 1
DELE 1                # Mark mail ID 1 to be deleted on session close
QUIT                  # Commit changes and disconnect from the server
```

#### IMAP active session

{% hint style="warning" %}
Every command must be prefixed by an arbitrary alphanumeric label identifier (e.g., `A001`, `1`). Empty lines or omitting tags will prompt a `* BAD` response.
{% endhint %}

```
A001 LOGIN robin robin          # Login using raw user credentials
A002 LIST "" *                  # Enumerate all root and sub-folder trees
A003 SELECT INBOX               # Open a folder to query content
A004 FETCH 1 BODY[TEXT]         # Extract the text content from mail ID 1
A004 FETCH 1 all                # Pull headers, metadata, routing details, and body in one response
A005 LOGOUT                     # Safely disconnect from the IMAP shell
```

Use `FETCH 1 all` when `BODY[TEXT]` returns incomplete results or the message contains nested parts.

[https://donsutherland.org/crib/imap](https://donsutherland.org/crib/imap)

#### Parsing IMAP directory flags

* `\Noselect`: The folder is a virtual structural container (like a parent directory). It holds zero emails. Attempting to run `SELECT` on it will return a failure.
* `\HasChildren`: Subfolders exist hidden inside this branch.
* `\HasNoChildren`: A normal end-point mailbox folder. This can be selected and searched.

To query folders inside a virtual folder branch, step down using dot separators:

```
A003 SELECT DEV.DEPARTMENT.INT
```

### Post-enumeration priorities

* Looting Mailboxes: Once inside, focus your data collection on folders like `Sent`, `Drafts`, or internal IT branches (`DEV.DEPARTMENT.INT`). Search message bodies for internal API tokens, server passwords, database connection strings, and legacy configurations left behind by administrators.
* Credential Pivoting: Take any newly discovered corporate credentials or system naming structures and pivot them against Active Directory nodes, SSH endpoints, or internal SMB shares.
