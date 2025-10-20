# 🧱 Multi-Service Stack with systemd

## 🧭 Overview

This project demonstrates how I designed and managed a **multi-service Linux stack** using `systemd`.
It integrates **Nginx**, **Flask (Gunicorn)**, and **PostgreSQL**  all orchestrated with proper dependency ordering and automated health checks.

The goal was to replicate how real production servers handle web apps, database dependencies, and service orchestration without relying on Docker or container orchestration tools.

**Primary Technologies:**

* Linux (Ubuntu 22.04 LTS on Linode)
* systemd / systemctl / journalctl
* Nginx
* Flask + Gunicorn
* PostgreSQL
* Python3 (with psutil)

**Deployment Platform:** Linode Cloud Server (1 node)

---

## 🖼️ Screenshot Gallery

📌 **Screenshot Placeholder:** Output of `systemctl list-units --type=service | grep flask_app`
<img width="432" height="152" alt="listunits" src="https://github.com/user-attachments/assets/add76cee-2ce8-49a7-8197-06aaac53386a" />

📌 **Screenshot Placeholder:** Browser displaying “Hello from Flask via systemd!”
<img width="265" height="36" alt="flask thanks" src="https://github.com/user-attachments/assets/643a99f0-95e0-4238-b39e-a55ee8ea0917" />


📌 **Screenshot Placeholder:** Output of `systemctl list-timers | grep health_check`
<img width="523" height="27" alt="health check" src="https://github.com/user-attachments/assets/0d71ad08-ebd2-4c06-a2aa-7c4a09471526" />


📌 **Screenshot Placeholder:** `journalctl -u flask_app.service` logs showing restart events


---

## ⚙️ Project Structure

```
/etc/systemd/system/
 ├── flask_app.service
 ├── health_check.service
 ├── health_check.timer
/etc/nginx/sites-available/
 └── flask_app
/home/<user>/flask_app/
 ├── app.py
 └── venv/
```

---

## 🚀 Step-by-Step Implementation

### 1️⃣ Environment Setup

I updated the base system and installed all required components:

```bash
sudo apt update && sudo apt install -y nginx postgresql python3 python3-venv python3-pip git
sudo systemctl enable nginx postgresql
```

🔍 **Why:** Ensures the web and database services auto-start at boot — standard sysadmin procedure for persistent production uptime.

---

### 2️⃣ Flask Application

I built a lightweight Flask API to test process management.

```bash
mkdir -p ~/flask_app && cd ~/flask_app
python3 -m venv venv
source venv/bin/activate
pip install flask gunicorn
```

`app.py`

```python
from flask import Flask
app = Flask(__name__)

@app.route('/')
def index():
    return "Hello from Flask via systemd!"
```

Testing locally:

```bash
gunicorn -b 0.0.0.0:5000 app:app
```

📌 **Screenshot Placeholder:** Flask app running in browser at port `5000`.

<img width="213" height="55" alt="helloflask" src="https://github.com/user-attachments/assets/d84a0244-2130-4883-afe3-bf7bfd3c4cfb" />


---

### 3️⃣ Custom `systemd` Service

Created `/etc/systemd/system/flask_app.service`:

```ini
[Unit]
Description=Flask Application Service
After=network.target postgresql.service
Requires=postgresql.service

[Service]
User=www-data
WorkingDirectory=/home/<user>/flask_app
ExecStart=/home/<user>/flask_app/venv/bin/gunicorn -b 0.0.0.0:5000 app:app
Restart=always

[Install]
WantedBy=multi-user.target
```

