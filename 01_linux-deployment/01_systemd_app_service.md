# Systemd App Service Guide

**Purpose:** Run any web app in the background as a system service — auto-starts on reboot, auto-restarts on crash, no manual steps required.

> **Next Step:** Once your app is running via systemd, set up Nginx to expose it on port 80 — see [02_nginx_setup.md](02_nginx_setup.md).

---

## Table of Contents

1. [What is Systemd?](#what-is-systemd)
2. [Prerequisites](#prerequisites)
3. [Setup Steps](#setup-steps)
   - [Step 1 — Navigate to Project Folder](#step-1--navigate-to-project-folder)
   - [Step 2 — Create the Service File](#step-2--create-the-service-file)
   - [Step 3 — Add Configuration](#step-3--add-configuration)
   - [Step 4 — Reload Systemd](#step-4--reload-systemd)
   - [Step 5 — Start the App](#step-5--start-the-app)
   - [Step 6 — Check Status](#step-6--check-status)
   - [Step 7 — Enable Auto-Start on Boot](#step-7--enable-auto-start-on-boot)
   - [Step 8 — Access the App](#step-8--access-the-app)
4. [Daily Commands](#daily-commands)
5. [Debugging](#debugging)
6. [What This Achieves](#what-this-achieves)

---

## What is Systemd?

Systemd is the process manager built into Linux. It controls what runs when the server starts up and keeps services alive in the background.

When you register your app with systemd, Linux treats it like any other system service — it starts automatically on boot, restarts if it crashes, and runs without you needing to be logged in.

---

## Prerequisites

- Ubuntu server with your app ready
- Virtual environment set up at the project path
- `sudo` access

---

## Setup Steps

### Step 1 — Navigate to Project Folder

```bash
cd ~/your-project-folder
pwd
```

Replace `your-project-folder` with the actual path to your project. `pwd` confirms you are in the right place.

---

### Step 2 — Create the Service File

```bash
sudo nano /etc/systemd/system/myapp.service
```

Replace `myapp` with any name that identifies your app — e.g. `chatbot`, `dashboard`, `api`. This name is what you'll use in all systemctl commands later.

---

### Step 3 — Add Configuration

Paste the following into the file and update the values marked with `< >`:

```ini
[Unit]
Description=My Web App
After=network.target

[Service]
User=ubuntu
WorkingDirectory=/home/ubuntu/<your-project-folder>
ExecStart=/home/ubuntu/<your-project-folder>/myenv/bin/python <your-start-command>
Restart=always

[Install]
WantedBy=multi-user.target
```

**Example — Streamlit app:**
```ini
ExecStart=/home/ubuntu/myproject/myenv/bin/streamlit run app.py --server.port 8000 --server.address 0.0.0.0
```

**Example — FastAPI app:**
```ini
ExecStart=/home/ubuntu/myproject/myenv/bin/uvicorn main:app --host 0.0.0.0 --port 8000
```

Save and exit: `Ctrl + X` → `Y` → `Enter`

**Configuration explained:**

| Field | Description |
|---|---|
| `Description` | A label — shown in status output |
| `After=network.target` | Wait for network to be ready before starting |
| `User` | Which Linux user runs the app |
| `WorkingDirectory` | Project root folder — all relative paths in your app resolve from here |
| `ExecStart` | The exact command to start your app using the virtual environment |
| `Restart=always` | Auto-restarts the app if it crashes for any reason |
| `WantedBy=multi-user.target` | Enables auto-start on system boot |

---

### Step 4 — Reload Systemd

```bash
sudo systemctl daemon-reload
```

Run this every time you create or modify a service file. It tells systemd to re-read all service files.

---

### Step 5 — Start the App

```bash
sudo systemctl start myapp
```

Starts the app in the background immediately.

---

### Step 6 — Check Status

```bash
sudo systemctl status myapp
```

Confirms the app is running. Look for:

```
Active: active (running)
```

If you see `failed` — check the logs (see Debugging section below).

---

### Step 7 — Enable Auto-Start on Boot

```bash
sudo systemctl enable myapp
```

Without this step the app will not start automatically after a server reboot. Run this once per service after setting it up.

---

### Step 8 — Access the App

```
http://<server-ip>:<your-port>
```

At this point your app is accessible via the port you set in `ExecStart` (e.g. `8000`). Users will need to type the port in the URL.

> To remove the port from the URL and serve on clean `http://your-ip`, set up Nginx as a reverse proxy — see [02_nginx_setup.md](02_nginx_setup.md) and [03_nginx_reverse_proxy.md](03_nginx_reverse_proxy.md).

---

## Daily Commands

| Action | Command |
|---|---|
| Start | `sudo systemctl start myapp` |
| Stop | `sudo systemctl stop myapp` |
| Restart | `sudo systemctl restart myapp` |
| Check status | `sudo systemctl status myapp` |
| View live logs | `journalctl -u myapp -f` |

---

## Debugging

```bash
journalctl -u myapp -n 50
```

Shows the last 50 log lines. Use this when the app fails to start or crashes unexpectedly.

---

## What This Achieves

- App runs in the background (survives SSH disconnect)
- Auto-starts on server reboot
- Auto-restarts on crash
- No manual intervention needed after setup
