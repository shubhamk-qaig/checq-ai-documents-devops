# ChecQ-AI Cloud Deployment Guide for Demo Environment on AWS Cloud 

## 1. System Overview & Metadata

| Parameter | Value / Details |
| :--- | :--- |
| **Environment** | Demo |
| **Cloud Provider** | AWS |
| **AWS Region** | `ap-south-1` |
| **Instance Name** | `ChecQ-AI-Demo` |
| **Instance ID** | `i-0886c11f2acae6a72` |
| **Instance Type** | `t3.medium` (2 vCPU, 4 GB RAM, 60 GB Disk) |
| **Operating System** | Ubuntu 24.04 LTS (or Ubuntu 26 LTS preview) |
| **Elastic IP** | `13.203.61.50` (`CHECQ-AI-NGINX`) |
| **SSH Key** | `checq-ai-demo.pem` |

---

## 2. Prerequisites & Architecture Stack

* **Minimum Hardware**: 2 vCPUs, 4 GB RAM, 30+ GB Disk
* **Network & Domain**: AWS Elastic IP mapped to your domain with an `A` record
* **Runtimes & Engines**:
  * Node.js v22+ LTS (v24.22.1 used below)
  * Process Manager 2 (`pm2`)
  * PostgreSQL v17+
  * Nginx Reverse Proxy with SSL (Certbot / Let's Encrypt)
  * Go Engine (`golang-go`) & Suricata IDS Engine

---

## 3. Core Prerequisites Installation

### Node.js (v24 LTS)
```bash
# Download Node.js binary archive
wget https://nodejs.org/dist/v24.22.1/node-v24.22.1-linux-x64.tar.xz

# Extract archive
tar -xvf node-v24.22.1-linux-x64.tar.xz

# Copy binaries and libraries to system paths
sudo cp -r node-v24.22.1-linux-x64/{bin,include,lib,share} /usr/

# Verify installation
node --version
npm --version
```

### PM2 Process Manager
```bash
sudo npm install pm2@latest -g
```

### PostgreSQL Database
```bash
# Install PostgreSQL
sudo apt update && sudo apt install postgresql postgresql-contrib -y

# Verify service status
sudo systemctl status postgresql

# Set postgres superuser password
sudo -u postgres psql -c "\password postgres"

# Set up Database and User
sudo -u postgres psql << 'EOF'
CREATE USER checq_ai WITH PASSWORD 'PassDB@123';
CREATE DATABASE checq_ai OWNER checq_ai;
GRANT ALL PRIVILEGES ON DATABASE checq_ai TO checq_ai;
\c checq_ai
GRANT ALL ON SCHEMA public TO checq_ai;
EOF

# Test Database connection
psql -U checq_ai -d checq_ai -h 127.0.0.1 -W
```

### Directory Structure & Permissions
```bash
# Create application directories
sudo mkdir -p /var/opt/checq-ai/{dashboard,nmd,server}

# Grant current user ownership
sudo chown -R $USER:$USER /var/opt/checq-ai/
sudo chmod -R 775 /var/opt/checq-ai/
```

---

## 4. Service Deployments

### A. ChecQ-AI Dashboard (Frontend React App)

1. **Deploy Source Code**: Copy or clone frontend code into `/var/opt/checq-ai/dashboard/`.
2. **Configure Environment (`/var/opt/checq-ai/dashboard/.env`)**:
   ```env
   BACKEND_URL=https://checq-ai.qforte.net/api
   ALLOWED_IPS=0.0.0.0/0
   ```
3. **Build and Launch Service**:
   ```bash
   cd /var/opt/checq-ai/dashboard/
   npm install
   npm run build

   # Start frontend app with PM2 (Listening on Port 3000)
   pm2 start -n checq-ai-dashboard server/index.mjs

   # Save process list and generate systemd startup hook
   pm2 save
   pm2 startup systemd
   # (Execute the 'sudo env PATH=...' command output by pm2 startup)
   ```

---

### B. ChecQ-AI Server (Backend Node.js API)

1. **Deploy Source Code**: Copy or clone backend code into `/var/opt/checq-ai/server/`.
2. **Configure Environment (`/var/opt/checq-ai/server/.env`)**:
   ```env
   CHECQ_DB_HOST=127.0.0.1
   CHECQ_DB_PORT=5432
   CHECQ_DB_NAME=checq_ai
   CHECQ_DB_USER=checq_ai
   CHECQ_DB_PASSWORD=PassDB@123
   CHECQ_DB_SSL=false

   CHECQ_S3_ENDPOINT=http://127.0.0.1:8333
   CHECQ_S3_REGION=us-east-1
   CHECQ_S3_ACCESS_KEY_ID=your_access_key
   CHECQ_S3_SECRET_ACCESS_KEY=your_secret_key
   CHECQ_S3_BUCKET=checq
   CHECQ_S3_FORCE_PATH_STYLE=true

   CHECQ_ENCRYPTION_KEY=your_secure_encryption_key
   ```
3. **Configure Systemd Service**:
   Create `/etc/systemd/system/checq-ai-server.service`:
   ```ini
   [Unit]
   Description=ChecQ AI API Server
   After=network.target postgresql.service

   [Service]
   Type=simple
   User=root
   WorkingDirectory=/var/opt/checq-ai/server
   Environment=NODE_ENV=production
   Environment=PORT=9000
   Environment=HOST=0.0.0.0
   ExecStart=/usr/bin/node /var/opt/checq-ai/server/bin/checq.js serve --port 9000
   Restart=always
   RestartSec=5
   StandardOutput=journal
   StandardError=journal

   [Install]
   WantedBy=multi-user.target
   ```
4. **Enable & Start Backend Service**:
   ```bash
   sudo systemctl daemon-reload
   sudo systemctl start checq-ai-server.service
   sudo systemctl enable checq-ai-server.service

   # Verify ports 3000 (Frontend) and 9000 (Backend) are listening
   sudo netstat -tulpn | grep -E '3000|9000'
   ```

---

### C. Nginx Reverse Proxy & SSL Setup

1. **Create Nginx Configuration**:
   Create `/etc/nginx/sites-available/checq-ai.conf`:
   ```nginx
   server {
       listen 80;
       server_name checq-ai.qforte.net; # Replace with your domain / Elastic IP

       # Frontend Application
       location / {
           proxy_pass http://127.0.0.1:3000;
           proxy_http_version 1.1;
           proxy_set_header Upgrade $http_upgrade;
           proxy_set_header Connection 'upgrade';
           proxy_set_header Host $host;
           proxy_cache_bypass $http_upgrade;
       }

       # Backend API
       location /api {
           proxy_pass http://127.0.0.1:9000;
           proxy_http_version 1.1;
           proxy_set_header X-Real-IP $remote_addr;
           proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
           proxy_set_header Host $host;
       }
   }
   ```
2. **Enable Site & Install Certbot**:
   ```bash
   # Enable Nginx site configuration
   sudo ln -s /etc/nginx/sites-available/checq-ai.conf /etc/nginx/sites-enabled/
   sudo nginx -t && sudo systemctl reload nginx

   # Install Certbot and SSL certificate
   sudo apt update && sudo apt install certbot python3-certbot-nginx -y
   sudo certbot --nginx -d checq-ai.qforte.net
   ```

---

### D. NMD (Network Monitoring Service / Suricata)

1. **Install Suricata & Go Language**:
   ```bash
   # Install Suricata IDS
   sudo add-apt-repository ppa:oisf/suricata-stable -y
   sudo apt update && sudo apt install suricata golang-go -y

   # Detect active interface (e.g., eth0 or enX0)
   ip a

   # Update default interface in Suricata configuration
   sudo sed -i 's/eth0/enp0s31f6/g' /etc/suricata/suricata.yaml

   # Initialize Suricata rules and start service
   sudo suricata-update
   sudo systemctl enable --now suricata

   # Fix permissions for logging
   sudo chmod 755 /var/log/suricata
   sudo chmod 644 /var/log/suricata/eve.json
   ```

2. **Deploy NMD Source Code**:
   Copy NMD service files to `/var/opt/checq-ai/nmd/`.

3. **Configure NMD Systemd Service**:
   Create `/etc/systemd/system/checq-ai-network-monitor.service`:
   ```ini
   [Unit]
   Description=Checq-AI Network Monitoring Node
   After=network-online.target
   Wants=network-online.target

   [Service]
   Type=simple
   User=root
   WorkingDirectory=/var/opt/checq-ai/nmd
   Environment="PATH=/usr/local/go/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
   Environment="HOME=/root"
   StandardInput=data
   StandardInputText=1
   ExecStart=/bin/bash /var/opt/checq-ai/nmd/run.sh
   Restart=on-failure
   RestartSec=5
   KillMode=control-group
   TimeoutStopSec=30
   StandardOutput=journal
   StandardError=journal

   [Install]
   WantedBy=multi-user.target
   ```

4. **Enable & Start NMD Service**:
   ```bash
   sudo systemctl daemon-reload
   sudo systemctl start checq-ai-network-monitor.service
   sudo systemctl enable checq-ai-network-monitor.service

   # Verify NMD listening port (8090)
   sudo netstat -tulpn | grep 8090
   ```

---

## 5. UI Post-Deployment Configuration

1. Log in to the **ChecQ-AI Dashboard**.
2. Navigate to: **Settings** $\rightarrow$ **Configuration** $\rightarrow$ **Network Monitoring URL**.
3. Update the URL value to: `http://<YOUR_SERVER_PRIVATE_OR_PUBLIC_IP>:8090` (e.g., `http://172.31.6.224:8090`).
4. Click **Save Configuration** and navigate to the **Network Monitoring Section** to verify real-time dashboard data flows.
