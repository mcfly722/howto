# Configuring Wireguard

### 1. Install
```
apt install -y wireguard
```
### 2. Generate new wireguard private+public key
```
wg genkey | tee /etc/wireguard/privatekey | wg pubkey | tee /etc/wireguard/publickey
```
### 3. Set keys access rights
```
chmod 600 /etc/wireguard/privatekey
```
### 4. Create new wireguard config with new private key
```
cat > /etc/wireguard/wg0.conf << EOL
[Interface]
PrivateKey = <INSERT YOUR PRIVATE KEY HERE FROM /etc/wireguard/privatekey>
Address = 10.0.0.1/24
ListenPort = 27010
PostUp = iptables -A FORWARD -i %i -j ACCEPT  ; iptables -t nat -A POSTROUTING -o ens3 -j MASQUERADE
PostDown = iptables -D FORWARD -i %i -j ACCEPT; iptables -t nat -D POSTROUTING -o ens3 -j MASQUERADE
EOL
```
ensure that your interface is ens3, otherwise change it to eth0
### 5. Enable ip forwarding
```
echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf
```
```
sysctl -p
```
### 6. Create systemd service
```
systemctl enable wg-quick@wg0.service
```
### 7. Start new service
```
systemctl start wg-quick@wg0.service
```
### 8. Check current status
```
systemctl status wg-quick@wg0.service
```
### 9. Create new client keys
```
wg genkey | tee /etc/wireguard/client_privatekey | wg pubkey | tee /etc/wireguard/client_publickey
```
### 10. Add new client
```
cat >> /etc/wireguard/wg0.conf << EOL

[Peer]
PublicKey = <INSERT HERE PUBLIC CLIENT KEY FROM /etc/wireguard/client_publickey>
AllowedIPs = 10.0.0.2/32
EOL
```
### 11. Apply new client config
```
systemctl restart wg-quick@wg0.service
```
### 12. Use next config for your client
```
[Interface]
PrivateKey = <INSERT YOUR PRIVATE KEY HERE FROM /etc/wireguard/client_privatekey>
Address = 10.0.0.2/32
DNS = 8.8.8.8

[Peer]
PublicKey = <INSERT HERE PUBLIC SERVER KEY FROM /etc/wireguard/publickey>
Endpoint = <YOUR VM IP ADDRESS>:27010
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 20
```
### 13. Check your client access
Install wireguard client from https://www.wireguard.com/install/<br>
Import your client config and try to connect<br>

### 14. Dont forget to delete client private key from wireguard server
```
rm /etc/wireguard/client_privatekey
```