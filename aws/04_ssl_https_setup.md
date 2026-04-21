# Enterprise SSL Certificate Setup on AWS EC2 with Nginx

**Purpose:** Configure HTTPS on an EC2 server using SSL certificate files provided by an IT team — without using Let's Encrypt or Certbot.

> **Prerequisites:**
> - EC2 instance running — see [01_ec2_guide.md](01_ec2_guide.md)
> - Nginx installed — see [02_nginx_setup.md](../02_nginx_setup.md)
> - Nginx reverse proxy configured — see [03_nginx_reverse_proxy.md](../03_nginx_reverse_proxy.md)

---

## Table of Contents

1. [Introduction](#introduction)
2. [Background — How This Works in Enterprise](#background--how-this-works-in-enterprise)
3. [Understanding the SSL Certificate Files](#understanding-the-ssl-certificate-files)
4. [How the Three Files Work Together](#how-the-three-files-work-together)
5. [Step-by-Step Configuration in Nginx](#step-by-step-configuration-in-nginx)
   - [Step 1 — Upload Certificate Files to the Server](#step-1--upload-certificate-files-to-the-server)
   - [Step 2 — Open the Nginx Config File](#step-2--open-the-nginx-config-file)
   - [Step 3 — Add the HTTPS Server Block](#step-3--add-the-https-server-block)
   - [Step 4 — Add HTTP to HTTPS Redirect](#step-4--add-http-to-https-redirect)
   - [Step 5 — Test and Restart Nginx](#step-5--test-and-restart-nginx)
   - [Step 6 — Verify in Browser](#step-6--verify-in-browser)
6. [Security Considerations](#security-considerations)
7. [Common Mistakes to Avoid](#common-mistakes-to-avoid)
8. [Key Takeaways](#key-takeaways)

---

## Introduction

In enterprise environments, SSL certificates are not always generated automatically using tools like Let's Encrypt or Certbot. Instead, the IT or DevOps team provides a set of certificate files issued and signed by a trusted Certificate Authority (CA) — often a paid provider like GoDaddy, DigiCert, or Sectigo.

This is called a **manual SSL configuration** — you receive the files, upload them to the server, and configure Nginx to use them.

---

## Background — How This Works in Enterprise

When you request a domain from the IT team, they typically send a ZIP file containing:

- A certificate file (your domain's identity)
- A `.pem` formatted version of the certificate
- A CA bundle file (the trust chain)
- A private key file (the secret used to encrypt traffic)

Each file has a distinct role — understanding them is the key to getting this working.

---

## Understanding the SSL Certificate Files

### The Main Certificate File

**Example filename:** `certificate.pem`

This is the SSL certificate issued for your domain.
- Proves your domain is legitimate and verified by a CA
- Contains the domain name, validity period, and issuer information

Think of this as your **identity card** for the domain.

### The PEM Format Certificate

**Example filename:** `certificate.pem`

The certificate saved in `.pem` format — the standard text-based format used in Linux. Content is Base64 encoded between `-----BEGIN CERTIFICATE-----` and `-----END CERTIFICATE-----` markers. This is the file referenced in your Nginx config.

### The CA Bundle (Chain Certificate)

**Example filename:** `ca_bundle.pem`

Provided by the Certificate Authority. Contains **intermediate certificates** that form a trust chain between your certificate and the root CA.
- Without this, some browsers may show a security warning even if your certificate is valid
- Referenced as `ssl_trusted_certificate` in Nginx

### The Private Key

**Example filename:** `your_domain.key`

The most sensitive file in the set.
- Generated when the certificate was requested
- Used by the server to encrypt and decrypt HTTPS traffic
- Must always match the certificate — a mismatch causes the server to fail
- Must **never** be shared publicly or committed to version control

### Summary Table

| File | Role | Used In Nginx As |
|---|---|---|
| `certificate.pem` | Your domain certificate | `ssl_certificate` |
| `your_domain.key` | Private key for encryption | `ssl_certificate_key` |
| `ca_bundle.pem` | CA trust chain | `ssl_trusted_certificate` |

---

## How the Three Files Work Together

```
Step 1: Browser connects to https://app.yourdomain.com

Step 2: Nginx presents the SSL certificate (.pem)
        → Browser reads: "Who issued this certificate?"

Step 3: Nginx provides the CA bundle
        → Browser verifies: "I trust this certificate authority"

Step 4: Private key (.key) is used on the server side
        → Encrypts the session so data cannot be intercepted

Step 5: Secure connection established — padlock appears in browser
```

The certificate is the **public** part (can be shared). The private key is the **secret** part — it must stay on the server only.

---

## Step-by-Step Configuration in Nginx

### Step 1 — Upload Certificate Files to the Server

Transfer all three files from your local machine to the EC2 server using SCP, WinSCP, or MobaXterm's file browser. Store them in a dedicated directory:

```bash
sudo mkdir -p /etc/nginx/ssl
```

Place the following files there:
- `certificate.pem`
- `your_domain.key`
- `ca_bundle.pem`

---

### Step 2 — Open the Nginx Config File

```bash
sudo nano /etc/nginx/sites-available/myapp
```

---

### Step 3 — Add the HTTPS Server Block

Replace `8085` with your app's actual internal port:

```nginx
server {
    listen 443 ssl;
    server_name app.yourdomain.com;

    ssl_certificate         /etc/nginx/ssl/certificate.pem;
    ssl_certificate_key     /etc/nginx/ssl/your_domain.key;
    ssl_trusted_certificate /etc/nginx/ssl/ca_bundle.pem;

    location / {
        proxy_pass http://localhost:8085;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

**What each directive does:**

| Directive | Purpose |
|---|---|
| `listen 443 ssl` | Accept HTTPS traffic on port 443 |
| `server_name` | The domain this block applies to |
| `ssl_certificate` | Path to the `.pem` certificate file |
| `ssl_certificate_key` | Path to the private key file |
| `ssl_trusted_certificate` | Path to the CA bundle for chain verification |
| `proxy_pass` | Forward requests to your app's internal port |

---

### Step 4 — Add HTTP to HTTPS Redirect

Add this second server block to automatically redirect all HTTP traffic to HTTPS:

```nginx
server {
    listen 80;
    server_name app.yourdomain.com;

    return 301 https://$host$request_uri;
}
```

This is a **permanent redirect (301)** — any browser visiting `http://` is automatically sent to `https://`.

---

### Step 5 — Test and Restart Nginx

```bash
sudo nginx -t
```

Expected:
```
nginx: configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

Then restart:
```bash
sudo systemctl restart nginx
```

---

### Step 6 — Verify in Browser

```
https://app.yourdomain.com
```

A padlock icon in the address bar confirms HTTPS is working correctly.

---

## Security Considerations

**Restrict private key permissions** — set this immediately after uploading:
```bash
chmod 600 /etc/nginx/ssl/your_domain.key
```
This ensures only the root user can read the file.

**Never commit the key to version control** — if the `.key` file is pushed to Git, anyone with access can decrypt your traffic or impersonate your server.

**Ensure port 443 is open** in the EC2 Security Group:
- Port 443 (HTTPS) — inbound
- Port 80 (HTTP) — inbound (for the redirect rule)

> See [01_ec2_guide.md → Security Group Rules](01_ec2_guide.md#security-group-rules).

---

## Common Mistakes to Avoid

| Mistake | Impact | Fix |
|---|---|---|
| Wrong file path in Nginx config | Nginx fails to start | Double-check all paths |
| Certificate and key do not match | SSL handshake fails | Ensure both came from the same certificate request |
| CA bundle not added | Browser shows security warning | Add `ssl_trusted_certificate` |
| Port 443 not open in AWS | Connection refused | Update Security Group inbound rules |
| Key file has wrong permissions | Security risk | Run `chmod 600` on the key file |

---

## Key Takeaways

**This is real production SSL** — exactly how enterprise companies configure HTTPS when using paid certificates from GoDaddy, DigiCert, or Sectigo and the IT team manages certificates centrally.

**The difference from Certbot:**
- Certbot automatically generates, installs, and renews certificates
- Manual SSL — IT team provides the files, you configure Nginx, IT team handles renewal on expiry

**One-line summary:**
The IT team provides three files (certificate + private key + CA bundle). Place them on the server, reference them in Nginx, and HTTPS is live.
