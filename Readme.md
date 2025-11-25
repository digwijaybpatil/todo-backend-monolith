# 🚀 Deploying Python FastAPI Application on Azure VM (Ubuntu 20.04)

This guide explains how to deploy and run a **Python FastAPI backend** on an **Azure Ubuntu 20.04 VM**, using **pyodbc + Microsoft SQL Server**, **systemd**, and **GitHub Actions CI/CD**.

---

# 📌 Prerequisites

Ensure your VM was created using:

```hcl
source_image_reference = {
  publisher = "Canonical"
  offer     = "0001-com-ubuntu-server-focal"
  sku       = "20_04-lts"
  version   = "latest"
}
```

Your VM must have:

- Python 3.8+
- pip
- curl
- unixODBC
- msodbcsql18 (SQL Server ODBC Driver)
- GitHub Actions SSH key access

---

# 🔧 Step 1 — Connect to VM

```bash
ssh -i ~/.ssh/vm-ssh-private-key adminuser@<VM_PUBLIC_IP>
```

---

# 🔧 Step 2 — Install Required System Packages

```bash
sudo apt update
sudo apt install -y curl gnupg2 unixodbc unixodbc-dev
```

---

# 🔧 Step 3 — Install Microsoft SQL Server ODBC Driver (msodbcsql18)

```bash

sudo su

apt-get update
apt-get install -y curl gnupg2

curl https://packages.microsoft.com/keys/microsoft.asc | apt-key add -

curl https://packages.microsoft.com/config/ubuntu/20.04/prod.list \
    > /etc/apt/sources.list.d/mssql-release.list

apt-get update
ACCEPT_EULA=Y apt-get install -y msodbcsql18 unixodbc unixodbc-dev


odbcinst -q -d #it should show [ODBC Driver 18 for SQL Server]

```

Verify installation:

```bash
odbcinst -q -d
```

You should see:

```
[ODBC Driver 18 for SQL Server]
```

---

# 🐍 Step 4 — Install Python & pip

```bash
sudo apt install -y python3 python3-pip
```

Verify:

```bash
python3 --version
pip3 --version
```

---

# 🗂 Step 5 — Create Backend Directory on VM

```bash
sudo mkdir -p /var/www/backendapp
sudo chmod -R 777 /var/www/backendapp
```

---

# 🔧 Step 6 — Setup systemd Service (Auto-Start Backend)

Create service:

```bash
sudo nano /etc/systemd/system/backend.service
```

Paste:

```ini
[Unit]
Description=FastAPI Backend Service
After=network.target

[Service]
User=adminuser
WorkingDirectory=/var/www/backendapp
ExecStart=/usr/bin/python3 /var/www/backendapp/app.py
Restart=always

[Install]
WantedBy=multi-user.target
```

Enable & start:

```bash
sudo systemctl daemon-reload
sudo systemctl enable backend
sudo systemctl restart backend
sudo systemctl status backend
```

---

# 🔄 Step 7 — GitHub Actions CI/CD Deployment

Create this file:

**`.github/workflows/deploy.yml`**

```yaml
name: Deploy Backend to Azure VM

on:
  push:
    branches: ["main"]

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v3

      - name: Zip project
        run: zip -r backend.zip . -x "*.git*"

      - name: Copy to VM
        uses: appleboy/scp-action@v0.1.7
        with:
          host: ${{ secrets.SSH_HOST }}
          username: ${{ secrets.SSH_USER }}
          key: ${{ secrets.SSH_KEY }}
          source: "backend.zip"
          target: "/var/www/backendapp/"

      - name: Run deployment commands on VM
        uses: appleboy/ssh-action@v0.1.7
        with:
          host: ${{ secrets.SSH_HOST }}
          username: ${{ secrets.SSH_USER }}
          key: ${{ secrets.SSH_KEY }}
          script: |
            cd /var/www/backendapp
            sudo apt update

            # Install unzip if missing
            if ! command -v unzip &> /dev/null
            then
              sudo apt install -y unzip
            fi

            unzip -o backend.zip
            pip3 install -r requirements.txt
            sudo systemctl restart backend
```

---

# 🔐 GitHub Secrets Required

Go to:

**GitHub → Repo → Settings → Secrets → Actions**

Add:

| Secret Name | Value |
|------------|--------|
| `SSH_HOST` | VM Public IP |
| `SSH_USER` | adminuser |
| `SSH_KEY`  | Private SSH key (~/.ssh/vm-ssh-private-key) |

---

# 🧪 Step 8 — Test Deployment

SSH into VM:

```bash
sudo systemctl status backend
sudo journalctl -u backend -n 50
```

Test API:

```bash
curl http://localhost:8000/api
```

Or from browser:

```
http://<VM_PUBLIC_IP>:8000
```

---

# 🔗 API Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/` | Create tables |
| GET | `/api/tasks` | List tasks |
| GET | `/api/tasks/{id}` | Get task |
| POST | `/api/tasks` | Create |
| PUT | `/api/tasks/{id}` | Update |
| DELETE | `/api/tasks/{id}` | Delete |

---

# ✅ Conclusion

You now have:

✔ FastAPI backend running on Azure VM  
✔ Connected to SQL Server using pyodbc  
✔ GitHub Actions CI/CD for auto deployment  
✔ systemd service for auto restart  

Feel free to extend the project, add monitoring, logging, or front-end integrations.
