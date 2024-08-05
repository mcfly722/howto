# Configure Nekoray client on Raspberry Pi 4

Distrib = Ubuntu 22.04.4 LTS (64-bit)<br>
configure MAC reservation for Raspberry <b>00:00:00:00:00:05=192.168.0.5</b><br>

### import ssh keys
```
sudo ssh-import-id-gh mcfly722
```
login as: anonymous + smartcard
 

### configure network bridge br0
```
sudo tee /etc/netplan/50-cloud-init.yaml << EOF
network:
    ethernets:
        eth0:
            dhcp4: true
            link-local: [ipv4]
            optional: true
        wlan0:
            dhcp4: false
            link-local: [ipv4]
            optional: true
    bridges:
      br0:
        macaddress: 00:00:00:00:00:05
        interfaces: [eth0]
        dhcp4: true
        link-local: [ipv4]
        optional: false
    version: 2
EOF
```


### install lxd/lxc (if not installed)
```
sudo snap install lxd
```

### lxd init
```
lxd init
```
configure to br0 bridge

### create new container
```
sudo lxc launch ubuntu:22.04 nekoray
```

### configure container network & mac for reservation
```
lxc exec nekoray -- netplan set ethernets.eth0.macaddress=00:00:00:00:00:07
lxc exec nekoray -- netplan set ethernets.eth0.link-local=[ipv4]
lxc exec nekoray -- netplan apply
lxc exec nekoray -- netplan status
```