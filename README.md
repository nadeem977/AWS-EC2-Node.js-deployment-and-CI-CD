# 🚀 Self-Hosted CI/CD with AWS EC2, GitHub Actions & Nginx

A step-by-step guide to deploy your Node.js app using **AWS EC2**, **GitHub Actions** (self-hosted runner), **PM2**, and **Nginx** with **SSL**.

---

## 📋 Table of Contents

1. [Prerequisites](#-prerequisites)
2. [Part 1: Create AWS EC2 Instance](#-part-1-create-aws-ec2-instance)
3. [Part 2: GitHub Actions Self-Hosted Runner](#-part-2-github-actions-self-hosted-runner)
4. [Part 3: Repository Secrets](#-part-3-repository-secrets)
5. [Part 4: CI/CD Workflow](#-part-4-cicd-workflow)
6. [Part 5: Node.js & App on EC2](#-part-5-nodejs--app-on-ec2)
7. [Part 6: PM2 Process Manager](#-part-6-pm2-process-manager)
8. [Part 7: Nginx Configuration](#-part-7-nginx-configuration)
9. [Part 8: SSL with Let's Encrypt](#-part-8-ssl-with-lets-encrypt)
10. [Verify Everything](#-verify-everything)

---

## ✅ Prerequisites

- **AWS account** with EC2 access
- **GitHub repository** with your Node.js project
- **Domain name** (optional but required for SSL)
- **SSH client** to connect to EC2

---

## 🖥️ Part 1: Create AWS EC2 Instance

1. Log in to **AWS Console** → **EC2** → **Launch Instance**.
2. Choose an **Ubuntu** AMI (e.g. Ubuntu 22.04 LTS).
3. Select instance type (e.g. **t2.micro** for free tier).
4. Create or select a **key pair** (.pem) and download it — you need this to SSH.
5. Configure **Security Group**:
   - **SSH (22)** — Your IP (for SSH).
   - **HTTP (80)** — 0.0.0.0/0 (for Nginx).
   - **HTTPS (443)** — 0.0.0.0/0 (for SSL).
   - **Custom TCP** — Port your app uses (e.g. **8001**) — 0.0.0.0/0 or localhost only as needed.
6. Launch the instance and note the **Public IP**.

**Connect via SSH:**
```bash
ssh -i "your-key.pem" ubuntu@<EC2-PUBLIC-IP>
```

---

## 🤖 Part 2: GitHub Actions Self-Hosted Runner

1. On **GitHub** go to your repo → **Settings** → **Actions** → **Runners**.
2. Click **New self-hosted runner**.
3. Select **Linux** and **x64** (for Ubuntu EC2).
4. On your **EC2 instance** (via SSH), run the **Download** commands shown by GitHub, e.g.:

   ```bash
   # Example (use exact commands from GitHub):
   mkdir actions-runner && cd actions-runner
   curl -o actions-runner-linux-x64-2.xxx.x.tar.gz -L https://github.com/actions/runner/releases/download/v2.xxx.x/actions-runner-linux-x64-2.xxx.x.tar.gz
   tar xzf ./actions-runner-linux-x64-2.xxx.x.tar.gz
   ```

5. Run the **Configure** command and follow prompts (you can press **Enter** for defaults):

   ```bash
   ./config.sh --url https://github.com/YOUR_USERNAME/YOUR_REPO --token YOUR_TOKEN
   ```

6. **Do not** run the "Run" or "Service" commands from GitHub yet — we'll use `svc.sh` next.
7. Install and start the runner as a service:

   ```bash
   sudo ./svc.sh install
   sudo ./svc.sh start
   ```

8. On GitHub **Settings → Actions → Runners** you should see the runner as **Idle** (green).

---

## 🔐 Part 3: Repository Secrets

1. GitHub repo → **Settings** → **Secrets and variables** → **Actions**.
2. Click **New repository secret**.
3. Create a secret named **`PROD_ENV`**.
4. Paste the **entire contents** of your `.env` file (all key=value lines).
5. Save.  
   This way the workflow can write env vars into `.env` on the runner without committing secrets.

---

## 📜 Part 4: CI/CD Workflow

1. In your repo go to **Actions** → **New workflow** (or create a new file).
2. Search for **Node.js** and use **Set up this workflow** (or create the file manually).
3. Replace the contents with the workflow below.  
   Path: **`.github/workflows/node.js.yml`** (or name it e.g. `vite-cicd.yml`).

**File: `.github/workflows/node.js.yml`**

```yaml
name: Vite CI/CD

on:
  push:
    branches: ["main"]

jobs:
  build-and-deploy:
    runs-on: self-hosted

    strategy:
      matrix:
        node-version: [22.x]

    steps:
      - name: Checkout Repository
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
          cache: 'npm'

      - name: Install Dependencies
        run: npm install

      - name: Create .env file
        run: |
          touch .env
          echo "${{ secrets.PROD_ENV }}" >> .env
```

4. Commit and push to **main**. The workflow will run on your self-hosted runner.
5. Open **Actions** tab and confirm the run succeeds — **your CI/CD is done**.

---

## 📦 Part 5: Node.js & App on EC2

Run these on your **EC2 instance** (SSH).

**Install Node.js 18 (via NodeSource):**
```bash
curl -sL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt install nodejs
node --version
```

**Clone your app (example):**
```bash
git clone https://github.com/piyushgargdev-01/short-url-nodejs
cd short-url-nodejs
```

*(Replace with your own repo URL and directory.)*

---

## ⚡ Part 6: PM2 Process Manager

**Install PM2 globally:**
```bash
sudo npm i pm2 -g
```

**Start your app (adjust entry file if needed):**
```bash
pm2 start index.js
# or: pm2 start index
# or: pm2 start npm --name "app" -- start
```

**Useful PM2 commands:**
| Command | Description |
|--------|-------------|
| `pm2 show app` | Show app details |
| `pm2 status` | List all processes |
| `pm2 restart app` | Restart app |
| `pm2 stop app` | Stop app |
| `pm2 logs` | Live log stream |
| `pm2 flush` | Clear logs |

**Start app on system reboot:**
```bash
pm2 startup ubuntu
# Run the command it prints (sudo env PATH=...)
pm2 save
```

---

## 🌐 Part 7: Nginx Configuration

**Install Nginx:**
```bash
sudo apt install nginx
```

**Edit default site config:**
```bash
sudo nano /etc/nginx/sites-available/default
```

**Set `server_name` and add `location /` block** (replace `yourdomain.com` and port if needed):

```nginx
server_name yourdomain.com www.yourdomain.com;

location / {
    proxy_pass http://localhost:8001;   # Port your app runs on
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection 'upgrade';
    proxy_set_header Host $host;
    proxy_cache_bypass $http_upgrade;
}
```

**Check config and reload:**
```bash
sudo nginx -t
sudo nginx -s reload
```

---

## 🔒 Part 8: SSL with Let's Encrypt

*(Fix: use `sudo`, not `zsudo`.)*

**Install Certbot:**
```bash
sudo add-apt-repository ppa:certbot/certbot
sudo apt-get update
sudo apt-get install python3-certbot-nginx
```

**Get certificate (replace with your domain):**
```bash
sudo certbot --nginx -d yourdomain.com -d www.yourdomain.com
```

**Test renewal (certificates are valid 90 days):**
```bash
sudo certbot renew --dry-run
```

---

## ✅ Verify Everything

| Step | What to check |
|------|----------------|
| **EC2** | SSH works, app and Nginx are installed |
| **Runner** | GitHub → Settings → Actions → Runners shows runner **Idle** |
| **Secrets** | `PROD_ENV` exists under Settings → Secrets and variables → Actions |
| **Workflow** | Push to `main` → Actions tab shows successful run |
| **PM2** | `pm2 status` shows app **online** |
| **Nginx** | `curl http://localhost` and `http://YOUR_DOMAIN` return your app |
| **SSL** | `https://yourdomain.com` loads with padlock |

---

## 📁 Project Layout (Reference)

```
your-repo/
├── .github/
│   └── workflows/
│       └── node.js.yml    # CI/CD workflow
├── .env                   # Created by workflow from PROD_ENV (do not commit)
├── index.js               # Or your app entry
├── package.json
└── README.md
```

---

## 🆘 Quick Troubleshooting

- **Runner offline:** On EC2 run `sudo ./svc.sh start` and check `sudo ./svc.sh status`.
- **Workflow fails on “Create .env”:** Ensure `PROD_ENV` secret is set and contains valid env content.
- **502 Bad Gateway:** App not running or wrong port — check `pm2 status` and `proxy_pass` port in Nginx.
- **Nginx config error:** Run `sudo nginx -t` and fix syntax, then `sudo nginx -s reload`.

---

*Guide: Self-hosted CI/CD with AWS EC2, GitHub Actions, PM2, Nginx & Let's Encrypt.*
