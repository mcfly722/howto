# not required additions

## calculate local-ip.co address for external connections
```
IP              = 89.255.68.221
IP Hex          = 59.FF.44.DD
IP Hex Reversed = DD.44.FF.59
Num Reversed    = 3712286553
Local-IP        = 1pe76mx
nip.io addr     = 59ff44dd.nip.io
```
To convert Reversed Number to local-IP use calculator:
https://www.rapidtables.com/convert/number/base-converter.html
with Base=36


## install Rancher
https://rancher.com/docs/rancher/v2.6/en/installation/other-installation-methods/behind-proxy/install-rancher/

### install cert manager
```
helm repo add jetstack https://charts.jetstack.io

kubectl create namespace cert-manager

kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.7.1/cert-manager.crds.yaml

helm upgrade --install cert-manager jetstack/cert-manager --namespace cert-manager --version v1.7.1
```
### issue Rancher Web certificate
```
openssl req -x509 -nodes \
 -newkey rsa:4096 \
 -CA rootCA.crt \
 -CAkey rootCA.key \
 -days 365000 \
 -subj '/CN=rancher.59ff44dd.nip.io' \
 -addext 'extendedKeyUsage=1.3.6.1.5.5.7.3.1' \
 -addext 'keyUsage=keyEncipherment' \
 -keyout rancher-web.key \
 -out rancher-web.crt
```

### add Rancher web certificate to k8s secret
```
kubectl create secret tls rancher-web-tls \
  --namespace cattle-system \
  --key rancher-web.key \
  --cert rancher-web.crt
```
### install Rancher 
I using v2.5.7 (2.5.8 has Fleet what is not stable enought)<br>
(https://artifacthub.io/packages/helm/rancher-stable/rancher/2.5.7 )

```
helm repo add rancher-stable https://releases.rancher.com/server-charts/stable
helm search repo rancher-stable --versions

kubectl create namespace cattle-system

helm upgrade --install \
--namespace cattle-system \
--set hostname=rancher.59ff44dd.nip.io \
--set ingress.tls.source=rancher-web-tls \
--set replicas=1 \
rancher rancher-stable/rancher --version 2.5.7
```

```
kubectl apply -f rancher-ingress.yaml
```
wait till Rancher container will be started<br>
logon to https://rancher.59ff44dd.nip.io/ and set new admin password<br>

---
## install tools
```
sudo apt install mc
sudo apt install net-tools
sudo apt install nethogs
```
