## Deploy Flexisip

### 1. Deploy Flexisip container
```
sudo lxc launch ubuntu:22.04/arm64 flexisip
```
Enter to container shell
```
sudo lxc shell flexisip
```
Configure network interface
```
cat > /etc/netplan/50-cloud-init.yaml << EOF
network:
  version: 2
  ethernets:
    eth0:
      link-local: []
      addresses:
        - 192.168.0.9/24
      nameservers:
        addresses: [192.168.0.1]
      routes:
        - to: 0.0.0.0/0
          via: 192.168.0.1
EOF

sudo netplan apply
```
### 2. Increase file limits
```
sudo lxc config set flexisip limits.kernel.nofile 65535
sudo lxc restart flexisip
```

### 3. Deploy flexisip
Instruction [link](https://github.com/mcfly722/rasp-flexisip)