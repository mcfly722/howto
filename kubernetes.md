# Configuring Kubernetes Cluster

## 1. install helm
https://helm.sh/docs/intro/install/
```
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
sudo apt-get install apt-transport-https --yes
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm
```
## 2. install metallb
(https://metallb.universe.tf/installation/#installation-with-helm)
```
helm repo add metallb https://metallb.github.io/metallb
helm install --namespace metallb-system --create-namespace metallb metallb/metallb
```
## 3. add metallb ip pool for ingress-controller
```
cat <<EOT > metallb-IPAddressPool.yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: ingress-ip-pool
  namespace: metallb-system
spec:
  addresses:
  - <specify your ip address>/32
EOT
```
```
kubectl apply -f metallb-IPAddressPool.yaml
```
## 4. install Nginx Ingress Controller
(https://docs.nginx.com/nginx-ingress-controller/installation/installation-with-helm/)
```
helm repo add nginx-stable https://helm.nginx.com/stable

helm install \
 --namespace ingress-system \
 --create-namespace \
 --set controller.service.annotations."metallb\.universe\.tf/address-pool"=ingress-ip-pool \
 ingress-system nginx-stable/nginx-ingress
```
## 5. issue root CA Certificate (only for myself)
```
openssl req -x509 -nodes \
 -newkey rsa:4096 \
 -keyout rootCA.key \
 -out rootCA.crt \
 -days 365000 \
 -subj '/CN=59ff44dd.nip.io/O=Company' \
 -addext 'keyUsage=cRLSign, digitalSignature, keyCertSign'
```
