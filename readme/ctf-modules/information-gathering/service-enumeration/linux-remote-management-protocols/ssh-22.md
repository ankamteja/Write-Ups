# SSH (22)

Use this page to enumerate SSH and review common configuration risks.

### SSH (Secure Shell)

Secure Shell (SSH) enables two computers to establish an encrypted, direct connection within a potentially insecure network over standard port TCP `22`.

It prevents third parties from intercepting the data stream and sensitive information.

### Core concepts and architectural protocols

* **SSH-1 vs. SSH-2**
  * **SSH-1** — outdated and highly vulnerable to Man-In-The-Middle (MITM) attacks.
  * **SSH-2** — advanced version offering major structural improvements in encryption metrics, data transfer speed, connection stability, and overall perimeter security.
* **Capabilities** — can be used to securely send terminal commands, transfer complex file structures, or execute local and remote port forwarding tunnels.

### Authentication methods

OpenSSH supports six distinct authentication options:

1. Password authentication
2. Public-key authentication
3. Host-based authentication
4. Keyboard authentication
5. Challenge-response authentication
6. GSSAPI authentication

#### Public-key authentication mechanics

1. **Server validation** — the server transmits its unique public host key to the client to prove its identity. This is vulnerable to interception only during the first initialization connection.
2. **The key pair** — uses an asymmetric cryptographic structure consisting of an immutable public-private key mapping.
   * **Private key (`id_rsa`)** — kept strictly on the client computer and never shared over the wire. It is secured locally via an extended passphrase.
   * **Public key (`id_rsa.pub`)** — stored permanently on the target server.
3. **The challenge-response process** — the server builds a custom cryptographic problem utilizing the client’s public key and forwards it across the session. The client solves it locally using its matching private key and returns the solution to log in without passing a password.

### Configuration baseline analysis

#### `/etc/ssh/sshd_config`

By default, most administrative variables inside `sshd_config` are commented out with a hash mark (`#`) and require manual customization.

A default operational configuration reveals underlying environmental settings:

```
Include /etc/ssh/sshd_config.d/*.conf
ChallengeResponseAuthentication no
UsePAM yes
X11Forwarding yes
PrintMotd no
AcceptEnv LANG LC_*
Subsystem       sftp    /usr/lib/openssh/sftp-server
```

#### Dangerous real-world settings

| **Parameter variable setting** | **Offensive exposure and attack vector**                                                                               |
| ------------------------------ | ---------------------------------------------------------------------------------------------------------------------- |
| `PasswordAuthentication yes`   | Allows password-based logins, exposing user profiles to password mutation guessing and dictionary brute-force strings. |
| `PermitEmptyPasswords yes`     | Permits accounts with blank, unassigned password fields to connect over the network socket.                            |
| `PermitRootLogin yes`          | Grants administrative root access tokens directly over external network lines.                                         |
| `Protocol 1`                   | Forces the daemon to leverage obsolete, MITM-susceptible SSH-1 security baselines.                                     |
| `X11Forwarding yes`            | Permits remote graphical interface application tunnels. Vulnerable to command injection in OpenSSH version `7.2p1`.    |
| `AllowTcpForwarding yes`       | Permits inbound and outbound local and remote network port mapping.                                                    |
| `PermitTunnel`                 | Allows hardware-layer network interface tunneling.                                                                     |
| `DebianBanner yes`             | Forces the application socket to explicitly reveal the underlying distribution family when targeted.                   |

### Footprinting and enumeration playbook

#### Comprehensive auditing via `ssh-audit`

`ssh-audit` analyzes the server configuration to map out banner software, cryptographic parameters, and weak cipher blocks:

```bash
git clone https://github.com/jtesta/ssh-audit.git && cd ssh-audit
./ssh-audit.py <TARGET_IP>
```

> Reading banners: banners indicating `SSH-1.99-OpenSSH_3.9p1` accept both SSH-1 and SSH-2 connections. Banners stating `SSH-2.0-OpenSSH_8.2p1` strictly enforce the modern SSH-2 protocol framework. Old versions may expose flaws like `CVE-2020-14145` (MITM vulnerability).

#### Active debug interrogation

```bash
# Extract accepted authentication strings natively via verbose tracking logs
ssh -v user@<TARGET_IP>

# Force the SSH terminal to target password prompts for brute-force mapping
ssh -v user@<TARGET_IP> -o PreferredAuthentications=password
```
