# Selenium API - VPS Setup and Documentation

This document contains step-by-step instructions for setting up, running, and accessing your Selenium API on a Linux VPS.

---

## 1. Project Structure

```
/var/www/selenium/api
│
├── api_main.py        # FastAPI main file
├── login.py           # Selenium script
├── requirements.txt   # Python dependencies
├── .env               # Environment variables
├── scripts/           # Folder containing Selenium scripts
└── venv/              # Python virtual environment
```

* `/var/www/selenium/api` is the recommended location for web apps on Linux.
* Consistency and documentation are more important than the exact folder.

---

## 2. Virtual Environment Setup

1. Install venv package (Debian/Ubuntu):

```bash
sudo apt update
sudo apt install python3-venv
```

2. Create a virtual environment inside the project:

```bash
cd /var/www/selenium/api
python3 -m venv venv
```

3. Activate the virtual environment:

```bash
source venv/bin/activate
```

4. Deactivate when done:

```bash
deactivate
```

---

## 3. Python Dependencies

**requirements.txt** example (latest versions):

```
fastapi
uvicorn
selenium
requests
cloudinary
```

Install dependencies:

```bash
pip install -r requirements.txt
```

> `requirements.txt` is equivalent to Node.js `package.json`. It lists all Python packages needed.

---

## 4. Environment Variables (.env)

**.env** content example:

```
CLIENT_EMAIL=your_email@example.com
CLIENT_PASSWORD=your_password
BUBBLE_API_URL=https://happier.selenium.ai/version-test/api/1.1/obj/Screenshots
BUBBLE_API_TOKEN=your_bubble_api_token
CLOUDINARY_CLOUD_NAME=your_cloud_name
CLOUDINARY_API_KEY=your_api_key
CLOUDINARY_API_SECRET=your_api_secret
```

* Use `python-dotenv` or `os.getenv()` in scripts to access these variables.
* Never commit `.env` to version control.

---

## 5. Running the FastAPI Server

### Development (with auto-reload)

```bash
uvicorn api_main:app --reload --host 0.0.0.0 --port 8001
```

* `--host 0.0.0.0` makes the server accessible from the VPS public IP.
* Swagger docs: `http://<VPS-IP>:8001/docs`
* ReDoc: `http://<VPS-IP>:8001/redoc`

### Production (with Gunicorn)

```bash
gunicorn api_main:app -k uvicorn.workers.UvicornWorker --bind 127.0.0.1:8001
```

* Use Nginx to reverse proxy from `80/443` to `127.0.0.1:8001`.
* Ensures Uvicorn is not directly exposed.

---

## 6. Making the API Auto-Start on Reboot (systemd)

1. Create a systemd service file:

```bash
sudo nano /etc/systemd/system/selenium-api.service
```

Paste the following:

```ini
[Unit]
Description=Selenium FastAPI Service
After=network.target

[Service]
User=cyril
WorkingDirectory=/var/www/selenium/api
Environment="PATH=/var/www/selenium/api/venv/bin"
ExecStart=/var/www/selenium/api/venv/bin/uvicorn api_main:app --host 0.0.0.0 --port 8001
Restart=always

[Install]
WantedBy=multi-user.target
```

2. Enable and start the service:

```bash
sudo systemctl daemon-reload
sudo systemctl enable selenium-api
sudo systemctl start selenium-api
```

3. Check status:

```bash
sudo systemctl status selenium-api
```

> Now your API will **restart automatically after server reboots**. Use this setup for production instead of running Uvicorn manually.

---

## 7. Handling Port Conflicts

1. Check which process is using a port:

```bash
sudo lsof -i :8000
```

2. Kill process if necessary:

```bash
sudo kill -9 <PID>
```

3. Always choose an unused port for Uvicorn (e.g., 8001).

---

## 8. Firewall Setup (UFW)

1. Allow the API port:

```bash
sudo ufw allow 8001/tcp
```

2. Check UFW status:

```bash
sudo ufw status
```

3. Ensure VPS provider firewall also allows this port.

---

## 9. Testing Access

1. Test locally on VPS:

```bash
curl http://127.0.0.1:8001/docs
```

2. Test externally in browser:

```
http://<VPS-IP>:8001/docs
```

* If using Nginx reverse proxy: `https://api.selenium.ai/docs`

---

## 10. Selenium Script Notes

* Place scripts in `/scripts` folder.
* Use environment variables from `.env`.
* Example structure:

```python
import os
CLIENT_EMAIL = os.getenv("CLIENT_EMAIL")
CLIENT_PASSWORD = os.getenv("CLIENT_PASSWORD")
```

* Handles Xvfb, video recording, and Cloudinary uploads.

---

## 11. Recommended Workflow

1. Activate virtual environment:

```bash
source venv/bin/activate
```

2. Run FastAPI for development:

```bash
uvicorn api_main:app --reload --host 0.0.0.0 --port 8001
```

3. Access Swagger docs: `http://<VPS-IP>:8001/docs`
4. Make API calls from Bubble frontend.
5. Deactivate venv when done.

---

## 12. Notes

* Consistency is key: always document your ports, virtual environment, and directories.
* For production, use **systemd + Gunicorn + Nginx + HTTPS**. Do not expose Uvicorn directly to the internet.
* Keep `.env` secret and never commit it.
* Check for port conflicts before starting the API.
* Use `CTRL+C` or `kill <PID>` to stop running servers.
* The `--reload` flag is for development only; in production use systemd/Gunicorn.

---

This documentation serves as a complete guide to setting up the Selenium API project on a VPS, including environment setup, firewall, server running, auto-start on reboot, and external access.
