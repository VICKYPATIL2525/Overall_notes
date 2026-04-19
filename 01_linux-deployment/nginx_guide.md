# Nginx Guide

**Purpose:** Understand what Nginx is, why it exists, and how to install and connect it to your app running on an EC2 server.

---

## Table of Contents

1. [What is Nginx?](#what-is-nginx)
2. [The Problem Without Nginx](#the-problem-without-nginx)
3. [How Nginx Solves This](#how-nginx-solves-this)
4. [Basic Concepts](#basic-concepts)
5. [Installation](#installation)
   - [Step 1 — Install Nginx](#step-1--install-nginx)
   - [Step 2 — Start Nginx](#step-2--start-nginx)
   - [Step 3 — Enable Auto-Start on Reboot](#step-3--enable-auto-start-on-reboot)
   - [Step 4 — Check Status](#step-4--check-status)
   - [Step 5 — Test in Browser](#step-5--test-in-browser)
6. [Troubleshooting — If the Page Does Not Open](#troubleshooting--if-the-page-does-not-open)
7. [What Just Happened](#what-just-happened)
8. [Step 6 — Connect Nginx to Streamlit](#step-6--connect-nginx-to-streamlit)
9. [Where Nginx Config Files Live](#where-nginx-config-files-live)
10. [Daily Commands](#daily-commands)

---

## What is Nginx?

Nginx (pronounced "engine-x") is a **web server and reverse proxy**. It sits in front of your application and handles all incoming traffic from the internet before passing it to your app.

Think of it as a **traffic controller** — it decides where each request should go.

---

## The Problem Without Nginx

When you run a on an EC2 server, it starts on a specific port, for example:

```
http://your-ip:8501
```

Without Nginx, you face these problems:

| Problem | Why it's bad |
|---|---|
| Users must type the port (`:8501`) in the URL | Looks unprofessional, easy to forget |
| No way to use a clean domain like `app.yourdomain.com` | Can't point a domain to a port directly |
| No HTTPS/SSL out of the box | Insecure — browser shows "Not Safe" warning |
| Only one app can use port 80 (default web port) | Can't run multiple apps on the same server easily |
| You'd have to open every app port in the Security Group | Security risk — too many open ports |

### What you'd have to do manually (without Nginx)

- Open port 8501 in the EC2 Security Group for every app
- Tell every user to add `:8501` to the URL
- Set up SSL certificates manually per app
- Write custom scripts to route traffic between multiple apps

This becomes messy and hard to manage fast.

---

## How Nginx Solves This

Nginx acts as a **reverse proxy** — it receives all traffic on port 80 (HTTP) or 443 (HTTPS) and forwards it internally to your app on port 8501.

```
Internet → Port 80 → Nginx → Port 8501 → Your App
```

Benefits:

- Users visit a clean URL like `http://your-ip` or `app.yourdomain.com` — no port needed
- Only port 80/443 needs to be open in the Security Group
- You can run multiple apps on one server — Nginx routes each to the right one
- SSL is handled once at the Nginx level — one certificate on the server covers all apps running behind it. Each individual app does not need its own SSL setup.

---

## Basic Concepts

| Term | Meaning |
|---|---|
| **Web Server** | Serves files (HTML, CSS, images) directly to the browser. Nginx can do this, but you mainly use it as a reverse proxy for apps. |
| **Reverse Proxy** | Sits in front of your app and forwards requests to it. The user talks to Nginx, Nginx talks to your app — the app is never directly exposed. |
| **Port 80** | Default HTTP port. Browsers automatically use this when you type just `http://your-ip` — no need to add `:80` in the URL. |
| **Port 443** | Default HTTPS port. Used for secure encrypted connections. Browsers show the padlock icon when this is active. |
| **Upstream** | The app running internally that Nginx forwards traffic to. Example: your Streamlit app on port 8501 is the upstream — hidden from the internet. |
| **Server Block** | A section in the Nginx config file that defines rules for one domain or IP — what port to listen on, where to forward traffic, and which SSL cert to use. |

---

## Installation

### Step 1 — Install Nginx

```bash
sudo apt install nginx -y
```

This downloads and installs Nginx from Ubuntu's package manager. The `-y` flag auto-confirms the installation so it doesn't pause and ask you "do you want to continue?". After this step Nginx is installed on the server but not running yet.

---

### Step 2 — Start Nginx

```bash
sudo systemctl start nginx
```

This starts the Nginx process right now. After this command, Nginx is running and listening for traffic. However, if the server reboots, Nginx will NOT start automatically — that's what Step 3 handles.

---

### Step 3 — Enable Auto-Start on Reboot

```bash
sudo systemctl enable nginx
```

**What this does:** Registers Nginx with the system so it starts automatically every time the server boots up.

**Why this matters:** EC2 instances get restarted during maintenance, updates, or if you stop and start them. Without this step, Nginx stays off after every reboot and your app goes down — you'd have to SSH in and manually start it each time.

**Is this necessary every time?** Yes — run this once on every new server you set up. You only need to do it once per server, not every time you change a config. It's a one-time setup step.

> Think of Step 2 as "start now" and Step 3 as "always start after reboot". You need both.

---

### Step 4 — Check Status

```bash
sudo systemctl status nginx
```

Confirms that Nginx is actually running. Always run this after starting or restarting Nginx to make sure there are no errors.

You should see:

```
Active: active (running)
```

If you see `failed` or `inactive`, something went wrong — check the Troubleshooting section below.

---

### Step 5 — Test in Browser

Open your browser and go to:

```
http://your-elastic-ip
```

You should see the default **"Welcome to nginx!"** page.

![Nginx Welcome Screen](../images/nginx%20welcome%20screen.png)

This confirms Nginx is installed, running, and reachable from the internet. At this point Nginx is not yet connected to your app — it's just showing its own default page. The next step is to configure it to forward traffic to your Streamlit app.

---

## Troubleshooting — If the Page Does Not Open

**Check 1: Security Group**

Make sure port 80 is allowed in your EC2 Security Group inbound rules.

**Check 2: Firewall (if UFW is enabled)**

```bash
sudo ufw allow 'Nginx Full'
```

---

## What Just Happened

At this point your server looks like this:

```
Internet → Port 80 → Nginx → (default welcome page)
```

Nginx is running but not yet connected to your Streamlit app. That's the next step.

---

## Step 6 — Connect Nginx to Streamlit

Once you confirm the "Welcome to nginx!" page loads, you'll update the Nginx config to forward traffic to your Streamlit app running on port 8501.

```
Internet → Port 80 → Nginx → Port 8501 → Streamlit App
```

> Config steps for this will be added in the next section.

---

## Where Nginx Config Files Live

Nginx splits its configuration across three locations:

| Path | Purpose |
|---|---|
| `/etc/nginx/nginx.conf` | **Master config** — global settings (worker processes, logging). It also imports everything from `sites-enabled/` automatically. |
| `/etc/nginx/sites-available/` | Where you **write** your site configs. Files here are inactive — think of it as a drafts folder. |
| `/etc/nginx/sites-enabled/` | Where **active** configs live. Nginx reads only from here. |

### The Enable / Disable Pattern

You don't copy files between folders. You create a **symlink** (a shortcut) from `sites-enabled/` pointing to the file in `sites-available/`:

```bash
# Enable a site
sudo ln -s /etc/nginx/sites-available/myapp /etc/nginx/sites-enabled/

# Disable a site (original file stays safe)
sudo rm /etc/nginx/sites-enabled/myapp
```

This means you can have 10 site configs written but only 3 active — clean and reversible.

---

### The Site Config File — Accepting Traffic and Routing It

This is the most important part. This file is where you tell Nginx:
- **Which port to listen on** (where traffic comes IN from the internet)
- **Where to send that traffic** (which internal port your app is running on)

---

#### Step 1 — Create the Config File

```bash
sudo nano /etc/nginx/sites-available/myapp
```

This opens a blank file in the editor. You can name it anything — `myapp`, `streamlit`, `chatbot`, etc. The name is just a label for you to identify it.

---

#### Step 2 — Write the Server Block

Paste this inside the file:

```nginx
server {

    listen 80;

    server_name your-ip-or-domain;

    location / {
        proxy_pass http://localhost:8501;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

}
```

Save and exit: press `Ctrl + X`, then `Y`, then `Enter`.

---

#### What Each Line Does

**`listen 80;`**
- Port 80 is the default HTTP port — the one browsers use automatically when you type just `http://your-ip`
- This line tells Nginx: "sit here and wait for anyone connecting on port 80"
- Traffic from the internet arrives HERE first

**`server_name your-ip-or-domain;`**
- Replace this with your actual EC2 elastic IP (e.g. `54.123.45.67`) or your domain (e.g. `app.mysite.com`)
- If you have multiple server blocks for different apps, Nginx uses this to decide which block handles which request
- For a single app, you can also just put `_` here as a catch-all

**`location / { ... }`**
- The `/` means "match ALL incoming URLs" — anything the user types after your IP goes in here
- Everything inside the `{ }` is what Nginx does with that request

**`proxy_pass http://localhost:8501;`**
- This is the routing line — it takes the request that came in on port 80 and forwards it to port 8501 on the same server
- `localhost` means "this same machine" — your Streamlit app is running there internally
- The user never sees port 8501 — they only ever talk to Nginx on port 80

**`proxy_set_header` lines**
- These pass extra information to your app so it knows the real IP of the user who made the request
- Without these, your app would think every request came from Nginx itself, not the actual user

---

#### The Full Picture

```
User types http://54.123.45.67 in browser
        ↓
    Port 80 (internet-facing)
        ↓
      Nginx reads this config file
        ↓
    proxy_pass → Port 8501 (internal, not exposed)
        ↓
    Your Streamlit App responds
        ↓
    Nginx sends the response back to the user
```

---

#### Step 3 — Enable the Config

Writing the file is not enough — you need to activate it by creating a symlink into `sites-enabled/`:

```bash
sudo ln -s /etc/nginx/sites-available/myapp /etc/nginx/sites-enabled/
```

Think of this as: "I wrote the rule in `sites-available`, now I'm switching it ON by linking it into `sites-enabled`."

---

#### Step 4 — Test for Errors

Before reloading, always check your config for typos:

```bash
sudo nginx -t
```

You should see:

```
nginx: configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

If it says `failed` — there's a typo in your server block. Fix it before moving on.

---

#### Step 5 — Reload Nginx

```bash
sudo systemctl reload nginx
```

This applies the new config without restarting Nginx — zero downtime. Your app stays live while the new routing rules take effect.

---

> **Key rule:** Without the symlink in Step 3, the config file does absolutely nothing — Nginx never reads it.

---

## Daily Commands

| Action | Command |
|---|---|
| Start | `sudo systemctl start nginx` |
| Stop | `sudo systemctl stop nginx` |
| Restart | `sudo systemctl restart nginx` |
| Check status | `sudo systemctl status nginx` |
| Test config | `sudo nginx -t` |
| Reload config (no downtime) | `sudo systemctl reload nginx` |
