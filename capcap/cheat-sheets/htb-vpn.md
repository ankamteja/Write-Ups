# HTB VPN

### Quick start

```bash
sudo openvpn user.ovpn    # connect to VPN
ifconfig                   # check tun0 adapter
netstat -rn               # see accessible networks
```

### Notes

* `tun0` usually confirms the VPN tunnel is up.
* `netstat -rn` shows the routes available through the tunnel.
