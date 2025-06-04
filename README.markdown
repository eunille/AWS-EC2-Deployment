# 🚀 Deploying Full Stack Apps to AWS EC2 with Postgres Sequalize

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
rsync -avz --exclude 'node_modules' --exclude '.git' --exclude '.env' -e "ssh -i ~/.ssh/NoteAlone.pem" . ubuntu@ec2-3-27-90-170.ap-southeast-2.compute.amazonaws.com:~/app
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

4. **Configure Database User**
   - Access PostgreSQL as the `postgres` user:
     ```bash
     psql
     ```
   - Create the database and user (we eventually used `yunil`):
     ```sql
     CREATE DATABASE notealone;
     CREATE USER yunil WITH PASSWORD 'your_secure_password';
     GRANT ALL PRIVILEGES ON DATABASE notealone TO yunil;
     ALTER USER yunil CREATEDB;
     ALTER USER yunil CREATEROLE;
     ALTER USER yunil SUPERUSER;
     \q
     ```

5. **Update PostgreSQL Authentication**
   - Edit the `pg_hba.conf` file to allow password authentication for `yunil`:
     ```bash
     sudo nano /etc/postgresql/16/main/pg_hba.conf
     ```
   - Add:
     ```
     local   notealone   yunil   md5
     local   all         all     peer
     ```
   - Reload PostgreSQL:
     ```bash
     sudo systemctl reload postgresql
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
   User=Ubuntu
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
   notealone.xyz {
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

## 🛠️ Problems Encountered

During the deployment of the `notealone-api`, we encountered several issues. Below is a detailed breakdown of each problem and how we resolved it.

### 1. Permission Denied Error When Copying SSH Key for `rsync`

**Problem**: When attempting to copy the SSH private key (`NoteAlone.pem`) from `/mnt/c/ssh/` to `/home/eunilleapo/.ssh/` in WSL (Windows Subsystem for Linux) to use with `rsync`, we encountered a "Permission denied" error:
```
cp: cannot create regular file '/home/eunilleapo/.ssh/NoteAlone.pem': Permission denied
```
This occurred despite the `.ssh` directory having correct permissions (`drwx------`).

**Root Cause**: The issue stemmed from WSL user permission misconfigurations, likely due to interference between the Windows NTFS file system (`/mnt/c`) and WSL’s Linux file system, or a corrupted WSL user state.

**Resolution**:
- Used `sudo` to bypass permission issues and copy the file:
  ```bash
  sudo cp /mnt/c/ssh/NoteAlone.pem ~/.ssh/NoteAlone.pem
  ```
- Fixed ownership and permissions:
  ```bash
  sudo chown eunilleapo:eunilleapo ~/.ssh/NoteAlone.pem
  chmod 600 ~/.ssh/NoteAlone.pem
  ```
- Verified the file’s presence:
  ```bash
  ls -l /home/eunilleapo/.ssh/NoteAlone.pem
  ```
- Ensured the `.ssh` directory had the correct permissions:
  ```bash
  chmod 700 ~/.ssh
  ```
- Successfully used `rsync` to deploy the code:
  ```bash
  rsync -avz --exclude 'node_modules' --exclude '.git' --exclude '.env' -e "ssh -i ~/.ssh/NoteAlone.pem" . ubuntu@ec2-3-27-90-170.ap-southeast-2.compute.amazonaws.com:~/app
  ```

**Lesson Learned**: WSL can have permission conflicts between Windows and Linux file systems. Using `sudo` and ensuring correct ownership/permissions resolves such issues. Always verify file permissions after copying between file systems.

---

### 2. `myapp.service` Failed with "start-limit-hit" Error

**Problem**: After deploying the code and starting the systemd service (`myapp.service`), it failed with:
```
Active: failed (Result: start-limit-hit) since Thu 2025-05-29 10:40:19 UTC; 227ms ago
```
The logs indicated the service was restarting too quickly because the Node.js process (`index.js`) was exiting immediately.

**Root Cause**: The Node.js application was crashing due to a database connection error (detailed in the next section), causing systemd to restart it repeatedly until it hit the default restart limit.

**Resolution**:
- Reset the failed state to allow retries:
  ```bash
  sudo systemctl reset-failed
  ```
- Modified the service file to increase restart tolerance:
  ```bash
  sudo nano /etc/systemd/system/myapp.service
  ```
  Added under `[Service]`:
  ```
  Restart=always
  RestartSec=5
  StartLimitInterval=0
  ```
- Reloaded and restarted the service:
  ```bash
  sudo systemctl daemon-reload
  sudo systemctl start myapp.service
  ```
- This allowed us to focus on fixing the underlying database issue without systemd blocking further start attempts.

**Lesson Learned**: The `start-limit-hit` error indicates an underlying application crash, not a systemd issue. Adjusting restart settings can help debug the root cause, but the application’s error (e.g., database connection) must be resolved.

---

### 3. `myapp.service` Failed Due to Missing `.env` File

**Problem**: The service failed with:
```
myapp.service: Failed to load environment files: No such file or directory
```
The service file specified `EnvironmentFile=/home/ubuntu/app/.env`, but the `.env` file was missing.

**Root Cause**: During `rsync`, the `.env` file was excluded (`--exclude '.env'`), and it wasn’t manually created on the EC2 instance.

**Resolution**:
- Created the `.env` file on the EC2 instance:
  ```bash
  nano /home/ubuntu/app/.env
  ```
  Added:
  ```
  PORT=5000
  JWT_SECRET=evolisto
  ```
- Set appropriate permissions:
  ```bash
  chmod 600 /home/ubuntu/app/.env
  chown ubuntu:ubuntu /home/ubuntu/app/.env
  ```
- Reloaded and restarted the service:
  ```bash
  sudo systemctl daemon-reload
  sudo systemctl reset-failed
  sudo systemctl start myapp.service
  ```

**Lesson Learned**: Always ensure environment files are transferred or created on the server if excluded during `rsync`. Verify the `EnvironmentFile` path in the service file matches the actual file location.

---

### 4. Database Connection Error: Password Authentication Failed

**Problem**: The service failed with:
```
Unable to connect to the database: ConnectionError: password authentication failed for user "notealone_user"
```
The app tried to connect using `notealone_user`, which didn’t exist in PostgreSQL.

**Root Cause**: The `DATABASE_URL` in `.env` referenced `notealone_user`, but PostgreSQL only had `postgres` and `yunil` users (`\du` output). Additionally, the `postgres` user had no password, causing further mismatches with Sequelize’s config expecting a password (`evolisto24`).

**Resolution**:
- Listed PostgreSQL roles:
  ```sql
  \du
  ```
  Output:
  ```
  Role name | Attributes
  -----------+------------------------------------------------------------
  postgres  | Superuser, Create role, Create DB, Replication, Bypass RLS
  yunil     | Superuser, Create role, Create DB
  ```
- Initially tried creating `notealone_user`:
  ```sql
  CREATE USER notealone_user WITH PASSWORD 'evolisto24';
  GRANT ALL PRIVILEGES ON DATABASE notealone TO notealone_user;
  ```
- Encountered a `peer` authentication error:
  ```
  FATAL: Peer authentication failed for user "notealone_user"
  ```
- Updated `pg_hba.conf` to use `md5` authentication for `notealone_user`:
  ```bash
  sudo nano /etc/postgresql/16/main/pg_hba.conf
  ```
  Added:
  ```
  local   notealone   notealone_user   md5
  ```
  Reloaded PostgreSQL:
  ```bash
  sudo systemctl reload postgresql
  ```
- Decided to switch to the existing `yunil` user (a superuser) to simplify setup:
  - Updated `.env`:
    ```bash
    nano /home/ubuntu/app/.env
    ```
    Set:
    ```
    DATABASE_URL=postgres://yunil:your_secure_password@localhost:5432/notealone
    ```
  - Updated Sequelize config:
    ```bash
    nano /home/ubuntu/app/config/config.json
    ```
    Set:
    ```json
    "production": {
      "username": "yunil",
      "password": "your_secure_password",
      "database": "notealone",
      "host": "127.0.0.1",
      "dialect": "postgres"
    }
    ```
- Updated `pg_hba.conf` for `yunil`:
  ```
  local   notealone   yunil   md5
  local   all         all     peer
  ```
- Reloaded PostgreSQL and restarted the service:
  ```bash
  sudo systemctl reload postgresql
  sudo systemctl daemon-reload
  sudo systemctl reset-failed
  sudo systemctl start myapp.service
  ```
- The service started successfully after these changes.

**Lesson Learned**: Ensure the database user in your app’s configuration matches an existing PostgreSQL role. Check `pg_hba.conf` for the correct authentication method (`md5` for password-based, `peer` for local system user matching). Using a superuser like `yunil` simplifies setup but is less secure for production—consider a dedicated user with limited privileges.

---

### 5. CORS Configuration Update

**Problem**: The frontend (`https://note-alone-ewvjr42sa-eunilles-projects.vercel.app`) was blocked by CORS when making requests to the backend (`https://notealone.xyz/api/`).