Then enabled it:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now flask_app.service
```

💡 **Reasoning:**
This guarantees the app only launches *after* PostgreSQL and network interfaces are live.
It mirrors real DevOps dependency chains for application tiers.

🧠 **Learning Moment:**
Understanding `[Unit]` ordering (`After=`, `Requires=`) is fundamental when orchestrating services in large-scale Linux systems.

---

### 4️⃣ Reverse Proxy with Nginx

Created `/etc/nginx/sites-available/flask_app`:

```nginx
server {
    listen 80;
    server_name _;

    location / {
        proxy_pass http://127.0.0.1:5000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

Then linked and reloaded:

```bash
sudo ln -s /etc/nginx/sites-available/flask_app /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
```

📌 **Screenshot Placeholder:** Browser at `http://<your-linode-ip>` showing Flask’s “Hello” message.

<img width="171" height="28" alt="helloflask2" src="https://github.com/user-attachments/assets/ee5fb337-6c2f-4f29-9808-1c30c877b743" />
<img width="213" height="55" alt="helloflask" src="https://github.com/user-attachments/assets/a50dcfb8-5408-4d54-909c-8ed5c788ddc4" />


---

### 5️⃣ Health Check Automation via `systemd` Timer

Installed psutil:

```bash
sudo pip install psutil
```

Created `/usr/local/bin/health_check.py`:

```python
#!/usr/bin/env python3
import datetime, psutil

log = "/var/log/health_check.log"
with open(log, "a") as f:
    f.write(f"[{datetime.datetime.now()}] CPU: {psutil.cpu_percent()}% | RAM: {psutil.virtual_memory().percent}%\n")
```

Made it executable:

```bash
sudo chmod +x /usr/local/bin/health_check.py
```

Created `/etc/systemd/system/health_check.service`:

```ini
[Unit]
Description=System Health Check Script

[Service]
ExecStart=/usr/bin/python3 /usr/local/bin/health_check.py
```

Created `/etc/systemd/system/health_check.timer`:

```ini
[Unit]
Description=Run Health Check Every 10 Minutes

[Timer]
OnBootSec=2min
OnUnitActiveSec=10min
Unit=health_check.service

[Install]
WantedBy=timers.target
```

Enabled both:

```bash
sudo systemctl enable --now health_check.timer
```

📌 **Screenshot Placeholder:** `systemctl list-timers | grep health_check`

<img width="523" height="27" alt="health check" src="https://github.com/user-attachments/assets/31eca31e-1996-4ceb-8d03-28b67a2a6b53" />


---

### 6️⃣ Monitoring and Verification

Checked live service health:

```bash
sudo systemctl status flask_app.service nginx postgresql health_check.timer
```

Viewed logs:

```bash
sudo journalctl -u flask_app.service --since "10 min ago"
sudo tail /var/log/health_check.log
```
<img width="599" height="413" alt="step6" src="https://github.com/user-attachments/assets/03ffcf14-bc63-41e0-99c7-7fd54012a0a5" />

🧩 **Real-World Context:**
This is exactly how DevOps engineers confirm whether services deployed via automation tools like Ansible or Terraform are functioning properly.

---

## 🧠 Challenges & Solutions

| Challenge                                              | Solution                                                                 |
| ------------------------------------------------------ | ------------------------------------------------------------------------ |
| Flask service failed to start due to permission issues | Changed `User=` to `www-data` and ensured correct directory permissions  |
| systemd timer not triggering                           | Ran `sudo systemctl daemon-reload` and re-enabled the timer              |
| Gunicorn not found in path                             | Used full path to virtualenv binary in `ExecStart=`                      |
| Conflicting Nginx default site                         | Removed `/etc/nginx/sites-enabled/default` before enabling custom config |

---

## 🧩 Summary of What I Achieved

✅ Configured and linked **three major services** under systemd
✅ Created **custom `.service` and `.timer` units** for automation
✅ Controlled dependency order between network, DB, and web layers
✅ Logged, monitored, and validated all services via `journalctl`
✅ Automated periodic health monitoring with a **Python timer job**

---

## 🧹 Cleanup Instructions

To reset the environment:

```bash
sudo systemctl disable --now flask_app.service health_check.timer nginx postgresql
sudo rm /etc/systemd/system/{flask_app.service,health_check.service,health_check.timer}
sudo rm /etc/nginx/sites-enabled/flask_app /etc/nginx/sites-available/flask_app
sudo rm -rf ~/flask_app /var/log/health_check.log
sudo systemctl daemon-reload
sudo apt remove -y nginx postgresql
```

This restores a clean Linode environment for future projects.

---

## 🧠 Key Learnings

* `systemd` is the backbone of process management on modern Linux systems.
* Proper dependency ordering ensures reliable multi-tier application startup.
* `systemd` timers are more flexible and maintainable than legacy `cron`.
* Observability with `journalctl` is essential for production troubleshooting.

---

## 🔗 References

* [systemd Documentation](https://www.freedesktop.org/software/systemd/man/systemd.html)
* [Gunicorn Docs](https://docs.gunicorn.org/en/stable/run.html)
* [Nginx Reverse Proxy Guide](https://docs.nginx.com/nginx/admin-guide/web-server/reverse-proxy/)
* [Linode Docs: Managing Services](https://www.linode.com/docs/guides/introduction-to-systemctl/)
