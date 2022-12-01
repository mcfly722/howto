# deploy K3S Kubernetes cluster
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
