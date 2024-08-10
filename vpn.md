# 1. Configuring foreign VM
<details><h2><summary>1.1 Configuring Ubuntu 22.04 on external VM</summary></h2>

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

</details>

<details><h2><summary> 1.2 Configuring WireGuard Server service</summary></h2>

```
# 1.2.1 Install package
apt install -y wireguard

# 1.2.2 Generate new wireguard private+public key
export PrivateKey=$(wg genkey)
export PublicKey=$(echo "$PrivateKey" | wg pubkey)

echo "$PublicKey" > /etc/wireguard/publickey
echo "$PrivateKey" > /etc/wireguard/privatekey

# 1.2.3 Set private key access rights
chmod 600 /etc/wireguard/privatekey

# 1.2.4 Create new wireguard config with new private key (!!! ensure that your interface is ens3, otherwise change it to eth0 !!!)
sudo tee /etc/wireguard/wg0.conf << EOF
[Interface]
PrivateKey = $PrivateKey
Address = 10.0.0.1/8
ListenPort = 51820
PostUp = iptables -A FORWARD -i %i -j ACCEPT  ; iptables -t nat -A POSTROUTING -o ens3 -j MASQUERADE
PostDown = iptables -D FORWARD -i %i -j ACCEPT; iptables -t nat -D POSTROUTING -o ens3 -j MASQUERADE
EOF


# 1.2.5 Enable ip forwarding
echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf
sysctl -p


# 1.2.6 Enable and start systemd service
systemctl enable wg-quick@wg0.service
systemctl start wg-quick@wg0.service

# 1.2.8 Check current status
systemctl status wg-quick@wg0.service
```
</details>

<details><h2><summary>1.3 Generating new client config for RaspberryPi GW</summary></h2>

```
# 1.3.1 generate new public/private keys pair
export CLOAK_CLIENT_LISTENING_ADDRESS=192.168.0.6
export    CLOAK_CLIENT_LISTENING_PORT=51820
export            WG_ClientPrivateKey=$(wg genkey)
export             WG_ClientPublicKey=$(echo "$WG_ClientPrivateKey" | wg pubkey)
export              WG_ClientAlowedIp="10.$((3 + $RANDOM % 250)).$((3 + $RANDOM % 250)).$((3 + $RANDOM % 250))"
export                   WG_ClientDNS="8.8.8.8"

# 1.3.2 append public key to config
sudo tee -a /etc/wireguard/wg0.conf << EOF
[Peer]
PublicKey = $WG_ClientPublicKey
AllowedIPs = $WG_ClientAlowedIp/32
EOF

echo "[Interface]
PrivateKey = $WG_ClientPrivateKey
Address = $WG_ClientAlowedIp/32
DNS = $WG_ClientDNS

[Peer]
PublicKey = $WG_ClientPublicKey
Endpoint = $CLOAK_CLIENT_LISTENING_ADDRESS:$CLOAK_CLIENT_LISTENING_PORT
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 20
" > wireguard_client.conf


# to view config
cat wireguard_client.conf

```
</details>

<details><h2><summary>1.3 Configuring Cloak service</summary></h2>

https://habr.com/ru/articles/758570/
https://github.com/cbeuw/Cloak

