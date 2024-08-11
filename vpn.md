# links:

- https://habr.com/ru/articles/758570/
- https://github.com/cbeuw/Cloak
- https://github.com/angristan/openvpn-install
- https://firstvds.ru/technology/ustanovka-openvpn
- https://github.com/n-r-w/shadow-client
- https://github.com/n-r-w/shadow-server

# 1. Remote foreing VM
## 1.1 Configuring Ubuntu 22.04
```
# 1.1.1 enable ufw and ssh access
sudo ufw allow ssh
sudo ufw enable

# 1.1.2 import public key for ssh access for my github account (mcfly722)
sudo ssh-import-id-gh mcfly722

# 1.1.3 disable sshd password authentication
sudo tee -a /etc/ssh/sshd_config << EOF
PasswordAuthentication no
EOF

# 1.1.4 restart sshd to apply authentication changes
sudo systemctl restart ssh
sudo service sshd restart
```

### 1.1.1 Install Cloak server
```
wget https://github.com/cbeuw/Cloak/releases/download/v2.7.0/ck-server-linux-amd64-v2.7.0 -O ck-server
chmod +x ck-server
sudo mv ck-server /usr/bin/ck-server
```
### 1.1.2 create cloak key-pair
```
export content=$(/usr/bin/ck-server -key)
export ck_publicKey=$(echo "$content" | head -n1 | awk '{print $NF}')
export ck_privateKey=$(echo "$content" | head -n2 | tail -1 | awk '{print $NF}')
export ck_uid=$(/usr/bin/ck-server -uid | awk '{print $NF}')
```
### 1.1.3 create cloak server config
```
sudo mkdir -p /etc/cloak
sudo tee /etc/cloak/cloak-server.json << EOF
{
  "ProxyBook": {
    "wireguard": [
      "udp",
      "127.0.0.1:51820"
    ]
  },
  "BindAddr": [
    ":443"
  ],
  "BypassUID": [
    "$ck_uid"
  ],
  "RedirAddr": "nl.mirror.flokinet.net",
  "PrivateKey": "$ck_privateKey"
}
EOF
```
### 1.1.4 create cloak client config
```
sudo mkdir -p /etc/cloak
sudo tee /etc/cloak/cloak-client.json << EOF
{
  "Transport": "direct",
  "ProxyMethod": "wireguard",
  "EncryptionMethod": "aes-128-gcm",
  "UID": "$ck_uid",
  "PublicKey": "$ck_publicKey",
  "ServerName": "nl.mirror.flokinet.net",
  "NumConn": 30,
  "KeepAlive": 0,
  "BrowserSig": "chrome",
  "StreamTimeout": 300
}
EOF
```
### 1.1.5 register Cloak server service
```
sudo tee /etc/systemd/system/cloak-server.service << EOF
[Unit]
Description=cloak-server
After=network.target
StartLimitIntervalSec=0
[Service]
Type=simple
ExecStart=/usr/bin/ck-server -c /etc/cloak/cloak-server.json
Restart=always
[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable cloak-server.service
sudo systemctl start cloak-server.service
sudo systemctl status cloak-server.service

sudo journalctl -u cloak-server.service -f
```
### 1.1.6 allow Cloak server TCP 443 for incomming connections
```
sudo ufw allow 443
```

## 1.2 Install WireGuard Server
```
sudo apt install -y wireguard openresolv iptables

# enable gateway forwarding
echo "net.ipv4.ip_forward=1"          | sudo tee -a /etc/sysctl.conf
echo "net.ipv4.conf.all.forwarding=1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```
### 1.2.1 Generate Wireguard Keys
```
export WG_ServerPrivateKey=$(wg genkey)
export  WG_ServerPublicKey=$(echo "$WG_ServerPrivateKey" | wg pubkey)
export WG_ClientPrivateKey=$(wg genkey)
export  WG_ClientPublicKey=$(echo "$WG_ClientPrivateKey" | wg pubkey)
```
### 1.2.2 Create Wireguard Server config
```
export default_interface=$(ip route | awk '/default/ {print $5; exit}')


sudo tee /etc/wireguard/wg0.conf << EOF
[Interface]
PrivateKey = $WG_ServerPrivateKey
Address = 10.1.1.1/24
ListenPort = 51820

PostUp = iptables -I INPUT -p udp --dport 51820 -j ACCEPT
PostUp = iptables -I FORWARD -i $default_interface -o wg0 -j ACCEPT
PostUp = iptables -I FORWARD -i wg0 -j ACCEPT
PostUp = iptables -t nat -A POSTROUTING -o $default_interface -j MASQUERADE
PostDown = iptables -D INPUT -p udp --dport 51820 -j ACCEPT
PostDown = iptables -D FORWARD -i $default_interface -o wg0 -j ACCEPT
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT
PostDown = iptables -t nat -D POSTROUTING -o $default_interface -j MASQUERADE

[Peer]
PublicKey = $WG_ClientPublicKey
AllowedIPs = 10.1.1.2/32
EOF
```
### 1.2.3 Create Wireguard Client config
```
export CLOAK_CLIENT_LISTENING_ADDRESS=<ADD HERE YOUR RASPBERRYPI LAN IP>

sudo tee /etc/wireguard/client-wg0.conf << EOF
[Interface]
PrivateKey = $WG_ClientPrivateKey
Address = 10.1.1.2/32
MTU = 1420

PostUp = iptables -t nat -A POSTROUTING -o wg0 -j MASQUERADE
PostDown = iptables -t nat -D POSTROUTING -o wg0 -j MASQUERADE

[Peer]
PublicKey = $WG_ServerPublicKey
Endpoint = $CLOAK_CLIENT_LISTENING_ADDRESS:1984
AllowedIPs = 0.0.0.0/0
EOF
```

