## Deploy Tunelled Gateway

### 1. Deploy gateway container
```
sudo lxc launch ubuntu:22.04/arm64 gateway
```
Enter to container shell
```
sudo lxc shell gateway
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
        - 192.168.0.8/24
      nameservers:
        addresses: [192.168.0.1]
      routes:
        - to: 0.0.0.0/0
          via: 192.168.0.1
EOF

sudo netplan apply
```
### 2. Enable universe repository (required for wireguard)
```
sudo apt update
sudo apt install -y software-properties-common
sudo add-apt-repository universe
sudo apt update
```
### 3. Confingure Wireguard and Cloak gateway
[Instruction](https://mcfly722.github.io/cloak-vpn-helper/)