```
# 1.3.1 Install Cloak
wget https://github.com/cbeuw/Cloak/releases/download/v2.7.0/ck-server-linux-amd64-v2.7.0 -O ck-server
chmod +x ck-server
sudo mv ck-server /usr/bin/ck-server

# 1.3.2 create cloak key-pair, save this keys
/usr/bin/ck-server -key

Your PUBLIC key is:                      # <PublicKey>
Your PRIVATE key is (keep it secret):    # <PrivateKey>


# 1.3.3 generate uid for user and admin, save this keys
/usr/bin/ck-server -uid
/usr/bin/ck-server -uid

Your UID is: <..................> #  <BypassUID>
Your UID is: <..................> #  <AdminUID>

# 1.3.4 create cloak configuration
sudo mkdir -p /etc/cloak
sudo tee /etc/cloak/ckserver.json << EOF
{
  "ProxyBook": {
    "shadowsocks": [
      "udp",
      "127.0.0.1:51820"
    ]
  },
  "BindAddr": [
    ":443"
  ],
  "BypassUID": [
    "<BypassUID>"
  ],
  "RedirAddr": "nl.mirror.flokinet.net",
  "PrivateKey": "<PrivateKey>",
  "AdminUID": "<AdminUID>",
  "DatabasePath": "userinfo.db"
}
EOF

# 1.3.5 register cloak service
sudo tee /etc/systemd/system/cloak-server.service << EOF
[Unit]
Description=cloak-server
After=network.target
StartLimitIntervalSec=0
[Service]
Type=simple
ExecStart=/usr/bin/ck-server -c /etc/cloak/ckserver.json
Restart=always
[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable cloak-server.service
sudo systemctl start cloak-server.service
sudo systemctl status cloak-server.service

# 1.3.6 allow TCP 443 for incomming connections
sudo ufw allow 443
```
</details>

# 2. Configuring Raspberry PI VPN Gateway
Install <b>Raspberry Pi OS Lite (64-bit)</b><br>

<details><h2><summary>2.2 Configure Cloak client</summary></h2>

```
# 2.2.1 Install Cloak client
curl -L https://github.com/cbeuw/Cloak/releases/download/v2.7.0/ck-client-linux-arm64-v2.7.0 > ck-client
chmod +x ck-client
sudo mv ck-client /usr/bin/ck-client

# 2.2.2 Cloak Config
sudo mkdir -p /etc/config/cloak/
sudo tee /etc/config/cloak/ckclient.json << EOF
{
"Transport": "direct",
"ProxyMethod": "openvpn",
"EncryptionMethod": "aes-gcm",
"UID": "<BypassUID>",
"PublicKey": "<PublicKey>",
"ServerName": "nl.mirror.flokinet.net",
"NumConn": 20,
"BrowserSig": "chrome",
"StreamTimeout": 300
}
EOF

# 2.2.3 Register Cloak service
export CLOAK_CLIENT_LISTENING_ADDRESS=0.0.0.0
export CLOAK_CLIENT_LISTENING_PORT=51820
export CLOAK_REMOTE_PORT=443
export CLOAK_REMOTE_SERVER="<enter your VDS IP>"

# 2.3.4 Create Cloak service config
sudo tee /lib/systemd/system/cloak.service << EOF
[Unit]
Description=Cloak Client Service
After=network-online.target

[Service]
ExecStart=/usr/bin/ck-client -s "$CLOAK_REMOTE_SERVER" -p "$CLOAK_REMOTE_PORT" -i "$CLOAK_CLIENT_LISTENING_ADDRESS" -l "$CLOAK_CLIENT_LISTENING_PORT" -u -c /etc/config/cloak/ckclient.json
WorkingDirectory=/tmp
StandardOutput=inherit
StandardError=inherit
Restart=always
User=root

[Install]
WantedBy=multi-user.target
EOF

# start Cloak client service
sudo systemctl start cloak.service
```

```
journalctl -u cloak.service
```
</details>






# 4. Debug & Performance testing

### debug cloak client
journalctl -u cloak.service -f
### debug cloak server
journalctl -u cloak-server.service -f



### check server speed
```
snap install speedtest-cli

speedtest-cli
```





## 1.2 Configuring OpenVPN service
https://github.com/angristan/openvpn-install
https://firstvds.ru/technology/ustanovka-openvpn
### 1.2.1 install OpenVPN using script
```
curl -O https://raw.githubusercontent.com/angristan/openvpn-install/master/openvpn-install.sh
chmod +x openvpn-install.sh
./openvpn-install.sh
```
Script answers:
```
IP address: 127.0.0.1
Do you want to enable IPv6 support (NAT)? [y/n]: n
What port do you want OpenVPN to listen to? 2) custom= 62162
Protocol [1-2]: 1 (UDP)
What DNS resolvers do you want to use with the VPN? 1) Current system resolvers
Enable compression? [y/n]: n
Customize encryption settings? [y/n]: n
```
