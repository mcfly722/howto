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

sudo docker run -d --restart=unless-stopped \
  -p 80:80 \
  -v /home/rancher:/var/lib/rancher \
  --privileged \
  rancher/rancher:v2.6.0
```
