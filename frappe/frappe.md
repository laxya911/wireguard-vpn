# Frappe Framework & ERPNext Installation Guide (Ubuntu / LXC)

This guide provides a **step-by-step process to install the Frappe Framework and ERPNext** on a fresh Ubuntu server or LXC container.

It covers:

* Ubuntu preparation
* Creating a non-root user
* Installing required dependencies
* Installing Frappe Bench
* Creating a new site
* Installing apps (ERPNext, HRMS, etc.)
* Running the system for development or production
* Initial ERPNext company setup

The steps below were tested on **Ubuntu 24.04 LTS** with **Python 3.12+** and **Frappe v15+**.

---

# 1. Install Ubuntu Server

Install **Ubuntu Server 24.04 LTS** (or latest LTS).

Update the system:

```bash
sudo apt update
sudo apt upgrade -y
```

Install basic tools:

```bash
sudo apt install -y \
git curl wget nano software-properties-common \
build-essential unzip
```

---

# 2. Create a Dedicated Frappe User

⚠️ **Do NOT run Bench as root**

Create a dedicated user:

```bash
sudo adduser frappeuser
sudo usermod -aG sudo frappeuser
```

Switch to the user:

```bash
su - frappeuser
```

All further commands should run as **frappeuser**.

---

# 3. Install System Dependencies

Install required packages:

```bash
sudo apt install -y \
python3-dev python3-pip python3-setuptools python3-venv \
mariadb-server mariadb-client \
redis-server \
libffi-dev libssl-dev \
wkhtmltopdf \
xfonts-75dpi xfonts-base
```

Install NodeJS and Yarn:

```bash
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs
sudo npm install -g yarn
```

Verify:

```bash
node -v
npm -v
yarn -v
```

---

# 4. Configure MariaDB

Start MariaDB:

```bash
sudo systemctl enable mariadb
sudo systemctl start mariadb
```

Secure installation:

```bash
sudo mysql_secure_installation
```

Login to MariaDB:

```bash
sudo mysql
```

Set recommended configuration:

```sql
SET GLOBAL innodb_file_per_table = ON;
SET GLOBAL innodb_file_format = 'Barracuda';
SET GLOBAL innodb_large_prefix = ON;
EXIT;
```

---

# 5. Install Redis

Install Redis:

```bash
sudo apt install redis-server
```

Start Redis:

```bash
sudo systemctl enable redis-server
sudo systemctl start redis-server
```

Test Redis:

```bash
redis-cli ping
```

Expected output:

```
PONG
```

---

# 6. Install Frappe Bench CLI

Install Bench using pip:

```bash
pip3 install --user frappe-bench
```

Add to PATH:

```bash
echo 'export PATH=$HOME/.local/bin:$PATH' >> ~/.bashrc
source ~/.bashrc
```

Verify installation:

```bash
bench --version
```

---

# 7. Initialize a Bench Environment

Create a workspace:

```bash
mkdir -p ~/frappe
cd ~/frappe
```

Initialize bench:

```bash
bench init my-bench --frappe-branch version-15
```

Directory structure:

```
frappe/
 └── my-bench/
     ├── apps
     ├── sites
     ├── env
     └── config
```

---

# 8. Create a New Site

Navigate into the bench directory:

```bash
cd ~/frappe/my-bench
```

Create a site:

```bash
bench new-site site1.local
```

You will be prompted for:

* MariaDB root password
* Administrator password

---

# 9. Install ERPNext

Download ERPNext:

```bash
bench get-app erpnext --branch version-15
```

Install on the site:

```bash
bench --site site1.local install-app erpnext
```

---

# 10. Start the Bench (Development Mode)

Start all services:

```bash
bench start
```

This launches:

* Redis Cache
* Redis Queue
* Web Server
* Workers
* Scheduler
* SocketIO

Access the system:

```
http://SERVER_IP:8000
```

---

# 11. ERPNext Initial Setup

Login using:

```
User: Administrator
Password: (the one you set)
```

The setup wizard will guide you through:

* Company name
* Country
* Currency
* Fiscal year
* Chart of accounts
* Users

After completion you will reach the **ERPNext Desk**.

---

# 12. Installing Additional Apps

Examples:

### HRMS

```bash
bench get-app hrms
bench --site site1.local install-app hrms
```

### Payments

```bash
bench get-app payments
bench --site site1.local install-app payments
```

### Helpdesk

```bash
bench get-app helpdesk
bench --site site1.local install-app helpdesk
```

---

# 13. Production Setup (Recommended)

Instead of running `bench start`, configure production services:

```bash
sudo apt install supervisor
bench setup production frappeuser
```

Start services:

```bash
sudo supervisorctl start all
```

This enables:

* automatic restart
* background services
* boot-time startup

---

# 14. Useful Bench Commands

Create another site:

```bash
bench new-site site2.local
```

Install app on site:

```bash
bench --site site1.local install-app appname
```

Update system:

```bash
bench update
```

Build assets:

```bash
bench build
```

Clear cache:

```bash
bench clear-cache
```

---

# 15. Troubleshooting Tips

### Redis Errors

If you see:

```
Connection refused 127.0.0.1:11000
```

Ensure Redis is installed and running:

```bash
sudo systemctl start redis-server
```

---

### Bench Should Not Run As Root

Always use:

```
frappeuser
```

Running bench as root causes permission conflicts.

---

### Port Already in Use

Check processes:

```bash
ps aux | grep redis
```

Kill unwanted processes:

```bash
kill -9 PID
```

---

# 16. Bench Architecture Overview

Frappe Bench runs several services:

```
           ┌───────────────┐
           │   Web Server  │
           └──────┬────────┘
                  │
      ┌───────────▼───────────┐
      │        Frappe         │
      │    (Python Backend)   │
      └───────────┬───────────┘
                  │
      ┌───────────▼───────────┐
      │       Workers         │
      │  Background Jobs      │
      └───────────┬───────────┘
                  │
           ┌──────▼──────┐
           │    Redis    │
           │ Queue/Cache │
           └─────────────┘
```

---

# 17. Next Steps

After installation you may want to:

* Configure **Nginx reverse proxy**
* Enable **HTTPS with Let's Encrypt**
* Setup **Backups**
* Configure **Email & Notifications**
* Customize ERPNext modules

---

# License

MIT License
