# Installing Kubernetes Cluster using Kkubeadm

---
## Prepearing K8S node (ControlPlane / Worker)

### 1. Configuring proxy (if required)
```
export http_proxy=http://<username>:<password>@<proxy url>:<proxy port>
export https_proxy=http://<username>:<password>@<proxy url>:<proxy port>
```

### 2. install required packages
```
sudo apt-get -y install curl software-properties-common apt-transport-https ca-certificates
curl -s http://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
echo "deb http://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
```
### 3. disable swap file
Open /etc/fstab and comment string with swap.img
To apply change restart required.

### 4. add required kernel modules
```
cat <<EOF > /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
```
### 5. start required kernel modules
```
sudo modprobe overlay
sudo modprobe br_netfilter
```
or just restart node.
### 6. test that required kernel modules are loaded
```
lsmod | egrep 'br_netfilter|overlay'
```
you have to find br_netfilter overlay on output.
### 7. add kernel ip configuration
```
cat <<EOT > /etc/sysctl.d/kubernetes.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOT
```
### 8. apply new kernel configuration and make it persistent
```
sysctl --system
sysctl -p
```

### 9. install containerd
Add docker repo:
```
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | apt-key add -
add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
apt update
```
install containerd
```
apt install containerd.io
```
### 10. Create default containerd configuration
```
containerd config default > /etc/containerd/config.toml
```
### 11. Turn on SystemdCgroup for containerd
Open <b>/etc/containerd/config.toml</b>, search next key and set <b>SystemdCgroup = true</b>
```
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
  SystemdCgroup = true
```
### 12. Specify Containerd proxy (if required)
```
mkdir /etc/systemd/system/containerd.service.d
cat <<EOT > /etc/systemd/system/containerd.service.d/http-proxy.conf
[Service]
Environment="HTTP_PROXY=http://<username>:<password>@<proxy url>:<proxy port>"
Environment="HTTPS_PROXY=http://<username>:<password>@<proxy url>:<proxy port>"
EOT
```

### 13. Apply configuration and restart Containerd service
```
systemctl daemon-reload
systemctl start containerd
```

### 14. Install kubelet kubeadm kubectl
```
curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
apt install -y kubelet kubeadm kubectl
apt-mark hold kubelet kubeadm kubectl
```

### 15. Pull required kubernetes images
```
kubeadm config images pull
```
### 16. Enable kubelet
```
systemctl enable kubelet
```
---
## Install First Master
To manage k8s cluster and join nodes to it, it is very confortable to specify DNS name. In future you could change it IP API Server address or envolve HA Balancer.

### 1. Configure API Server DNS name on first master
Add to /etc/host new endpoint cluster address
```
127.0.0.1 <FQDN DNS K8S API Server name>
```

### 2. Initialize new cluster
```
kubeadm init --pod-network-cidr=10.1.0.0/16 --service-cidr=10.0.0.0/16 --control-plane-endpoint <FQDN DNS K8S API Server name>:6443
```
You can specify your own pod/service CIDR ranges for internal K8S networks
### 3. Wait cluster successful initialization
### 4. Save .kube configuration to be able use kubeadm and other tools
```
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
```
