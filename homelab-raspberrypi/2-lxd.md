#### lxd

If you suppose deploy wireguard, you need to deploy it in container, because this service would be act as gateway and all traffic including traffic for other allications on same server would be forwarded via vpn channel.<br>
To to avoid this, we putting wireguard in container and for this we need full fill network not NAT.

### Create br0 and reassing ip to it from eth0
```
sudo apt upgrade
sudo apt install bridge-utils

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
Address=192.168.0.112/24
Gateway=192.168.0.1
DNS=192.168.0.1
EOF
```
```
sudo tee /etc/systemd/network/10-eth0.network << EOF
[Match]
Name=eth0

[Network]
Bridge=br0
EOF

sudo systemctl restart systemd-networkd
```
