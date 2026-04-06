## LXD

If you suppose deploy wireguard, you need to deploy it in container, because this service would be act as gateway and all traffic including traffic for other allications on same server would be forwarded via vpn channel.<br>
To to avoid this, we putting wireguard in container and for this we need full fill network not NAT.

### 1. Create br0 and reassing ip to it from eth0
```
sudo apt upgrade
sudo apt install bridge-utils
```
```
sudo tee /etc/systemd/network/10-br0.netdev << EOF
[NetDev]
Name=br0
Kind=bridge
EOF
```
Replace IP, Gateway, DNS with actual values<br>
```
sudo tee /etc/systemd/network/10-br0.network << EOF
[Match]
Name=br0

[Network]
LinkLocalAddressing=no
Address=192.168.0.7/24
Gateway=192.168.0.1
DNS=192.168.0.1
EOF
```
```
sudo tee /etc/systemd/network/10-eth0.network << EOF
[Match]
Name=eth0

[Network]
LinkLocalAddressing=no
Bridge=br0
EOF
```
```
sudo systemctl restart systemd-networkd
```
Enable systemd-networkd service even after restart
```
sudo systemctl enable systemd-networkd
sudo systemctl enable systemd-resolved
sudo systemctl start systemd-networkd
sudo systemctl start systemd-resolved
```
Disable NetworkManager because it conflicting with systemd-networkd
```
sudo systemctl stop NetworkManager
sudo systemctl disable NetworkManager
```
Enable containers autostart
```
sudo mkdir -p /etc/lxc
sudo tee -a /etc/lxc/default.conf << EOF
lxc.start.auto = 1
lxc.start.delay = 5
lxc.group = onboot
EOF
```
### 2. Install LXD
```
sudo apt update && sudo apt upgrade -y
```
```
sudo apt install lxd lxd-client -y
```
```
cat > ~/lxd-init-config.yaml << EOF
config:
  images.auto_update_interval: "0"
networks: []
storage_pools:
- config: {}
  description: ""
  name: default
  driver: dir
profiles:
- config: {}
  description: ""
  devices:
    eth0:
      name: eth0
      nictype: bridged
      parent: br0
      type: nic
    root:
      path: /
      pool: default
      type: disk
  name: default
projects: []
cluster: null
EOF

sudo lxd init --preseed < ~/lxd-init-config.yaml
```
### 3. Enable IPv4 Gateway Forwarding
```
echo "net.ipv4.ip_forward=1"          | sudo tee -a /etc/sysctl.conf
echo "net.ipv4.conf.all.forwarding=1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```
