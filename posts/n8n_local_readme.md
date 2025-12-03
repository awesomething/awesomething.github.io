---
title: n8n Local Setup with systemd (Ubuntu 22+)
date: 2025-05-22
---

# n8n Local Setup with systemd (Ubuntu 22+)

This guide helps you install n8n on a bare-metal Ubuntu 22+ system using systemd and local disk.

---

## 🏗️ Install Steps (bare metal + systemd)

### 1. Create `n8n` user
```bash
sudo useradd -m -s /bin/bash n8n
```

### 2. Install Node.js 20 LTS
```bash
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs
```

### 3. Install n8n
```bash
sudo su - n8n
npm install -g n8n
exit
```

### 4. Create systemd service
```bash
sudo nano /etc/systemd/system/n8n.service
```

Paste this configuration:
```ini
[Unit]
Description=n8n automation
After=network.target

[Service]
Type=simple
User=n8n
Environment=GENERIC_TIMEZONE=Asia/Bangkok
Environment=EXECUTIONS_MODE=regular
Environment=DB_TYPE=sqlite
ExecStart=/usr/bin/n8n
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

> 💡 Change to PostgreSQL later if scaling is needed.

### 5. Start & enable the service
```bash
sudo systemctl daemon-reexec
sudo systemctl daemon-reload
sudo systemctl enable --now n8n
```

### 6. Access n8n
Open your browser at: [http://localhost:5678](http://localhost:5678)

---

## 💾 Disk Allocation Guide

This layout assumes 100GB disk for a small to medium n8n setup:

| Mount Point     | Size       | Notes                                 |
|------------------|------------|----------------------------------------|
| `/`              | 20 GB      | OS + base system                      |
| `/var`           | 20–30 GB   | Logs, PostgreSQL database, runtime data |
| `/home/n8n`      | 20 GB      | n8n user files, working directory     |
| `/opt/n8n_data`  | 20–30 GB   | Optional – backups, uploads, executions |
| `swap`           | 2–4 GB     | Only if RAM < 8 GB                    |

---

## ✅ Notes
- PostgreSQL stores data in `/var/lib/postgresql` by default.
- You can use `/opt/n8n_data` for backups, uploaded files, and executions.
- This setup is ideal for non-Docker environments.
