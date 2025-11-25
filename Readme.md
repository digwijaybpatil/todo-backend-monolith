# 🚀 FastAPI Backend Deployment on Azure VM  
This guide explains how to deploy and run the Python (FastAPI) backend application on an Azure Virtual Machine using GitHub Actions and simple SSH-based deployment.

---

## 📌 Prerequisites
Before deployment, ensure the following are ready:

### **1. Azure VM (Ubuntu 20.04 or above)**
- The backend VM must be created and accessible via SSH.
- Public IP must be available for GitHub Actions to connect.

### **2. Python Installed**
Check version:

```bash
python3 --version
```

### **3. Install pip (if missing)**
If pip is missing, fix repository index and install:

```bash
sudo apt update
sudo apt install python3-pip -y
```

---

## 📁 Directory Setup on VM (One-Time Setup)

SSH into the VM:

```bash
sudo mkdir -p /var/www/backendapp
sudo chmod -R 777 /var/www/backendapp
```

---

## 🔧 Install SQL Server ODBC Driver (One-Time Setup)

FastAPI connects to Azure SQL using **pyodbc**, so install ODBC driver:

```bash
sudo apt update
sudo apt install -y curl gnupg2 unixodbc unixodbc-dev

curl -fsSL https://packages.microsoft.com/keys/microsoft.asc | \
    gpg --dearmor -o /usr/share/keyrings/microsoft.gpg

echo "deb [arch=amd64 signed-by=/usr/share/keyrings/microsoft.gpg] \
https://packages.microsoft.com/ubuntu/20.04/prod focal main" \
| sudo tee /etc/apt/sources.list.d/mssql-release.list

sudo apt update
sudo ACCEPT_EULA=Y sudo apt install -y msodbcsql18
```

---

## 🔐 Add SSH Keys to GitHub

In GitHub:

**Settings → Secrets and Variables → Actions → New Repository Secret**

Add:

| Secret Name | Value |
|------------|--------|
| `SSH_HOST` | `<Your-VM-IP>` |
| `SSH_USER` | `ubuntu` |
| `SSH_KEY`  | Private SSH Key from your VM |

---

## ⚙️ Create Systemd Service (Auto-start backend)

On the VM:

```bash
sudo nano /etc/systemd/system/backend.service
```

Paste:

```ini
[Unit]
Description=FastAPI Backend Service
After=network.target

[Service]
User=ubuntu
WorkingDirectory=/var/www/backendapp
ExecStart=/usr/bin/python3 /var/www/backendapp/app.py
Restart=always

[Install]
WantedBy=multi-user.target
```

Enable service:

```bash
sudo systemctl daemon-reload
sudo systemctl enable backend
```

---

## 📦 GitHub Actions Deployment Pipeline

Create file:

`.github/workflows/deploy.yml`

Paste this exact pipeline:

```yaml
name: Deploy Backend to Azure VM

on:
  push:
    branches:
      - main

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout Code
        uses: actions/checkout@v3

      - name: Zip Project
        run: zip -r app.zip . -x "*.git*"

      - name: Copy Files to Azure VM
        uses: appleboy/scp-action@v0.1.7
        with:
          host: ${{ secrets.SSH_HOST }}
          username: ${{ secrets.SSH_USER }}
          key: ${{ secrets.SSH_KEY }}
          port: 22
          source: "app.zip"
          target: "/var/www/backendapp/"

      - name: Run Deployment Script on VM
        uses: appleboy/ssh-action@v0.1.6
        with:
          host: ${{ secrets.SSH_HOST }}
          username: ${{ secrets.SSH_USER }}
          key: ${{ secrets.SSH_KEY }}
          port: 22
          script: |
            cd /var/www/backendapp
            unzip -o app.zip

            sudo apt update
            sudo apt install -y unixodbc unixodbc-dev curl gnupg2

            if ! dpkg -s msodbcsql18 > /dev/null 2>&1; then
              curl -fsSL https://packages.microsoft.com/keys/microsoft.asc | gpg --dearmor > microsoft.gpg
              sudo install -o root -g root -m 644 microsoft.gpg /usr/share/keyrings/
              echo "deb [signed-by=/usr/share/keyrings/microsoft.gpg] https://packages.microsoft.com/ubuntu/20.04/prod focal main" |
                sudo tee /etc/apt/sources.list.d/msprod.list
              sudo apt update
              sudo ACCEPT_EULA=Y apt install -y msodbcsql18
            fi

            pip install -r requirements.txt

            sudo systemctl restart backend
            sudo systemctl status backend --no-pager
```

---

## 🧪 Testing Backend After Deployment

Run:

```bash
curl http://<VM-PUBLIC-IP>:8000
```

Test DB connection endpoint:

```bash
curl http://<VM-PUBLIC-IP>:8000/api/tasks
```

If the connection string is correct → app will respond successfully.

---

## 🗄️ Updating SQL Connection String

Edit `app.py`:

```python
connection_string = (
    "Driver={ODBC Driver 18 for SQL Server};"
    "Server=tcp:<YOUR-SQL-SERVER>.database.windows.net,1433;"
    "Database=<DBNAME>;"
    "Uid=<USERNAME>;"
    "Pwd=<PASSWORD>;"
    "Encrypt=yes;"
    "TrustServerCertificate=no;"
    "Connection Timeout=30;"
)
```

---

## 🎉 Deployment Flow Summary

1. Push code to `main`  
2. GitHub Action zips project  
3. SCP uploads to Azure VM  
4. VM installs dependencies  
5. Systemd service restarts  
6. Latest version goes live instantly  

---

## 📞 Support

If deployment fails, check:

```bash
sudo journalctl -u backend -f
```

---

# ✅ Done — Your backend deployment documentation is complete and production-ready.
