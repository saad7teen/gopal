# End-to-End Infrastructure, Setup & Deployment Guide
## Nexus Global (Vinmar) — React, Node.js/TypeScript, MongoDB Atlas & Disaster Recovery

This document is a complete, phased step-by-step implementation guide:
1. **Phase 1:** Launch and configure AWS EC2 & MongoDB Atlas.
2. **Phase 2:** Host and test on **plain HTTP via Elastic IP** using NGINX (Verify everything works first).
3. **Phase 3:** Configure **GoDaddy DNS** `A Record` to point to the Elastic IP.
4. **Phase 4:** Run **Certbot for free SSL** and switch to **HTTPS**.
5. **Phase 5:** Set up automated **GitHub Actions CI/CD** pipelines.
6. **Phase 6:** Automated daily database backups to S3.
7. **Phase 7:** Disaster Recovery (DR) Setup (S3 Cross-Region Replication & Database DR).

---

> [!NOTE]
> **Optional Staging Environment:**
> Staging is **completely optional**. If your project only needs a **Production** environment right now, you can completely ignore all staging steps (staging directories, staging NGINX blocks, staging git branches, and staging database). Only set up the **Production** configurations.

---

## 1. Phase 1: AWS Infrastructure & MongoDB Atlas Setup

### 1.1 Launch the Primary EC2 Instance (Mumbai)
1. Go to **AWS Console $\rightarrow$ EC2 $\rightarrow$ Launch Instance**.
2. **Name:** `nexus-global-production`
3. **AMI:** `Ubuntu Server 24.04 LTS` (or `22.04 LTS`), 64-bit (x86).
4. **Instance Type:** `t3.small` (2 vCPU, 2 GB RAM).
5. **Key Pair:** Create or select your key pair (`nexus-key.pem`).
6. **Storage:** `30 GB` gp3 SSD.
7. **Security Group (`nexus-sg`):**
   * **SSH (Port 22):** Your Office IP / Admin IPs.
   * **HTTP (Port 80):** `0.0.0.0/0` (Needed for initial HTTP testing and Certbot validation).
   * **HTTPS (Port 443):** `0.0.0.0/0`.
8. **Detailed Monitoring:** Leave **DISABLED** (Standard monitoring is 100% free; detailed monitoring costs extra).
9. Click **Launch Instance**.

### 1.2 Allocate & Attach an Elastic IP
1. Go to **EC2 $\rightarrow$ Network & Security $\rightarrow$ Elastic IPs**.
2. Click **Allocate Elastic IP address** (Region: `ap-south-1` Mumbai).
3. Select the IP $\rightarrow$ **Actions $\rightarrow$ Associate Elastic IP address**.
4. Choose the instance `nexus-global-production`. Note down your `<ELASTIC_IP>`.

### 1.3 Enable EC2 Auto-Recovery (100% Free — $0.00)
> [!TIP]
> **Does this alarm cause extra costs? NO.**
> AWS CloudWatch includes **10 free alarm metrics** in the Free Tier forever. Furthermore, the metric `StatusCheckFailed_System` is a standard built-in EC2 metric provided at **no extra charge**.
1. Select the instance $\rightarrow$ **Actions $\rightarrow$ Monitor and troubleshoot $\rightarrow$ Manage CloudWatch alarms**.
2. Create an alarm:
   * **Alarm action:** `Recover this instance`.
   * **Metric:** `StatusCheckFailed_System` >= 1 for 2 consecutive minutes.
3. This restarts your instance automatically on healthy hardware in Mumbai if the underlying physical host ever crashes.

---

