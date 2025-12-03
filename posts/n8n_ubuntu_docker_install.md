---
title: Install n8n docker on Ubuntu Server
date: 2025-12-02
---

# Install n8n docker on Ubuntu Server!
## Step 1: Update Your Server
```bash
sudo apt-get update && sudo apt-get upgrade -y
```
- Set time
```bash
sudo timedatectl set-timezone Asia/Bangkok
timedatectl
```
# Reboot!

## Step 2: Install and Configure UFW Firewall
```bash
sudo apt install ufw -y
sudo ufw allow 22    # SSH
sudo ufw allow 80    # HTTP
sudo ufw allow 443   # HTTPS
sudo ufw enable
```
- Check status
```sudo ufw status verbose```

## Or Just Disable it!
```bash
sudo ufw disable
```
## Step 3: Docker Installation
- Setup dependencies and Docker's GPG key:
```bash
sudo apt-get update
sudo apt-get install ca-certificates curl gnupg

sudo install -m 0755 -d /etc/apt/keyrings

curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
| sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

sudo chmod a+r /etc/apt/keyrings/docker.gpg
```
- Setup dependencies and Docker's GPG key:
```bash
echo \
  "deb [arch=$(dpkg --print-architecture) \
signed-by=/etc/apt/keyrings/docker.gpg] \
https://download.docker.com/linux/ubuntu \
$(. /etc/os-release && echo $VERSION_CODENAME) stable" \
| sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
```
- Install Docker Engine and compose-plugin:
```bash
sudo apt-get install docker-ce docker-ce-cli \
containerd.io docker-buildx-plugin docker-compose-plugin -y
```
- Check installation:
```bash
sudo docker run hello-world
```
- Example Output
- <img width="796" height="576" alt="Image" src="https://github.com/user-attachments/assets/c1377aa3-430a-494d-bf9b-ee87dacb909c" />

- Configure Docker to start on boot with systemd
```bash
sudo systemctl enable docker.service
sudo systemctl enable containerd.service
```

## Step 4: Installing Caddy for Automatic HTTPS
- Install Caddy:
```bash
sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https curl tcpdump

curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' \
| sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg

curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' \
| sudo tee /etc/apt/sources.list.d/caddy-stable.list

sudo apt update
sudo apt install caddy -y
```
- Edit the Caddyfile configuration file:
```bash
sudo mv /etc/caddy/Caddyfile /etc/caddy/Caddyfile.orig
sudo vi /etc/caddy/Caddyfile
```
- Enter your domain and configure reverse proxy. Replace "yourdomain.com" with your actual domain name:
```bash
yourdomain.com {
    tls internal
    reverse_proxy localhost:5678
}
```
- For port 443
```bash
10.1.200.20 {
    tls internal
    reverse_proxy localhost:5678
}
```
- If no domain yet, use this temporarily, but need to edit n8n "env" for N8N_SECURE_COOKIE=false
```bash
:80 {
    reverse_proxy localhost:5678
}
```
- Restart Caddy
```bash
sudo systemctl restart caddy
```

## Step 5: Installing Caddy for Automatic HTTPS
### Easy start
```bash
sudo docker volume create n8n_data

sudo docker run -it --rm --name n8n -p 5678:5678 -v n8n_data:/home/node/.n8n docker.n8n.io/n8nio/n8n
```

### Clean step
```bash
mkdir ~/n8n-docker
cd ~/n8n-docker
```
- Create `docker-compose.yml` with the following content:
```bash
services:
  n8n:
    image: docker.n8n.io/n8nio/n8n
    container_name: n8n
    restart: always
    ports:
      - "5678:5678"
    volumes:
      - n8n_data:/home/node/.n8n
    environment:
      - TZ=Asia/Bangkok
      - GENERIC_TIMEZONE=Asia/Bangkok
      - N8N_RUNNERS_ENABLED=true
      - N8N_SECURE_COOKIE=false

volumes:
  n8n_data:
```
- Now deploy n8n by running Docker compose:
```bash
sudo docker compose up -d
```
- Viewing Log `sudo docker logs --tail=50 <container_name>`
```bash
sudo docker logs --tail=50 n8n
```

## Step 6: Accessing Your Self-Hosted n8n Instance
Visit your domain in any web browser. Your n8n instance should now load successfully at https://yourdomain.com. Follow the setup steps in the interface to complete your initial setup.

https://<my-ip>
or
http://<my-ip or localhost>:5678

# Updating your n8n Installation
```bash
sudo docker compose pull
sudo docker compose up -d
```

# Maintenace
- Check status ```sudo docker ps```
- Shutdown Container ```sudo docker stop <container_name>```
- Start Container ```sudo docker start <container_name>```
- Restart Container ```sudo docker restart <container_name>```
- Viewing Log ```sudo docker logs --tail=50 <container_name>```
- Remove container ```sudo docker rm <container_name>```

