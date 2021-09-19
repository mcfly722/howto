# k8scluster
Home cluster
```
sudo hostnamectl set-hostname 'node1'

sudo snap install classic --devmode --edge
sudo snap install docker
sudo snap install curl

sudo classic
sudo apt install mc
sudo apt install nano

# from sudo bash
sudo echo 'dwc_otg.lpm_enable=0 console=serial0,115200 elevator=deadline rng_core.default_quality=700 vt.handoff=2 quiet splash cgroup_enable=cpuset cgroup_memory=1 cgroup_enable=memory' > /run/mnt/ubuntu-seed/cmdline.txt

sudo docker run -d \
  -p 443:443 \
  -v /home/rancher:/var/lib/rancher \
  --privileged \
  --restart=unless-stopped \
  rancher/rancher:v2.6.0
```
