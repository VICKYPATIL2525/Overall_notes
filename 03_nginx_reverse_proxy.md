# Nginx Reverse Proxy — Forward Traffic to Your App

**Purpose:** Configure Nginx to forward HTTP traffic from port 80 to your app's internal port, making it accessible at a clean URL with no port number.

> **Prerequisites:**
> - App must be running as a background service — see [01_systemd_app_service.md](01_systemd_app_service.md)
> - Nginx must be installed and running — see [02_nginx_setup.md](02_nginx_setup.md)

---

## Table of Contents

1. [How It Works](#how-it-works)
2. [Key Concept — Proxy Headers](#key-concept--proxy-headers)
3. [Setup Steps](#setup-steps)
   - [Step 1 — Verify Your App is Running](#step-1--verify-your-app-is-running)
   - [Step 2 — Create the Nginx Config File](#step-2--create-the-nginx-config-file)
   - [Step 3 — Write the Config](#step-3--write-the-config)
   - [Step 4 — Enable the Config](#step-4--enable-the-config)
   - [Step 5 — Test for Errors](#step-5--test-for-errors)
   - [Step 6 — Reload Nginx](#step-6--reload-nginx)
   - [Step 7 — Test in Browser](#step-7--test-in-browser)
4. [Shortcut Method](#shortcut-method)
5. [Config Field Explanations](#config-field-explanations)
6. [Daily Commands](#daily-commands)
7. [Troubleshooting](#troubleshooting)
8. [What This Setup Achieves](#what-this-setup-achieves)
9. [Next Step — Custom Domain](#next-step--custom-domain)

---

## How It Works

**Before** (without Nginx reverse proxy):
```
Internet Traffic
      ↓
Port 8000  —  Your App
(User must type :8000 in the URL)
```

**After** (with Nginx reverse proxy):
```
Internet Traffic
      ↓
Port 80  —  Nginx (Reverse Proxy)
      ↓
Port 8000  —  Your App
(User types just http://ip — no port needed)
```

Nginx intercepts all traffic on port 80 and invisibly forwards it to your app's internal port. The user never sees the internal port.

---

## Key Concept — Proxy Headers

When Nginx forwards a request to your app, it must also pass along information about the original request. Without these headers, your app loses track of who is actually connecting.

| Header | Purpose |
|---|---|
| `proxy_http_version 1.1` | Uses HTTP/1.1 — required for WebSocket support |
| `Upgrade $http_upgrade` | Allows HTTP connection to upgrade to WebSocket |
| `Connection "upgrade"` | Works with Upgrade — keeps WebSocket connections alive |
| `Host $host` | Forwards original domain/IP so your app builds correct internal URLs |
| `X-Real-IP $remote_addr` | Passes the real user IP to your app |

---

## Setup Steps

### Step 1 — Verify Your App is Running

```bash
sudo systemctl status myapp
```

Expected: `Active: active (running)`

If your app is not running, start it first — see [01_systemd_app_service.md](01_systemd_app_service.md).

---

### Step 2 — Create the Nginx Config File

```bash
sudo nano /etc/nginx/sites-available/myapp
```

Name it anything that identifies your app — `myapp`, `chatbot`, `dashboard`, etc.

---

### Step 3 — Write the Config

Paste the following and replace `8000` with your app's actual port:

```nginx
server {
    listen 80;
    server_name _;

    location / {
        proxy_pass http://localhost:8000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

Save and exit: `Ctrl + X` → `Y` → `Enter`

---

### Step 4 — Enable the Config

```bash
sudo ln -s /etc/nginx/sites-available/myapp /etc/nginx/sites-enabled/
```

This activates the config. Without this step Nginx never reads the file.

---

### Step 5 — Test for Errors

```bash
sudo nginx -t
```

Expected:
```
nginx: configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

If it says `failed` — there is a typo in your config. Fix it before moving on.

---

### Step 6 — Reload Nginx

```bash
sudo systemctl reload nginx
```

Applies the new config with zero downtime — your app stays live.

---

### Step 7 — Test in Browser

```
http://your-elastic-ip
```

You should see your app with no port number in the URL.

---

## Shortcut Method

Instead of using nano, overwrite the config in one command:

```bash
sudo tee /etc/nginx/sites-available/myapp > /dev/null <<EOF
server {
    listen 80;
    server_name _;

    location / {
        proxy_pass http://localhost:8000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade \$http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host \$host;
        proxy_set_header X-Real-IP \$remote_addr;
    }
}
EOF
```

Then enable and reload:
```bash
sudo ln -s /etc/nginx/sites-available/myapp /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

---

## Config Field Explanations

| Line | Meaning |
|---|---|
| `listen 80;` | Accept traffic coming in on port 80 (default HTTP port) |
| `server_name _;` | Match all domain names — underscore is a wildcard catch-all |
| `location / {` | Apply these rules to all incoming URLs |
| `proxy_pass http://localhost:8000;` | Forward the request to your app's internal port |
| `proxy_http_version 1.1;` | Use HTTP/1.1 (required for WebSocket connections) |
| `Upgrade $http_upgrade;` | Allow connection upgrade to WebSocket |
| `Connection "upgrade";` | Keep WebSocket connections alive |
| `Host $host;` | Forward the original Host header to your app |
| `X-Real-IP $remote_addr;` | Forward the real user IP to your app |

---

## Daily Commands

| Action | Command |
|---|---|
| Restart Nginx (apply changes) | `sudo systemctl restart nginx` |
| Reload Nginx (no downtime) | `sudo systemctl reload nginx` |
| Check Nginx status | `sudo systemctl status nginx` |
| Test config syntax | `sudo nginx -t` |
| View error logs | `sudo tail -f /var/log/nginx/error.log` |
| View access logs | `sudo tail -f /var/log/nginx/access.log` |

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| 502 Bad Gateway | Your app is not running | Start it — see [01_systemd_app_service.md](01_systemd_app_service.md) |
| Connection refused | Nginx is down or port 80 is closed | Check `systemctl status nginx` and EC2 Security Group |
| Config error on reload | Typo in config file | Run `sudo nginx -t` to see the exact error |
| App missing real-time features | WebSocket headers missing | Verify `Upgrade` and `Connection` headers are in the config |

---

## What This Setup Achieves

- App accessible at clean URL `http://your-ip` — no port number needed
- Only port 80 exposed to the internet — internal app port is hidden
- Nginx acts as a buffer between the internet and your app
- WebSocket support for real-time features
- Easy to extend — add more apps by creating additional config files

---

## Next Step — Custom Domain

To use a domain like `app.yourdomain.com` instead of a raw IP:

1. Change `server_name _;` to `server_name app.yourdomain.com;`
2. Point your domain's DNS A record to the server's elastic IP
3. Reload Nginx: `sudo systemctl reload nginx`
