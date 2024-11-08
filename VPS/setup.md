# Racknerd VPS lockdown

## set up the VPS with ubuntu-server 22.04

### update
- apt update && apt upgrade -y

### create non-root user
- adduser \<user\>
- usermod -aG sudo \<user\>
- su - \<user\>

### set up UFW
- sudo ufw default deny incoming
- sudo ufw default allow outgoing
- sudo ufw allow http
- sudo ufw allow https
- sudo ufw allow OpenSSH
- sudo ufw enable
- sudo ufw status verbose

## make sure you can still connect to your VPS as a non-root user
<p style="color: red; font-weight: bold;">Important: leave the original ssh session open!</p>
- start a new session and log in using your credentials on port 22
- if succesful, then continue

### change ssh port and config
- sudo nano /etc/ssh/sshd_config
- change and/or uncomment the following lines:
  - Port 22 - to whatever port you want over 1000 
  - AddressFamily inet <---  this only allows IPv4 traffic
  - PermitRootLogin no 
  - PubkeyAuthentication yes
  - AuthorizedKeysFile      .ssh/authorized_keys
  - PasswordAuthentication yes <--  leave yes for now
  - PermitEmptyPasswords no
- save config
- sudo systemctl restart ssh
  
### confirm ssh is listening on the new port
- sudo ss -tulpn | grep ssh  -or-
- sudo netstat -tulpn | grep ssh  -or-
- sudo lsof -i :<port_number>

### set up your key pair for SSH
- ssh-keygen -t ed25519 -C "name"   -or-
- ssh-keygen -t rsa -b 4096 -C "name"
for the name, I usually use the name of the machine I will be connecting from so I can easily identify what keys go to what machines

### add key to remote VPS
- ssh-copy-id -i ~/.ssh/keyname.pub user@remote_server_ip
- *"-i ~/.ssh/keyname.pub" is only needed if you didnt use the default name (id_rsa), but I always name mine something useful so yours may instead look like:*
- ssh-copy-id -i ~/.ssh/ubuntu-server1.pub user@remote_server_ip
- sudo systemctl restart ssh

##-OR-

### manually add key to remote VPS
- mkdir -p ~/.ssh
- chmod 700 ~/.ssh
- echo "yourkey.pub_here" >> ~/.ssh/authorized_keys
- chmod 600 ~/.ssh/authorized_keys
- sudo systemctl restart ssh

## test the key pair
### make sure you can connect to your VPS with the new SSH settings, key pair, and non-root user
- as before, leave the original ssh session open
- start a new session and log in using your name@ip:\<new_port\>
- if succesful, then continue

### add docker and add yourself to docker group
- curl -sSL https://get.docker.com | sh && sudo usermod -aG docker \<user\>  && sudo usermod -aG docker $USER

### check timezone
- sudo dpkg-reconfigure tzdata

### set up unattended updates
- sudo dpkg-reconfigure --priority=low unattended-upgrades