### 1.4 Setup MongoDB Atlas (M10 in Mumbai)
1. In [cloud.mongodb.com](https://cloud.mongodb.com), create a cluster:
   * **Tier:** `M10` (Dedicated).
   * **Provider & Region:** `AWS` $\rightarrow$ `ap-south-1` (Mumbai).
   * **Cluster Name:** `nexus-cluster`.
2. **Database Access:** Create user `nexus_app_user` with a strong password.
3. **Network Access:** Whitelist your EC2 **Elastic IP address**.
4. **Connection String:** Copy the URI:
   ```text
   mongodb+srv://nexus_app_user:<PASSWORD>@nexus-cluster.xxxxx.mongodb.net/vinmar_prod?retryWrites=true&w=majority
   ```

---

## 2. Phase 2: Host & Test on Plain HTTP via Elastic IP

In this phase, we set up NGINX to serve the React frontend and Node.js backend over **plain HTTP using the Elastic IP** so you can smoke test the entire application before touching DNS or SSL.

### 2.1 Server Initialization
SSH into your server:
```bash
ssh -i nexus-key.pem ubuntu@<ELASTIC_IP>
```

Run package installations:
```bash
# Update OS
sudo apt update && sudo apt upgrade -y

# Install essential tools & NGINX
sudo apt install -y curl git unzip ufw nginx

# Install Node.js 22 LTS
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt install -y nodejs

# Install PM2 globally
sudo npm install -g pm2
pm2 startup systemd -u ubuntu --hp /home/ubuntu
# (Run the 'sudo env PATH=...' command that PM2 prints)

# Create application directories
mkdir -p /home/ubuntu/backend
sudo mkdir -p /var/www/html
sudo chown -R ubuntu:ubuntu /var/www/html
```

---

### 2.2 Configure Initial HTTP-Only NGINX Site
Create the NGINX site configuration:
```bash
sudo nano /etc/nginx/sites-available/nexus.conf
```

Paste the **HTTP-only configuration** (listens on Port 80 for any IP or host):

```nginx
server {
    listen 80 default_server;
    listen [::]:80 default_server;
    server_name _;

    client_max_body_size 25M;

    # 1. Frontend: Serve React static files from /var/www/html
    root /var/www/html;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }

    # 2. Backend: Reverse Proxy /server to Node.js running on Port 3002
    location /server {
        proxy_pass http://127.0.0.1:3002;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_read_timeout 120s;
    }
}
```

Enable the configuration:
```bash
sudo ln -sf /etc/nginx/sites-available/nexus.conf /etc/nginx/sites-enabled/
sudo rm -f /etc/nginx/sites-enabled/default
sudo nginx -t
sudo systemctl reload nginx
```

---

### 2.3 Create Initial HTTP `.env` File
Create `/home/ubuntu/backend/.env`:
```bash
nano /home/ubuntu/backend/.env
```

Paste the initial settings (Notice `COOKIE_SECURE=false` while testing on HTTP):
```env
# Server Configuration
NODE_ENV=production
PORT=3002
APP_NAME=nexus-backend
APP_TYPE=PRODUCTION
SERVER_URL=http://<ELASTIC_IP>
FRONTEND_URL=http://<ELASTIC_IP>
CORS_ORIGINS=http://<ELASTIC_IP>

# Database (MongoDB Atlas)
MONGO_URI=mongodb+srv://nexus_app_user:<PASSWORD>@nexus-cluster.xxxxx.mongodb.net/vinmar_prod?retryWrites=true&w=majority
MONGO_PROD_DATABASE=vinmar_prod

# Authentication & Security
SESSION_SECRET=generate_a_random_64_char_secret_here
PASSWORD_ENC_SECRET=generate_a_random_32_char_secret_here
PASSWORD_SALT_ROUNDS=12
COOKIE_SECURE=false

# AWS S3 Storage
AWS_REGION=ap-south-1
S3_BUCKET=vinmar-tegral-bucket

# Mail & Integrations
TENANT_ID=your_azure_tenant_id
CLIENT_ID=your_azure_client_id
CLIENT_SECRET=your_azure_client_secret
OUTLOOK_SENDER_EMAIL=notifications@example.com

# SAP Integrations
ERP_IMPORT_KEY=your_erp_secret_key
SAP_SFTP_HOST=sftp.example.com
SAP_SFTP_PORT=22
SAP_SFTP_USER=sftp_user
SAP_SFTP_PASS=sftp_password
```

Lock permissions:
```bash
chmod 600 /home/ubuntu/backend/.env
```

---

### 2.4 Deploy Code for Initial HTTP Testing

1. **Deploy Backend:**
   * Build the backend: `npm run build`.
   * Copy `build/`, `package.json`, and `package-lock.json` to `/home/ubuntu/backend/`.
   * On the server, run:
     ```bash
     cd /home/ubuntu/backend
     npm install --omit=dev
     pm2 start build/server.js --name nexus-backend
     pm2 save
     ```

2. **Deploy Frontend:**
   * Build React: `npm run build`.
   * Copy the contents of the `build/` folder into `/var/www/html/`.

---

### 2.5 Verify on HTTP (Smoke Test)
From your laptop browser and terminal:
1. **API Health Check:**
   ```bash
   curl -s http://<ELASTIC_IP>/server/health
   # Expected: {"status":"ok","database":"connected"}
   ```
2. **Frontend Test:** Open `http://<ELASTIC_IP>` in your browser. Verify the login page opens and you can log in.

---

## 3. Phase 3: GoDaddy DNS Configuration

Now that the app is verified and working over HTTP, connect your custom domain name.

### 3.1 What to do in GoDaddy:
1. Log in to your **[GoDaddy Account](https://dcc.godaddy.com/)**.
2. Go to **My Products** $\rightarrow$ Scroll to **Domains** $\rightarrow$ Click **Manage DNS** (or **DNS**) next to your domain.
3. In the **DNS Records** table, add an **`A Record`**:

| Type | Name / Host | Value / Points to | TTL | Purpose |
|---|---|---|---|---|
| **A** | `app` | `<YOUR_EC2_ELASTIC_IP>` | `1/2 Hour` (or 600s) | URL: `http://app.example.com` |
| **A** *(Optional)* | `@` | `<YOUR_EC2_ELASTIC_IP>` | `1/2 Hour` | Use `@` if pointing root domain `example.com` |

> [!WARNING]
> **Do NOT delete or edit existing MX or TXT (SPF/DKIM) records.** Those manage your company emails. Only add or edit the `A Record` for your application hostname.

### 3.2 Verify DNS Propagation
Wait 5 to 10 minutes, then check from your terminal:
```bash
nslookup app.example.com
```
When it returns your AWS Elastic IP, DNS is ready!

---

## 4. Phase 4: Configure Certbot SSL & Switch to HTTPS

### 4.1 Install Certbot & Obtain Free SSL Certificate
On your EC2 server, run:
```bash
sudo apt install -y certbot python3-certbot-nginx

# Request the SSL certificate
sudo certbot --nginx -d app.example.com
```

* Enter your admin email when prompted.
* Agree to the terms of service.
* **Certbot will automatically:**
  1. Validate your domain via NGINX.
  2. Issue the SSL certificate.
  3. **Automatically update `/etc/nginx/sites-available/nexus.conf`** with SSL certificate paths and automatic HTTP $\rightarrow$ HTTPS redirection!

---

### 4.2 Update `.env` to Production HTTPS
Now update `/home/ubuntu/backend/.env` with your secure domain and enable HTTPS cookies:

```bash
nano /home/ubuntu/backend/.env
```

Update these 4 variables:
```env
SERVER_URL=https://app.example.com
FRONTEND_URL=https://app.example.com
CORS_ORIGINS=https://app.example.com
COOKIE_SECURE=true
```

Restart the backend to load the new environment variables:
```bash
pm2 restart nexus-backend --update-env
pm2 save
```

---

### 4.3 Verify HTTPS & Auto-Renewal
1. **Test HTTPS Redirect:**
   ```bash
   curl -I http://app.example.com
   # Expected: HTTP/1.1 301 Moved Permanently -> Location: https://app.example.com/
   ```
2. **Test Secure API:**
   ```bash
   curl -s https://app.example.com/server/health
   # Expected: {"status":"ok","database":"connected"}
   ```
3. **Verify SSL Auto-Renewal Timer:**
   ```bash
   sudo systemctl status certbot.timer
   # Expected: active (waiting) — Certbot auto-renews certificates before expiry!
   ```

---

## 5. Phase 5: GitHub Actions Automated CI/CD Setup

### 5.1 Add GitHub Secrets
In both repositories (Backend & Frontend), go to **Settings $\rightarrow$ Secrets and variables $\rightarrow$ Actions** and add:

| Secret Name | Value Description |
|---|---|
| `EC2_HOST` | Your EC2 Elastic IP address (e.g., `3.111.131.220`) |
| `EC2_USERNAME` | `ubuntu` |
| `EC2_SSH_KEY` | Entire content of your private key file (`nexus-key.pem`) |

---

### 5.2 Backend CI/CD Pipeline (`.github/workflows/deploy-backend.yml`)
Place this in your **backend** repository:

```yaml
name: Deploy Backend API

on:
  push:
    branches:
      - main       # Deploys to Production
      # - staging  # (Uncomment only if staging is used)

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
      - name: 📥 Check out repository code
        uses: actions/checkout@v4

      - name: ⚙️ Set up Node.js 22
        uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: 'npm'

      - name: 📦 Install dependencies & compile TypeScript
        run: |
          npm ci
          npm run build

      - name: 🗜️ Package build artifact
        run: |
          mkdir -p deploy
          cp -r build deploy/
          cp package.json package-lock.json deploy/
          cd deploy && zip -r ../backend-artifact.zip . && cd ..

      - name: 🏷️ Set target deployment variables
        id: env-target
        run: |
          if [ "${{ github.ref_name }}" = "main" ]; then
            echo "app_dir=/home/ubuntu/backend" >> $GITHUB_OUTPUT
            echo "pm2_name=nexus-backend" >> $GITHUB_OUTPUT
          else
            echo "app_dir=/home/ubuntu/staging-backend" >> $GITHUB_OUTPUT
            echo "pm2_name=nexus-staging-backend" >> $GITHUB_OUTPUT
          fi

      - name: 🚀 Copy artifact to EC2 via SCP
        uses: appleboy/scp-action@v0.1.7
        with:
          host: ${{ secrets.EC2_HOST }}
          username: ${{ secrets.EC2_USERNAME }}
          key: ${{ secrets.EC2_SSH_KEY }}
          source: 'backend-artifact.zip'
          target: ${{ steps.env-target.outputs.app_dir }}

      - name: 🔄 Deploy, Install Dependencies & Restart PM2
        uses: appleboy/ssh-action@v1.0.3
        with:
          host: ${{ secrets.EC2_HOST }}
          username: ${{ secrets.EC2_USERNAME }}
          key: ${{ secrets.EC2_SSH_KEY }}
          script: |
            TARGET_DIR="${{ steps.env-target.outputs.app_dir }}"
            APP_NAME="${{ steps.env-target.outputs.pm2_name }}"
            
            cd $TARGET_DIR
            unzip -o backend-artifact.zip
            rm backend-artifact.zip
            npm install --omit=dev
            
            # Zero-downtime PM2 reload
            pm2 reload $APP_NAME || pm2 start build/server.js --name $APP_NAME
            pm2 save
```

---

### 5.3 Frontend CI/CD Pipeline (`.github/workflows/deploy-frontend.yml`)
Place this in your **frontend** repository:

```yaml
name: Deploy Frontend (React)

on:
  push:
    branches:
      - main       # Deploys to Production
      # - staging  # (Uncomment only if staging is used)

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
      - name: 📥 Check out repository code
        uses: actions/checkout@v4

      - name: ⚙️ Set up Node.js 22
        uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: 'npm'

      - name: 🏷️ Set target web directory
        id: env-target
        run: |
          if [ "${{ github.ref_name }}" = "main" ]; then
            echo "web_root=/var/www/html" >> $GITHUB_OUTPUT
          else
            echo "web_root=/var/www/staging" >> $GITHUB_OUTPUT
          fi

      - name: 📦 Install dependencies & build static bundle
        run: |
          npm ci
          npm run build

      - name: 🚀 Deploy static build to EC2 via SCP
        uses: appleboy/scp-action@v0.1.7
        with:
          host: ${{ secrets.EC2_HOST }}
          username: ${{ secrets.EC2_USERNAME }}
          key: ${{ secrets.EC2_SSH_KEY }}
          source: 'build/*'
          target: ${{ steps.env-target.outputs.web_root }}
          strip_components: 1

      - name: 🧹 Clean up permissions on server
        uses: appleboy/ssh-action@v1.0.3
        with:
          host: ${{ secrets.EC2_HOST }}
          username: ${{ secrets.EC2_USERNAME }}
          key: ${{ secrets.EC2_SSH_KEY }}
          script: |
            sudo chown -R ubuntu:ubuntu ${{ steps.env-target.outputs.web_root }}
            sudo chmod -R 755 ${{ steps.env-target.outputs.web_root }}
```

---

## 6. Phase 6: Automated Daily Database Backups (Local S3)

On the EC2 server, create `/home/ubuntu/backup_mongo.sh`:
```bash
nano /home/ubuntu/backup_mongo.sh
```

Paste:
```bash
#!/bin/bash
DATE=$(date +%Y-%m-%d_%H%M%S)
BACKUP_DIR="/home/ubuntu/db_backups"
BUCKET_NAME="vinmar-tegral-bucket"
MONGO_URI="mongodb+srv://nexus_app_user:<PASSWORD>@nexus-cluster.xxxxx.mongodb.net/vinmar_prod"

mkdir -p $BACKUP_DIR

# Dump database to compressed archive
mongodump --uri "$MONGO_URI" --archive="$BACKUP_DIR/backup-$DATE.gz" --gzip

# Upload to AWS S3 (Requires EC2 IAM role with S3 write access)
aws s3 cp "$BACKUP_DIR/backup-$DATE.gz" "s3://$BUCKET_NAME/backups/backup-$DATE.gz"

# Keep only the last 3 days of local backup files to save disk space
find $BACKUP_DIR -type f -name "*.gz" -mtime +3 -delete
```

Make it executable and add to crontab:
```bash
chmod +x /home/ubuntu/backup_mongo.sh

# Open crontab
crontab -e

# Add this line to run every night at 02:00 AM:
0 2 * * * /home/ubuntu/backup_mongo.sh > /home/ubuntu/backup.log 2>&1
```

---

## 7. Phase 7: Disaster Recovery (DR) Setup (Secondary Region)

This section implements the **Disaster Recovery (DR)** architecture to protect against a catastrophic regional outage in Mumbai.

### 7.1 S3 Cross-Region Replication (CRR) to Glacier Instant Retrieval
This ensures every invoice, PO, and document uploaded to Mumbai is automatically replicated to the DR region (Hyderabad `ap-south-2` or Singapore) in **milliseconds** while cutting storage cost by **83%**.

1. **Create DR Bucket:**
   * Go to AWS S3 Console $\rightarrow$ Create bucket `vinmar-tegral-bucket-dr`.
   * **Region:** `ap-south-2` (Hyderabad).
   * **Versioning:** Enable (Replication requires versioning).
2. **Enable Versioning on Primary Bucket:**
   * On `vinmar-tegral-bucket` (Mumbai), ensure **Bucket Versioning** is enabled.
3. **Configure Replication Rule (Mumbai $\rightarrow$ DR):**
   * On `vinmar-tegral-bucket` $\rightarrow$ **Management** tab $\rightarrow$ **Replication rules** $\rightarrow$ **Create replication rule**.
   * Rule name: `replicate-to-dr`.
   * Destination: `vinmar-tegral-bucket-dr` in `ap-south-2`.
   * **Destination storage class:** Check the box **"Change the storage class for the replicated objects"** $\rightarrow$ Select **S3 Glacier Instant Retrieval** (~$0.004/GB vs $0.023/GB).
   * Delete marker replication: Keep **DISABLED** (Protects DR bucket if someone accidentally deletes a file in prod).
4. **Add Reverse Replication Rule (DR $\rightarrow$ Mumbai for Failback):**
   * Create an identical rule on the DR bucket pointing back to Mumbai.
   * *Cost:* **$0.00 while idle**; only copies files if the DR region is actively serving uploads during an emergency.

---

### 7.2 MongoDB Atlas Disaster Recovery Configuration
To prevent the 24-hour data loss of the proposed bash script:

#### Recommended Choice: Atlas Cloud Backup with Cross-Region Snapshot Copy
1. In MongoDB Atlas $\rightarrow$ Go to your **M10 Cluster** $\rightarrow$ **Backup** tab.
2. Ensure **Cloud Backup** is turned ON.
3. Under **Backup Schedule**, click **Edit Policy**:
   * Set snapshot frequency (e.g. every 2 or 4 hours).
   * Under **Cross-Region Snapshot Copy**, select your DR region (`AWS / ap-south-2` Hyderabad).
4. **Cost:** Only pay a few cents per GB of snapshot storage (~$2 to $4/month).
5. **How Failover Works:** If Mumbai suffers an outage, spin up or restore the snapshot into an M10 cluster in Hyderabad in 15 minutes.

*(Optional Alternative: If zero seconds of data loss is required by the client, configure the M10 as a **Multi-Region Cluster** with 2 nodes in Mumbai and 1 node in Hyderabad).*

---

### 7.3 Standby DR EC2 Instance (Pilot Light)
1. Launch an identical `t3.small` instance in **Hyderabad (`ap-south-2`)**:
   * Install Node.js 22, NGINX, and PM2.
   * Copy the production NGINX config.
   * Attach an Elastic IP in Hyderabad (so third-party SAP vendors can whitelist it in advance).
2. **Stop the instance:**
   * In AWS EC2 console, click **Instance State $\rightarrow$ Stop Instance**.
   * **Cost while stopped:** Compute is **$0.00/hour**. You only pay ~$3.20/mo for the EBS disk and ~$3.60/mo for the reserved IP.
3. If Mumbai ever goes down, simply start this instance, restore the DB snapshot, and update the GoDaddy `A Record` to the DR Elastic IP.

---

## 8. Execution Order Summary

1. **Step 1:** Launch EC2 `t3.small` in Mumbai + attach Elastic IP. Set CloudWatch Auto-Recovery (Free).
2. **Step 2:** Setup MongoDB Atlas M10 + whitelist the Elastic IP.
3. **Step 3:** SSH into EC2, install packages, configure **HTTP NGINX**, and create initial HTTP `.env` (`COOKIE_SECURE=false`).
4. **Step 4:** Deploy build and **smoke test on `http://<ELASTIC_IP>/`**.
5. **Step 5:** Add `A Record` in GoDaddy (`app` $\rightarrow$ Elastic IP). Verify with `nslookup`.
6. **Step 6:** Run `sudo certbot --nginx -d app.example.com`.
7. **Step 7:** Update `.env` with `https://app.example.com` and `COOKIE_SECURE=true`, then `pm2 restart nexus-backend --update-env`.
8. **Step 8:** Configure GitHub Secrets and commit workflows for automatic future deployments.
9. **Step 9:** Configure S3 Cross-Region Replication to Glacier Instant Retrieval and Atlas backup for Disaster Recovery.