**Root Cause**: The backend (`index.js`) lacked proper CORS configuration to allow requests from the frontend domain.

**Resolution**:
- Updated `index.js` to include CORS middleware:
  ```javascript
  const cors = require('cors');
  app.use(
    cors({
      origin: "https://note-alone-ewvjr42sa-eunilles-projects.vercel.app",
      credentials: true,
      methods: ["GET", "POST", "PUT", "DELETE"],
      allowedHeaders: ["Content-Type", "Authorization"],
    })
  );
  ```
- Redeployed the updated code using `rsync` (as shown above).
- Restarted the service:
  ```bash
  sudo systemctl restart myapp.service
  ```

**Lesson Learned**: Ensure CORS is configured to allow your frontend domain, especially when deploying to production. Test API calls from the frontend to confirm CORS settings work as expected.

---

## 📌 Final Configuration Notes

- **Database User**: We settled on using the `yunil` PostgreSQL user (a superuser) for simplicity. For better security, consider creating a dedicated user with limited privileges in production.
- **Environment Variables**: The `.env` file must be manually created or copied to the EC2 instance if excluded during `rsync`.
- **Service Management**: Use `sudo systemctl reset-failed` to clear `start-limit-hit` errors during debugging.
- **Used Cadaddy And Namecheap for Domain Used Vercel**: Cost 115 pesos.

---

## 📌 TechStack and Tools Used
- **ReactTS**: Frontend
- **ExpressJS**: Backend
- **Postgres Sequalize**: Database
- **Grok**: AI Model Provider
- **Figma**: UI Design
