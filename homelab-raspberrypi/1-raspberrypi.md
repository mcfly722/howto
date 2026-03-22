#### Configuring OS

### 1. Image
download and install Raspberry Pi Imager
https://www.raspberrypi.com/software/

OS=Raspberry PI OS (other) -> Raspberry PI OS Lite (64-bit)

### 2. Configure ssh public key
```
sudo mkdir ~/.ssh
sudo chown $(whoami) ~/.ssh
sudo chgrp $(id -gn) ~/.ssh

sudo chmod 700 ~/.ssh

echo 'ecdsa-sha2-nistp384 AAAAE2VjZHNhLXNoYTItbmlzdHAzODQAAAAIbmlzdHAzODQAAABhBMd3ZCJN3qC8cMG9UbU5bsZFS9vYmynoI4TQe7O8AG1rlChbtmmFYFv+/6N8AXb4fR8xBcREY1hOQCuYmmJp8JD4CPsS3wR3H8nrOYQxlz/J3ctQH5kTG4cpArSiWaFe7w==  CAPI:19d98a2e091114c9bc7ec7e9064b380710c54c9a E=7221798@gmail.com' > ~/.ssh/authorized_keys


sudo chmod 600 ~/.ssh/authorized_keys
sudo chown $(whoami) ~/.ssh/authorized_keys
sudo chgrp $(id -gn) ~/.ssh/authorized_keys
```
### 3. Allow password only for local logins, for all other ssh key only
```
echo 'PermitRootLogin no' | sudo tee -a /etc/ssh/sshd_config
echo 'anonymous ALL=(ALL) NOPASSWD:ALL' | sudo tee -a /etc/sudoers
```
```
sudo systemctl restart sshd
```