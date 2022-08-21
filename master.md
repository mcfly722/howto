# Deploy master node

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
