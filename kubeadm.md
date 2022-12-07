# Installing Kubernetes Cluster using kubeadm

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
start required kernel modules (you can just restart the node for it)
```
sudo modprobe overlay
sudo modprobe br_netfilter
```
test that required kernel modules are loaded
```
lsmod | egrep 'br_netfilter|overlay'
```
you have to find <b>br_netfilter</b> overlay on output.
### 5. add kernel ip configuration
```
cat <<EOT > /etc/sysctl.d/kubernetes.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOT
```
apply new kernel configuration and make it persistent
```
sysctl --system
sysctl -p
```

### 6. install containerd
Add docker repo:
```
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | apt-key add -
add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
apt update
```
#### 6.1. install containerd
```
apt install containerd.io
```
#### 6.2. Create default containerd configuration
```
containerd config default > /etc/containerd/config.toml
```
#### 6.3. Turn on SystemdCgroup for containerd
Open <b>/etc/containerd/config.toml</b>, search next key and set <b>SystemdCgroup = true</b>
```
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
  SystemdCgroup = true
```
#### 6.4. Specify Containerd proxy (if required)
```
mkdir /etc/systemd/system/containerd.service.d
cat <<EOT > /etc/systemd/system/containerd.service.d/http-proxy.conf
[Service]
Environment="HTTP_PROXY=http://<username>:<password>@<proxy url>:<proxy port>"
Environment="HTTPS_PROXY=http://<username>:<password>@<proxy url>:<proxy port>"
EOT
```
#### 6.5. Apply configuration and restart Containerd service
```
systemctl daemon-reload
systemctl start containerd
```

### 7. Install kubelet kubeadm kubectl
```
curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
```
```
apt install -y kubelet kubeadm kubectl
```
```
apt-mark hold kubelet kubeadm kubectl
```

### 8. Pull required kubernetes images
```
kubeadm config images pull
```
### 16. Enable kubelet
```
systemctl enable kubelet
```

## Install First Master
To manage k8s cluster and join nodes to it, it is very confortable to specify DNS name. In future you could change it IP API Server address or envolve HA Balancer.

### 1. Configure API Server DNS name on first master
Add to /etc/host new endpoint cluster address
```
127.0.0.1 <FQDN DNS K8S API Service name>
```

### 2. Initialize new cluster
```
kubeadm init --pod-network-cidr=10.1.0.0/16 --service-cidr=10.0.0.0/16 --control-plane-endpoint <FQDN DNS K8S API Service name>:6443
```
You can specify your own pod/service CIDR ranges for internal K8S networks
### 3. Wait cluster successful initialization
### 4. Save .kube configuration to be able use kubeadm and other tools
```
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
```

## Join second and further masters
### 1. Obtain certificate-key (from working master)
```
kubeadm init phase upload-certs --upload-certs
```
### 2. Obtain token and discovery-token
```
kubeadm token create --print-join-command
```

### 3. Join new master to existing k8s cluster
```
kubeadm join <FQDN DNS K8S API Service name>:6443 --apiserver-advertise-address <new master IP> --control-plane --token <yout token> --discovery-token-ca-cert-hash <your discovery-token> --certificate-key <certificate-key>
```
---
## Join Worker to existing K8S cluster
### 1. get join command from one of masters
```
kubeadm token create --print-join-command
```
use this command on worker node to join it to k8s cluster

### 2. after successful join this node, mark it as 'worker'
```
kubectl label node <nodename> node-role.kubernetes.io/worker=worker
```
## Configuring K8S Cluster Components

### Install network plugin (flannel)

#### 1. download flannel yaml
```
wget https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
```

#### 2. modify default namespace 'kube-flannel' to 'kube-system'
```
sed -i 's/namespace: kube-flannel/namespace: kube-system/g' kube-flannel.yml
```

#### 3. modify 10.244.0.0/16 network to 10.42.0.0/16 (or you can use your own network)
```
sed -i 's/10.244.0.0\/16/10.42.0.0\/16/g' kube-flannel.yml
```

#### 4. deploy flannel
```
kubectl apply -f kube-flannel.yml
```


### Install HELM (on one of master nodes)
https://helm.sh/docs/intro/install/
```
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```

### Install nginx-Ingress-controller

#### 1. mark all master nodes with ingress role labels
```
./kubectl label nodes <NODE NAME> node-role.kubernetes/ingress=
```
#### 2. install ingress controller and schedule it on master nodes
https://docs.nginx.com/nginx-ingress-controller/installation/installation-with-helm/
```
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx

helm install ingress-nginx ingress-nginx/ingress-nginx \
--set controller.kind=DaemonSet \
--set controller.service.type=NodePort \
--set controller.service.httpPort.nodePort=80 \
--set controller.service.httpsPort.nodePort=443 \
--set-json 'controller.nodeSelector={"node-role.kubernetes/ingress":""}' \
--set-json 'controller.tolerations=[{"key":"","operator":"Exists","effect":"NoSchedule"}]' \
--set-json 'controller.service.externalIPs=["EXTERNAL IP#1","EXTERNAL IP#2","EXTERNAL IP#3"]' \
-n kube-system
```
for uninstalling you can use:
```
helm uninstall ingress-nginx -n kube-system
```
### Install MetalLB Balancer
```
helm repo add metallb https://metallb.github.io/metallb
helm install metallb --namespace kube-system metallb/metallb
```
### Install Kubernetes dashboard
#### 1. Deploy service
```
helm repo add kubernetes-dashboard https://kubernetes.github.io/dashboard/
helm install kubernetes-dashboard --namespace kube-system kubernetes-dashboard/kubernetes-dashboard
```

### Install K8S dashboard
#### 1. install dashboard
```
helm repo add kubernetes-dashboard https://kubernetes.github.io/dashboard/

helm install \
--namespace kube-system \
--set ingress.enabled=true \
--set replicaCount=2 \
--set ingress.hosts={<SPECIFY DASHBOARD SITE FQDN>} \
--set enable-skip-login=true \
--set enable-insecure-login=true \
--set serviceAccount.create=true \
--set serviceAccount.name=dashboard-admin \
--set rbac.clusterReadOnlyRole=false \
--set rbac.clusterAdminRole=true \
--set-json 'ingress.annotations={"kubernetes.io\/ingress.class":"nginx"}' \
kubernetes-dashboard kubernetes-dashboard/kubernetes-dashboard
```
#### 2. Generate Dashboard Access Token
```
kubectl create token dashboard-admin --namespace kube-system --duration 87600h
```
Use this to access to dashboard
#### 3. Grant full permissions to dashboard-admin serviceAccount
```
kubectl create clusterrolebinding kubernetes-dashboard-admin --clusterrole=cluster-admin --serviceaccount=kube-system:dashboard-admin
```
### Install Grafana
```
helm repo add grafana https://grafana.github.io/helm-charts

helm upgrade --install \
--create-namespace \
--namespace grafana \
--set adminUser=admin \
--set adminPassword=<ADMIN PASSWORD> \
--set ingress.enabled=true \
--set ingress.hosts={<GRAFANA FQDN HOST NAME>} \
--set-json 'ingress.annotations={"kubernetes.io\/ingress.class":"nginx"}' \
grafana grafana/grafana
```
