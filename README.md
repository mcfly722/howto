# k8scluster
download and install Raspberry Pi Imager
https://www.raspberrypi.com/software/

Write <b>Ubuntu 22.04 LTS (RPI ZERO 2/3/4/400) X64</b> image on sd-card and put it to <b>Raspberry 4B</b><br>
<br>

logon to installed image with <b>ubuntu:ubuntu</b><br>
change default password<br>
logon again, but through ssh<br>
<br>
rename node<br>
```
sudo hostnamectl set-hostname 'master'
```
## add new user (mcfly722)
```
sudo adduser mcfly722
sudo usermod -a -G admin mcfly722

sudo mkdir /home/mcfly722/.ssh
sudo chown mcfly722 /home/mcfly722/.ssh
sudo chgrp mcfly722 /home/mcfly722/.ssh
sudo chmod 700 /home/mcfly722/.ssh
exit
```
```
sudo bash

cd /home/mcfly722/.ssh/
echo 'ecdsa-sha2-nistp384 AAAAE2VjZHNhLXNoYTItbmlzdHAzODQAAAAIbmlzdHAzODQAAABhBEdHq09BV7XByoDXGD3sI/1KvrJR9LNLMsUq1zZtkx8rqiNSFDrUmoJXonzX3PmwKBM9cWqNDiC1zHeYP7nWCvg6wIH0msqc5KN6nU6zVv32szOV6TFNyMSYMMJDKITJ8g==  CAPI:0d8f679eb028d7d2cc39f7a9fdb53535c6e6407d E=7221798@gmail.com' > authorized_keys
echo 'net.ifnames=0 dwc_otg.lpm_enable=0 console=serial0,115200 console=tty1 root=LABEL=writable rootfstype=ext4 elevator=deadline rootwait fixrtc cgroup_enable=cpuset cgroup_memory=1 cgroup_enable=memory' > /boot/firmware/cmdline.txt

sudo chmod 600 authorized_keys
sudo chown mcfly722 authorized_keys
sudo chgrp mcfly722 authorized_keys

echo 'PermitRootLogin no' >> /etc/ssh/sshd_config
echo 'mcfly722 ALL=(ALL) NOPASSWD:ALL' >> /etc/sudoers

exit
```
logon as mcfly and delete ubuntu user
```
sudo deluser --remove-home ubuntu
```
change root password
```
sudo passwd root
```
## add vxlan support (required for kubernetes network plugin stable work)
https://github.com/k3s-io/k3s/issues/4234
```
sudo apt-get update
sudo apt install linux-modules-extra-raspi && sudo reboot
```
## install k3s
(https://rancher.com/docs/k3s/latest/en/installation/install-options/)
```
export K3S_KUBECONFIG_MODE="644"
export INSTALL_K3S_VERSION="v1.23.9+k3s1"
export INSTALL_K3S_EXEC="--disable-network-policy --cluster-cidr=10.1.0.0/16 --service-cidr=10.0.0.0/16 --kube-proxy-arg=proxy-mode=ipvs --disable=traefik --disable=servicelb"

curl -sfL https://get.k3s.io | sh -

export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
```
## install helm
https://helm.sh/docs/intro/install/
```
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
sudo apt-get install apt-transport-https --yes
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm
```
## install metallb
(https://metallb.universe.tf/installation/#installation-with-helm)
```
helm repo add metallb https://metallb.github.io/metallb
helm install --namespace metallb-system --create-namespace metallb metallb/metallb
```
## add metallb ip pool for ingress-controller
```
cat <<EOT > metallb-IPAddressPool.yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: ingress-ip-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.0.201/32
EOT
```
```
kubectl apply -f metallb-IPAddressPool.yaml
```
## install Nginx Ingress Controller
(https://docs.nginx.com/nginx-ingress-controller/installation/installation-with-helm/)
```
helm repo add nginx-stable https://helm.nginx.com/stable

helm install \
 --namespace ingress-system \
 --create-namespace \
 --set controller.service.annotations."metallb\.universe\.tf/address-pool"=ingress-ip-pool \
 ingress-system nginx-stable/nginx-ingress 
```
## issue root CA Certificate
```
openssl req -x509 -nodes \
 -newkey rsa:4096 \
 -keyout rootCA.key \
 -out rootCA.crt \
 -days 365000 \
 -subj '/CN=59ff44dd.nip.io/O=Company' \
 -addext 'keyUsage=cRLSign, digitalSignature, keyCertSign'
```
## install dashboard
(https://artifacthub.io/packages/helm/k8s-dashboard/kubernetes-dashboard )

### issue dashboard Web certificate
```
openssl req -x509 -nodes \
 -newkey rsa:4096 \
 -CA rootCA.crt \
 -CAkey rootCA.key \
 -days 365000 \
 -subj '/CN=dashboard.59ff44dd.nip.io' \
 -addext 'extendedKeyUsage=1.3.6.1.5.5.7.3.1' \
 -addext 'keyUsage=keyEncipherment' \
 -keyout dashboard-web.key \
 -out dashboard-web.crt
```
### install dashboard
```
helm repo add kubernetes-dashboard https://kubernetes.github.io/dashboard/

helm install \
 --namespace dashboard-system \
 --create-namespace \
kubernetes-dashboard kubernetes-dashboard/kubernetes-dashboard
```
### add dashboard web certificate to k8s secret
```
kubectl create secret tls dashboard-web-tls \
  --namespace dashboard-system \
  --key dashboard-web.key \
  --cert dashboard-web.crt
```
### create ingress rule for dashboard
```
cat <<EOT > dashboard-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: dashboard
  namespace: dashboard-system
  annotations:
    kubernetes.io/ingress.class: "nginx"
    nginx.org/ssl-services: "kubernetes-dashboard"
spec:
  rules:
  - host: "dashboard.59ff44dd.nip.io"
    http:
      paths:
      - path: "/"
        pathType: Prefix
        backend:
          service:
            name: kubernetes-dashboard
            port:
              number: 443
  tls:
    - hosts:
      - dashboard.59ff44dd.nip.io
      secretName: dashboard-web-tls
EOT
```
```
kubectl apply -f dashboard-ingress.yaml
```
### create k8s service account
https://github.com/kubernetes/dashboard/blob/master/docs/user/access-control/creating-sample-user.md

```
kubectl create serviceaccount mcfly722
```
### create k8s service account role binding

```
cat <<EOT > mcfly722-clusterRoleBinding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: mcfly722
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: mcfly722
  namespace: default
EOT
```
```
kubectl apply -f mcfly722-clusterRoleBinding.yaml
```
### get k8s service account token
```
kubectl get secrets -o jsonpath="{.items[?(@.metadata.annotations['kubernetes\.io/service-account\.name']=='mcfly722')].data.token}"|base64 --decode
```
---
## (not required) calculate local-ip.co address for external connections
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
