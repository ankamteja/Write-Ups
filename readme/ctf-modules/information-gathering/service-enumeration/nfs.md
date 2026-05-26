# NFS

* Ports — 111 and 2049
* Commands — showmount, mount, ls -n, umount
* Dangerous settings — no\_root\_squash, rw, insecure
* Key concept — NFS trusts UID, so matching UID = file access\
  \
  `Network File System` (`NFS`) is a network file system developed by Sun Microsystems and has the same purpose as SMB. Its purpose is to access file systems over a network as if they were local.
* OS Target: Primarily used between Linux/Unix systems.
* Ports: 111 (rpcbind/portmapper) and 2049 (NFS).
* The Flaw: NFSv2/v3 lacks native user authentication; it trusts the client machine’s UID/GID. If you change your local UID to match the target file owner, you get access.

### Dangerous Settings <a href="#dangerous-settings" id="dangerous-settings"></a>

| **Option**       | **Description**                                                                                                      |
| ---------------- | -------------------------------------------------------------------------------------------------------------------- |
| `rw`             | Read and write permissions.                                                                                          |
| `insecure`       | Ports above 1024 will be used.                                                                                       |
| `nohide`         | If another file system was mounted below an exported directory, this directory is exported by its own exports entry. |
| `no_root_squash` | All files created by root are kept with the UID/GID 0.                                                               |

Footprinting the Service:

{% code title="Footprint the service and check active RPC mappings" %}
```bash
sudo nmap 10.129.14.128 -p111,2049 -sV -sC
sudo nmap --script nfs* -p 111,2049 <IP>
```
{% endcode %}

{% code title="Show available NFS shares" %}
```bash
showmount -e 10.129.14.128
```
{% endcode %}

{% code title="Mount the target share to a local folder (-o nolock bypasses file locking)" %}
```bash
mkdir target-NFS
sudo mount -t nfs <IP>:/source/path ./target-NFS/ -o nolock
```
{% endcode %}

{% code title="Inspect raw numeric User IDs inside the mounted share" %}
```bash
ls -ln ./target-NFS/
```
{% endcode %}

{% code title="Clean up / Unmount the share" %}
```bash
sudo umount ./target-NFS
```
{% endcode %}
