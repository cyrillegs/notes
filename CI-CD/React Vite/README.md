# 🚀 CI/CD Deployment Guide for React + Vite Website (GitHub → VPS)

This README provides the complete instructions and explanations for deploying a **React + Vite** web application to a **VPS** (e.g., Contabo, DigitalOcean, Linode) using **GitHub Actions CI/CD** and **Nginx**.

---

# 📌 Table of Contents

1. [Overview](#overview)
2. [Server Setup (VPS)](#server-setup-vps)
3. [SSH Key Setup for CI/CD](#ssh-key-setup-for-cicd)
4. [GitHub Actions CI/CD Workflow](#github-actions-cicd-workflow)
5. [Vite Configuration](#vite-configuration)
6. [Deployment Process](#deployment-process)
7. [Troubleshooting](#troubleshooting)

---

# 1. ⭐ Overview

This CI/CD pipeline automatically deploys your Vite app whenever you push updates to your **main** branch.

### 🔄 Deployment Flow
1. Developer pushes to `main` branch.
2. GitHub Actions:
   - Installs Node.js
   - Installs dependencies
   - Builds Vite project → `/dist`
   - Uploads the build to the VPS
   - Restarts Nginx
3. Website is instantly updated.

---

# 2. 🖥 Server Setup (VPS)

## ✅ Install Nginx
```bash
sudo apt update
sudo apt install nginx -y
```
Nginx will serve your built frontend files.

---

## ✅ Create Web Directory
```bash
sudo mkdir -p /var/www/myapp
sudo chown -R $USER:$USER /var/www/myapp
```
This folder will store your deployed build files from GitHub Actions.

---

## ✅ Configure Nginx
Create a site config:
```bash
sudo nano /etc/nginx/sites-available/myapp
```
Paste:
```
server {
    listen 80;
    server_name YOUR_DOMAIN_OR_IP;

    root /var/www/myapp;
    index index.html;

    location / {
        try_files $uri /index.html;
    }
}
```
Enable and restart:
```bash
sudo ln -s /etc/nginx/sites-available/myapp /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx
```
`try_files $uri /index.html;` ensures React SPA routing works.

---

# 3. 🔐 SSH Key Setup for CI/CD

CI/CD requires GitHub to securely SSH into your VPS.

## ✅ Generate SSH Key on VPS
```bash
ssh-keygen -t ed25519 -C "github-actions"
```
Keep pressing Enter.

---

## ✅ Add Public Key to Authorized Keys
```bash
cat ~/.ssh/id_ed25519.pub >> ~/.ssh/authorized_keys
```
This allows GitHub Actions to log into your VPS.

---

## ✅ Add Secrets to GitHub
Go to:
**GitHub → Repository → Settings → Secrets → Actions**

Add the following:

| Secret Name        | Value                      |
|--------------------|----------------------------|
| `SSH_PRIVATE_KEY`  | Your private key from VPS  |
| `SERVER_HOST`      | VPS IP address             |
| `SERVER_USER`      | Usually `root`             |
| `SERVER_PATH`      | `/var/www/myapp`           |

These secrets are used by GitHub Actions to SSH and deploy.

---

# 4. ⚙️ GitHub Actions CI/CD Workflow

Create file:
`/.github/workflows/deploy.yml`

```yaml
name: Deploy Vite App

on:
  push:
    branches: ["main"]

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Node
        uses: actions/setup-node@v4
        with:
          node-version: 20

      - name: Install dependencies
        run: npm install

      - name: Build project
        run: npm run build

      - name: Upload files to VPS
        uses: appleboy/scp-action@v0.1.7
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          source: "dist/*"
          target: "${{ secrets.SERVER_PATH }}"

      - name: Restart Nginx
        uses: appleboy/ssh-action@v1.0.0
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: |
            sudo systemctl restart nginx
```

### 🔍 Explanation of Each Step
- **Checkout code** → downloads your repository contents.
- **Setup Node** → installs Node.js so GitHub can build the project.
- **Install dependencies** → runs `npm install`.
- **Build project** → generates production files in `/dist`.
- **Upload files** → securely copies the build to your VPS.
- **Restart Nginx** → reloads your website with new files.

---

# 5. ⚠️ Vite Configuration
Ensure your `vite.config.js` contains:

```js
export default defineConfig({
  base: "/",
});
```
Without this, routing and asset paths may break when deployed.

---

# 6. 🚀 Deployment Process

After everything is set up:

👉 Just **push** to the `main` branch:
```bash
git add .
git commit -m "deployment update"
git push origin main
```
GitHub Actions will automatically:
- Build the project
- Upload it to the VPS
- Restart Nginx

Your updated site goes live instantly.

---

# 7. 🔧 Troubleshooting

### ❌ Permissions denied during upload
Fix:
```bash
sudo chown -R $USER:$USER /var/www/myapp
```

### ❌ GitHub cannot SSH
Ensure your private key in GitHub Secrets **matches** VPS key.

### ❌ Nginx shows old version
Restart:
```bash
sudo systemctl restart nginx
```

### ❌ Blank screen on refresh
Ensure Nginx config includes:
```
try_files $uri /index.html;
```

---

# ✅ Done!
Your React + Vite app is now fully automated with GitHub Actions and VPS deployment.
