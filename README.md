# k8scluster
prepare Ubuntu 22.04 LTS (RPI ZERO 2/3/4/400) X64 node<br>
<br>

logon to installed image with ubuntu:ubuntu<br>
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

## install tools

```
sudo apt install mc
sudo apt install net-tools
sudo apt install nethogs
```

## install docker

install required tools
```
sudo apt-get update

sudo apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg \
    lsb-release    
```

```
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io
```
logon as root
```
su -
```
configure containerd (under root)
```
cat > /etc/containerd/config.toml << EOF
[plugins. " io.containerd.grpc.v1.cri " ]
systemd_cgroup = true 
EOF
```
restart containerd (under root)
```
systemctl restart containerd
```
## install k8s cluster

add k8s key
```
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add
```
add k8s repo
```
sudo apt-add-repository "deb http://apt.kubernetes.io/ kubernetes-xenial main"
```
install k8s tools
```
sudo apt install kubeadm kubelet kubectl kubernetes-cni
```
disable swap
```
sudo swapoff -a
```
cgroups configuration
```
sudo echo group_enable=memory cgroup_memory=1 >> /boot/firmware/cmdline.txt
```
install k8s cluster
```
sudo kubeadm init --pod-network-cidr=10.1.0.0/16 --service-cidr=10.2.0.0/16 --upload-certs
```
save k8s config
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
```
mkdir -p /home/mcfly722/.kube
sudo cp -i /etc/kubernetes/admin.conf /home/mcfly722/.kube/config
sudo chown mcfly722:mcfly722 /home/mcfly722/.kube/config
```

```
reboot
```

## install canal
```
curl https://projectcalico.docs.tigera.io/manifests/canal.yaml -O
kubectl apply -f canal.yaml
```
add worker role to master to run pods on it
```
kubectl label node master node-role.kubernetes.io/worker=worker
kubectl taint nodes master node-role.kubernetes.io/control-plane-
```
---

start rancher container
```
sudo docker run -d \
  -p 8080:80 -p 8443:443 \
  -v /opt/rancher/db:/var/lib/rancher \
  -v /opt/rancher/auditlog:/var/log/auditlog \
  -e AUDIT_LEVEL=1 \
  -e NO_PROXY="localhost,127.0.0.1,0.0.0.0,192.168.0.0/24" \
  --restart=unless-stopped \
  rancher/rancher:v2.4.15
```

to get bootstrap Rancher Password
```
sudo docker ps
sudo docker logs <Container ID> 2>&1 | grep "Bootstrap Password:"
```

to install tools
```
sudo apt install mc
sudo apt install net-tools
sudo apt install nethogs
```

stop snapd service
```
sudo systemctl mask snapd.service
sudo systemctl stop snapd.service
sudo systemctl disable snapd.service
```

to clean node

```
sudo docker stop $(sudo docker ps -aq)
sudo docker system prune -f
sudo docker volume rm $(sudo docker volume ls -q)
sudo docker image rm $(sudo docker image ls -q)
sudo docker rm -f $(sudo docker ps -a -q)
sudo service docker stop

sudo rm -rf /etc/ceph \
       /etc/cni \
       /etc/kubernetes \
       /opt/cni \
       /opt/rke \
       /opt/kubernetes/ssl \
       /opt/rancher \
       /run/secrets/kubernetes.io \
       /run/calico \
       /run/flannel \
       /var/lib/calico \
       /var/lib/etcd \
       /var/lib/docker \
       /var/lib/rancher \
       /var/lib/cni \
       /var/lib/kubelet \
       /var/lib/rancher/rke/log \
       /var/log/containers \
       /var/log/kube-audit \
       /var/log/pods \
       /var/run/calico

sudo service docker start    
```

to rancher download images manually
```
sudo docker pull rancher/coreos-etcd:v3.4.15-rancher1
sudo docker pull rancher/rke-tools:v0.1.75
```
