**Supabase Keepalive & Dashboard - Python Guide**

---

# 1️⃣ Directory Structure

```
cron-jobs/
├── .env
├── .gitignore
├── requirements.txt
├── logs/
│   └── ping.log
└── src/
    ├── app.py          # Flask dashboard + ping logic
    └── supabase_ping.py  # Ping function module
```

---

# 2️⃣ .env File

```env
SUPABASE_URL=https://YOUR_PROJECT_ID.supabase.co
SUPABASE_ANON_KEY=YOUR_PUBLIC_ANON_KEY
EDGE_FUNCTION_URL=https://YOUR_PROJECT_ID.functions.supabase.co/ping
```

# 3️⃣ Python Ping Module (src/supabase_ping.py)

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

SUPABASE_URL = os.getenv("SUPABASE_URL")
SUPABASE_ANON_KEY = os.getenv("SUPABASE_ANON_KEY")
EDGE_FUNCTION_URL = os.getenv("EDGE_FUNCTION_URL")

HEADERS = {
    "apikey": SUPABASE_ANON_KEY,
    "Authorization": f"Bearer {SUPABASE_ANON_KEY}"
}

def ping_supabase():
    try:
        res = requests.get(f"{SUPABASE_URL}/rest/v1/your_table?select=id&limit=1", headers=HEADERS, timeout=5)
        status = {"endpoint": "REST", "status_code": res.status_code, "message": "OK" if res.status_code == 200 else "Warning"}
    except Exception as e:
        status = {"endpoint": "REST", "status_code": 500, "message": str(e)}
    return status

def ping_edge():
    try:
        res = requests.get(EDGE_FUNCTION_URL, headers=HEADERS, timeout=5)
        status = {"endpoint": "Edge Function", "status_code": res.status_code, "message": "OK" if res.status_code == 200 else "Warning"}
    except Exception as e:
        status = {"endpoint": "Edge Function", "status_code": 500, "message": str(e)}
    return status
```

---

# 4️⃣ Flask Dashboard (src/app.py)

```python
from flask import Flask, render_template_string, jsonify
from supabase_ping import ping_supabase, ping_edge
import os
import time

app = Flask(__name__)
LOG_FILE = os.path.join(os.path.dirname(os.path.abspath(__file__)), "../logs/ping.log")

HTML_TEMPLATE = """
<!DOCTYPE html>
<html>
<head>
    <title>Supabase Status Dashboard</title>
    <style>
        body { font-family: Arial; padding: 20px; }
        .status { margin-bottom: 10px; }
        .ok { color: green; }
        .warn { color: orange; }
        .error { color: red; }
    </style>
    <script>
        async function refreshStatus(){
            const res = await fetch('/status');
            const data = await res.json();
            const container = document.getElementById('status-container');
            container.innerHTML = '';
            data.forEach(s => {
                const div = document.createElement('div');
                div.className = 'status ' + (s.status_code === 200 ? 'ok' : (s.status_code < 500 ? 'warn':'error'));
                div.innerText = `${s.endpoint}: ${s.message} (${s.status_code})`;
                container.appendChild(div);
            });
        }
        setInterval(refreshStatus, 10000);
        window.onload = refreshStatus;
    </script>
</head>
<body>
    <h1>Supabase Status Dashboard</h1>
    <div id="status-container"></div>
</body>
</html>
"""

@app.route("/")
def index():
    return render_template_string(HTML_TEMPLATE)

@app.route("/status")
def status():
    results = []
    results.append(ping_supabase())
    results.append(ping_edge())

    with open(LOG_FILE, "a") as f:
        log_line = f"{time.ctime()} | " + " | ".join([f"{r['endpoint']}={r['status_code']}" for r in results]) + "\n"
        f.write(log_line)

    return jsonify(results)

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
```

---

# 5️⃣ Install Dependencies

```bash
pip3 install flask requests python-dotenv
```

Or via `requirements.txt`:
```
flask
requests
python-dotenv
```

```bash
pip3 install -r requirements.txt
```

---

# 6️⃣ Run Dashboard Manually

```bash
cd /home/youruser/cron-jobs/src
python3 app.py
```
- Access via `http://YOUR_VPS_IP:5000/`
- Auto-refresh every 10 seconds

---

# 7️⃣ Run on VPS Reboot (systemd)

### Service File `/etc/systemd/system/supabase-dashboard.service`

```ini
[Unit]
Description=Supabase Dashboard
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=youruser
WorkingDirectory=/home/youruser/cron-jobs/src
ExecStart=/usr/bin/python3 app.py
EnvironmentFile=/home/youruser/cron-jobs/.env
Restart=always

[Install]
WantedBy=multi-user.target
```

Enable service:
```bash
sudo systemctl daemon-reload
sudo systemctl enable supabase-dashboard.service
sudo systemctl start supabase-dashboard.service
```

---

# 8️⃣ Optional Enhancements

- Ping multiple endpoints (REST, Edge, Storage) sequentially or in parallel
- Add retry/backoff logic
- Save historical logs to CSV or SQLite
- Display charts on the dashboard
- Restrict access with basic auth or VPN for security

---

**✅ Outcome:**

- Real-time frontend showing Supabase REST + Edge function status
- Logs stored in `logs/ping.log`
- Auto-refresh in browser every 10s
- Auto-starts on VPS reboot via systemd