### 1.2.4 Enable and start Wireguard server service
```
sudo systemctl enable wg-quick@wg0.service
sudo systemctl restart wg-quick@wg0.service
sudo systemctl status wg-quick@wg0.service
sudo wg
sudo journalctl -u wg-quick@wg0.service -f
```


# 2. LAN Gateway
## 2.1 Install Raspberry Pi OS Lite (64-bit)
```
# enable forwarding
echo "net.ipv4.ip_forward=1"          | sudo tee -a /etc/sysctl.conf
echo "net.ipv4.conf.all.forwarding=1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```
### 2.2.1 Specify static route to your external VM via local gateway
Otherwise, all Cloak client traffic would be routed back to WireGuard and would be loop.
```
sudo ip route add <YOUR EXTERNAL VM IP>/32 via <YOUR LOCAL GATEWAY>
```

### 2.2.2 Install Cloak client
```
curl -L https://github.com/cbeuw/Cloak/releases/download/v2.7.0/ck-client-linux-arm64-v2.7.0 > ck-client
chmod +x ck-client
sudo mv ck-client /usr/bin/ck-client
sudo mkdir -p /etc/config/cloak
```
### 2.2.3 Copy cloak client config from server to client
```
from server: /etc/cloak/cloak-client.json
to client:   /etc/cloak/cloak-client.json
```

### 2.2.4 Register Cloak service
```
export CLOAK_REMOTE_SERVER="<enter your Remote VM IP>"

sudo tee /lib/systemd/system/cloak-client.service << EOF
[Unit]
Description=Cloak Client Service
After=network-online.target

[Service]
ExecStart=/usr/bin/ck-client -s $CLOAK_REMOTE_SERVER -p 443 -i 127.0.0.1 -u -c /etc/cloak/cloak-client.json
WorkingDirectory=/tmp
StandardOutput=inherit
StandardError=inherit
Restart=always
User=root

[Install]
WantedBy=multi-user.target
EOF
```
### 2.2.5 start Cloak client service
```
sudo systemctl enable cloak-client.service
sudo systemctl restart cloak-client.service
sudo systemctl status cloak-client.service
sudo journalctl -u cloak-client.service -f
```


### 2.3.1 Install Wireguard Client
```
sudo apt install -y wireguard openresolv iptables
```
### 2.3.2 Copy Wireguard client config from server to client and rename it
```
from server: /etc/wireguard/client-wg0.conf
to client:   /etc/wireguard/wg0.conf
```
### 2.3.3 Enable and start Wireguard client service
```
sudo systemctl enable wg-quick@wg0.service
sudo systemctl restart wg-quick@wg0.service
sudo systemctl status wg-quick@wg0.service
sudo wg
sudo journalctl -u wg-quick@wg0.service -f
```
---
# 3. Performance testing
## 3.1 check external ip
```
curl checkip.amazonaws.com
```
## 3.2 check server speed
```
snap install speedtest-cli
speedtest-cli
```
## 3.3 check mtu
```
ping -c1 -s <SPECIFY YOUR TESTING PACKET SIZE> -M do 10.1.1.1
# you chould add 28 to your maximum transmitted packet size, and it would be correct mtu
```
## 3.4 check tcp speed
```
# on destination
iperf -s -p 5003

# on source
iperf -c <REMOTE IP ADDRESS> -p 5003 -P 5
```

# 4. Debug
## 4.1 for Remote VM
```
ss -nltu 'sport = 443'
ss -nltu 'sport = 51820'
sudo tcpdump -nei ens3 tcp port 443
sudo tcpdump -nei ens3 udp port 51820
```
# 4.2 for raspberry pi
```
# check that UDP1984 is listening by Cloak client
ss -nltu 'sport = 1984'

# check access to your external vm via 443 port.
sudo tcpdump -i any -nn src host <YOUR EXTERNAL VM IP> and port 443
# otherwise add static route:
sudo ip route add <YOUR EXTERNAL VM IP>/32 via 192.168.0.1

sudo systemctl restart wg-quick@wg0.service

```