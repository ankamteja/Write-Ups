# SMTP

Use this page to enumerate SMTP, review risky settings, and validate manual interaction.

### Key notes

* **What it is:** A text-based, client-server protocol dedicated to sending, routing, and relaying email traffic across IP networks.
* **Key ports:** `TCP 25`, `TCP 587`, and `TCP 465`.
* **Manual interaction:** `telnet` and `nc` let you inspect banners and test commands directly.
* **Useful commands:** `HELO`, `EHLO`, `VRFY`, `EXPN`, `MAIL FROM`, `RCPT TO`, `DATA`, and `QUIT`.
* **Automation:** `nmap -p25 -sV -sC` captures the banner and supported features. `smtp-open-relay` checks for relay abuse.
* **Dangerous setting:** `mynetworks = 0.0.0.0/0` creates an open relay.

## SMTP

SMTP is a text-based, client-server protocol dedicated to sending, routing, and relaying email traffic across IP networks.

Use it to identify mail software, test user enumeration, and check for open relay issues.

### Core concepts and architecture

#### Ports

* `TCP 25`: Standard plaintext port dedicated to server-to-server mail transfers (MTAs).
* `TCP 587`: Modern secure port used by clients to submit outgoing mail (uses `STARTTLS` to encrypt the session _before_ login credentials are exchanged).
* `TCP 465`: Legacy port used for SMTPS (SMTP wrapped inside an SSL/TLS layer from the start).

#### The email transport ecosystem

* **MUA (Mail User Agent):** The client application (e.g., Thunderbird, Gmail interface) where a user drafts an email.
* **MSA (Mail Submission Agent):** A relay component that accepts incoming mail requests from the MUA and checks the delivery parameters.
* **MTA (Mail Transfer Agent):** The backend server engine (e.g., Postfix, Exim) that queries the DNS to look up target MX records and routes the email across the network.
* **MDA (Mail Delivery Agent):** Receives the message from an incoming MTA and deposits it cleanly into the recipient's storage box to be retrieved via POP3 or IMAP.

### Dangerous configurations

| **Setting**              | **Description**                                                                                                                                                                                                                                                                                                                                                     |
| ------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `mynetworks = 0.0.0.0/0` | **Open Relay**. Tells the mail server to trust connections from every IP address globally. It completely bypasses the login verification (`AUTH PLAIN`) phase. Anyone on the internet can connect anonymously and use the server to blast untraceable spam or highly credible spoofed phishing emails that appear to originate from the organization’s real domain. |

### Manual interaction reference

#### Connect directly to the service

```bash
# Connect directly to the mail server over unencrypted port 25
telnet <TARGET_IP> 25
```

#### Core operational commands

* `HELO <domain>` or `EHLO <domain>` — Initializes the session. `EHLO` signals Extended SMTP, forcing the server to output its supported features (e.g., `STARTTLS`, max attachment size).
* `VRFY <username>` — Directly checks if a specific local user profile or mailbox mailbox exists on the server. Excellent for mapping Active Directory user naming layouts.
* `EXPN <list_name>` — Requests the server to expand a mailing list distribution group to reveal the individual email addresses inside.
* `MAIL FROM:<email>` — Declares the sender of the message.
* `RCPT TO:<email>` — Declares the target recipient.
* `DATA` — Starts the message body constructor. End the stream by hitting enter, typing a single period (`.`), and pressing enter again.
* `QUIT` — Safely closes the current TCP socket connection.

### Footprinting the service

{% code title="Capture the banner, list supported commands, and test relay behavior" %}
```bash
# 1. Capture the SMTP service banner and read allowed commands out of the EHLO list
sudo nmap -p25 -sV -sC <TARGET_IP>

# 2. Test the server for an Open Relay configuration across 16 different delivery variants
sudo nmap -p25 --script smtp-open-relay -v <TARGET_IP>

#To automate scanning the VRFY
smtp-user-enum -M VRFY -U /path/to/wordlist.txt -t <target_ip> -w 15 -v   

#alternatively use
Nmap (smtp-enum-users.nse) and Metasploit (auxiliary/scanner/smtp/smtp_enum) can also be used for this purpose. 
```
{% endcode %}

### Post-enumeration priorities

* If `VRFY` is enabled: Brute-force the command with a wordlist of standard account handles (e.g., `admin`, `it-support`, `hr`, names) to harvest valid corporate usernames. These can be mapped straight to Active Directory targets for credential-spraying on other open protocols.
* If an Open Relay is confirmed: Flag it as a critical vulnerability. This configuration gives an auditor or attacker the structural access required to drop highly authentic phishing payloads directly into corporate mailboxes, bypassing standard internet perimeter security blocks.
* Review Mail Headers: Inspect raw text email headers to find server banners, pathing timestamps, and the specific internal private IPs of the hidden mail relays that handled the data stream.
