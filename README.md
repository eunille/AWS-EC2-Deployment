# 🚀 Deploying Full Stack Apps to AWS EC2 with SQL Databases

This guide walks you through deploying a full stack Node.js app to an AWS EC2 instance, setting up PostgreSQL, running your app as a systemd service, and using Caddy as a reverse proxy with HTTPS support.

---

## 🛠️ EC2 Instance Setup

1. **Update and Upgrade**
   ```bash
   sudo apt update
   sudo apt upgrade
   ```

2. **Install Node.js**
   ```bash
   curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
   sudo apt-get install -y nodejs
   ```

---

## 🔁 Deploying Code via rsync

Sync your project to the EC2 instance, excluding unnecessary files:

```bash
rsync -avz --exclude 'node_modules' --exclude '.git' --exclude '.env' -e "ssh -i ~/.ssh/your-key.pem" . ubuntu@your-ec2-ip:~/app
```

---

## 🗃️ PostgreSQL Installation

1. **Install PostgreSQL**
   ```bash
   sudo apt install postgresql postgresql-contrib
   ```

2. **Start and Enable PostgreSQL**
   ```bash
   sudo systemctl start postgresql
   sudo systemctl enable postgresql
   ```

3. **Access the PostgreSQL User**
   ```bash
   sudo -i -u postgres
   ```

---

## ⚙️ Create a systemd Service

1. **Create the Service File**
   ```bash
   sudo vim /etc/systemd/system/myapp.service
   ```

2. **Paste the Following Configuration:**
   ```ini
  [Unit]
Description=NoteAlone API
After=network.target

[Service]
User=ubuntu
WorkingDirectory=/home/ubuntu/app
ExecStart=/usr/bin/node /home/ubuntu/app/index.js
Restart=always
Environment=NODE_ENV=production
EnvironmentFile=/home/ubuntu/app/.env
StandardOutput=journal
StandardError=journal
SyslogIdentifier=myapp

[Install]
WantedBy=multi-user.target
   ```

3. **Enable & Start the Service**
   ```bash
   sudo systemctl daemon-reload
   sudo systemctl enable myapp.service
   sudo systemctl start myapp.service
   ```

4. **Check Service Status**
   ```bash
   sudo systemctl status myapp.service
   ```

---

## 📜 View Logs

- **Tail Logs**
  ```bash
  sudo journalctl -fu myapp.service
  ```

- **All Logs**
  ```bash
  sudo journalctl -u myapp.service
  ```

---

## 🌐 Install and Configure Caddy (Reverse Proxy)

### Step 1: Install Caddy

```bash
sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https curl

curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg

curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | sudo tee /etc/apt/sources.list.d/caddy-stable.list

sudo apt update
sudo apt install caddy
```

### Step 2: Basic Reverse Proxy (HTTP)

Edit the Caddyfile:
```bash
sudo vim /etc/caddy/Caddyfile
```

Add:
```
:80 {
    reverse_proxy localhost:5000
}
```

Restart Caddy:
```bash
sudo systemctl restart caddy
```

---

### Step 3: Enable HTTPS (Production)

1. **Edit Caddyfile with Domain**
   ```bash
   sudo vim /etc/caddy/Caddyfile
   ```

   Replace with:
   ```
   yourdomain.com {
       reverse_proxy localhost:5000
   }
   ```

2. **Restart Caddy**
   ```bash
   sudo systemctl restart caddy
   ```

---

## ✅ Summary

| Component   | Purpose                      |
|-------------|------------------------------|
| Node.js     | Backend JavaScript runtime   |
| PostgreSQL  | Relational database          |
| systemd     | App auto-start and management|
| rsync       | Deploy files to EC2          |
| Caddy       | Reverse proxy + HTTPS        |

---

## 📎 Notes

- Ensure ports `80` and `443` are open in your EC2 security group.
- Use `.env` for secure environment variables.
- Caddy automatically issues HTTPS certificates with Let's Encrypt.

---


