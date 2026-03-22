# # Installing Kubernetes Cluster using k3s

## Prepearing K8S node (ControlPlane / Worker)
### 1.add vxlan support (required for kubernetes network plugin stable work)
https://github.com/k3s-io/k3s/issues/4234
```
sudo apt-get update
sudo apt install linux-modules-extra-raspi && sudo reboot
```
### 2.install k3s
(https://rancher.com/docs/k3s/latest/en/installation/install-options/)
```
export K3S_KUBECONFIG_MODE="644"
export INSTALL_K3S_VERSION="v1.23.9+k3s1"
export INSTALL_K3S_EXEC="--disable-network-policy --cluster-cidr=10.1.0.0/16 --service-cidr=10.0.0.0/16 --kube-proxy-arg=proxy-mode=ipvs --disable=traefik --disable=servicelb"

curl -sfL https://get.k3s.io | sh -

export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
```

## Configuring K8S Cluster Components
### 1. install helm
https://helm.sh/docs/intro/install/
```
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
sudo apt-get install apt-transport-https --yes
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm
```
### 2. install metallb
(https://metallb.universe.tf/installation/#installation-with-helm)
```
helm repo add metallb https://metallb.github.io/metallb
helm install --namespace metallb-system --create-namespace metallb metallb/metallb
```
### 4. install nginx Ingress Controller
(https://docs.nginx.com/nginx-ingress-controller/installation/installation-with-helm/)
```
helm repo add nginx-stable https://helm.nginx.com/stable

helm install \
 --namespace ingress-system \
 --create-namespace \
 --set controller.service.annotations."metallb\.universe\.tf/address-pool"=ingress-ip-pool \
 ingress-system nginx-stable/nginx-ingress
```
