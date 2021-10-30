a# k8scluster
prepare Ubuntu 20.04 node
```
sudo hostnamectl set-hostname 'node<#>'

sudo adduser mcfly722
sudo usermod -a -G admin mcfly722

sudo mkdir /home/mcfly722/.ssh
sudo chown mcfly722 /home/mcfly722/.ssh
sudo chgrp mcfly722 /home/mcfly722/.ssh
sudo chmod 700 /home/mcfly722/.ssh

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

sudo deluser --remove-home ubuntu

sudo passwd root

```

install docker
```
sudo apt-get update

sudo apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg \
    lsb-release

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

echo \
  "deb [arch=arm64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io

```

start rancher container
```
sudo docker run -d \
  -p 8080:80 -p 8443:443 \
  -v /opt/rancher:/var/lib/rancher \
  -v /opt/kubernetes/ssl:/var/lib/kubernetes/ssl \
  -e SSL_CERT_DIR="/var/lib/kubernetes/ssl" \
  --privileged \
  --restart=unless-stopped \
  -e AUDIT_LEVEL=3 \
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

stop napd service
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
```
