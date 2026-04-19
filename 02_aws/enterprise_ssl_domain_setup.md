# LEARNING DOCUMENT

# Enterprise SSL Certificate Setup on AWS EC2 with Nginx

*Understanding SSL Certificate Files Provided by IT Team
and Configuring HTTPS on a Production Server*

*A Personal Reference for Setting Up Domain-Based HTTPS
without Let's Encrypt / Certbot*

---

## Table of Contents

1. Introduction .................................................. 2
2. Background: How I Encountered This .............................. 2
3. Understanding the SSL Certificate Files ......................... 3
4. How the Three Files Work Together ............................... 5
5. Step-by-Step Configuration in Nginx ............................. 6
6. Security Considerations ......................................... 9
7. Common Mistakes to Avoid ........................................ 9
8. Key Takeaways ................................................... 10

---

## 1. Introduction

In enterprise and company environments, SSL certificates are not always generated
automatically using tools like Let's Encrypt or Certbot. Instead, the IT or DevOps
team provides a set of certificate files that have been issued and signed by a trusted
Certificate Authority (CA) — often a paid provider like GoDaddy, DigiCert, or
Sectigo.

When you receive these files, the setup is referred to as a **manual SSL
configuration**. This is how real production environments typically work when
organizations maintain control over their own certificates and domain names.

This document captures what I learned when the IT team assigned a domain and
provided a ZIP file containing the necessary SSL certificate files, and how to use
those files to enable HTTPS on an AWS EC2 instance running Nginx.

---

## 2. Background: How I Encountered This

The process began when I requested the IT team to assign a domain name to a
specific application running on an EC2 instance. Instead of just providing a domain,
they sent a ZIP file containing multiple files — which was initially confusing.

The ZIP file contained:

- A certificate file
- A `.pem` formatted version of the certificate
- A CA bundle file
- A private key file

Understanding what each file does and how to use them together was the core
learning of this entire exercise.

---

## 3. Understanding the SSL Certificate Files

The ZIP file from the IT team typically contains four types of files. Each serves a
distinct purpose in the HTTPS handshake process.

### 3.1 The Main Certificate File

**Example filename:** `44eebe8b532eded4`

This is the SSL certificate issued for your domain. It:

- Proves that your domain is legitimate and verified
- Is issued and signed by a Certificate Authority (CA)
- Contains information like the domain name, validity period, and issuer

Think of this as your **identity card** for the domain.

### 3.2 The PEM Format Certificate

**Example filename:** `44eebe8b532eded4.pem`

This is the same certificate as above but saved in `.pem` format.

- `.pem` stands for Privacy Enhanced Mail — it is the standard text-based format
  used in Linux environments
- Nginx and most Linux-based servers expect certificates in this format
- The content is Base64 encoded and wrapped between `-----BEGIN CERTIFICATE-----`
  and `-----END CERTIFICATE-----` markers

This is the file you will reference in your Nginx configuration.

### 3.3 The CA Bundle (Chain Certificate)

**Example filename:** `gd_bundle-g2`

This file is provided by the Certificate Authority (in this case, GoDaddy — `gd`
refers to GoDaddy).

- It contains the **intermediate certificates** that form a trust chain between
  your certificate and the root CA
- Browsers use this to verify: *"I trust the CA that signed this certificate"*
- Without this file, some browsers may show a **security warning** even though
  your certificate is technically valid

This file is referenced as the `ssl_trusted_certificate` in Nginx.

### 3.4 The Private Key

**Example filename:** `inadev_wildcard_2025.key`

This is the most sensitive file in the entire set.

- It is the **private key** that was generated when the certificate was requested
- It is used by the server to **encrypt and decrypt** HTTPS traffic
- It must **always match** the certificate — a mismatched key will cause the
  server to fail
- It must **never be shared publicly** or committed to version control

This file must be stored on the server with restricted permissions.

### 3.5 Summary Table

| File | Role | Used In Nginx As |
|------|------|-----------------|
| `44eebe8b532eded4.pem` | Your domain certificate | `ssl_certificate` |
| `inadev_wildcard_2025.key` | Private key for encryption | `ssl_certificate_key` |
| `gd_bundle-g2` | CA trust chain | `ssl_trusted_certificate` |

---

## 4. How the Three Files Work Together

When a browser connects to your domain over HTTPS, the following process happens:

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

The certificate and private key are mathematically linked. The certificate is the
**public** part (can be shared), and the private key is the **secret** part (must
stay on the server only).

---

## 5. Step-by-Step Configuration in Nginx

Once you have the SSL files from the IT team, follow these steps to configure HTTPS
on your EC2 instance.

### 5.1 Upload the Certificate Files to the Server

Transfer all three files from your local machine to the EC2 server using SCP or
WinSCP. Store them in a dedicated directory:

```
/etc/nginx/ssl/
```

Place the following files there:

- `44eebe8b532eded4.pem`
- `inadev_wildcard_2025.key`
- `gd_bundle-g2`

### 5.2 Open the Nginx Configuration File

```bash
sudo nano /etc/nginx/sites-available/default
```

### 5.3 Add the HTTPS Server Block

Add the following configuration to enable HTTPS on port 443:

```nginx
server {
    listen 443 ssl;
    server_name app.yourdomain.com;

    ssl_certificate     /etc/nginx/ssl/44eebe8b532eded4.pem;
    ssl_certificate_key /etc/nginx/ssl/inadev_wildcard_2025.key;
    ssl_trusted_certificate /etc/nginx/ssl/gd_bundle-g2;

    location / {
        proxy_pass http://localhost:8501;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
    }
}
```

**What each directive does:**

- `listen 443 ssl` — tells Nginx to listen on port 443 and use SSL
- `server_name` — the domain name this block applies to
- `ssl_certificate` — path to the `.pem` certificate file
- `ssl_certificate_key` — path to the private key file
- `ssl_trusted_certificate` — path to the CA bundle for chain verification
- `proxy_pass` — forwards the request to the actual application running locally
  (port 8501 is used by Streamlit)

### 5.4 Add HTTP to HTTPS Redirect

To ensure all traffic over HTTP (port 80) is automatically redirected to HTTPS,
add this additional server block:

```nginx
server {
    listen 80;
    server_name app.yourdomain.com;

    return 301 https://$host$request_uri;
}
```

This is a **permanent redirect (301)** — any browser that visits `http://` will be
automatically sent to `https://`.

### 5.5 Test and Restart Nginx

Before restarting, always test the configuration for syntax errors:

```bash
sudo nginx -t
```

If the output shows `syntax is ok` and `test is successful`, proceed to restart:

```bash
sudo systemctl restart nginx
```

### 5.6 Verify in Browser

Open the browser and navigate to:

```
https://app.yourdomain.com
```

A padlock icon in the address bar confirms the HTTPS connection is working
correctly.

---

## 6. Security Considerations

### 6.1 Restrict Private Key Permissions

The private key file must never be readable by unauthorized users. Set strict
file permissions immediately after uploading:

```bash
chmod 600 /etc/nginx/ssl/inadev_wildcard_2025.key
```

This ensures only the root user can read the file.

### 6.2 Never Commit the Key to Version Control

The `.key` file must never be pushed to Git or any shared repository. It contains
the private half of your encryption and, if exposed, would allow anyone to
decrypt your traffic or impersonate your server.

### 6.3 Ensure Port 443 is Open

In the AWS Security Group for your EC2 instance, make sure:

- **Port 443 (HTTPS)** is open for inbound traffic
- **Port 80 (HTTP)** is open if you are using the redirect rule

Without this, the browser will not be able to reach your server even if Nginx is
configured correctly.

---

## 7. Common Mistakes to Avoid

| Mistake | Impact | Fix |
|---------|--------|-----|
| Wrong file path in Nginx config | Nginx fails to start | Double-check all paths |
| Certificate and key do not match | SSL handshake fails | Ensure both came from same request |
| CA bundle not added | Browser shows warning | Add `ssl_trusted_certificate` |
| Port 443 not open in AWS | Connection refused | Update Security Group rules |
| Key file has wrong permissions | Security risk | Run `chmod 600` on the key |

---

## 8. Key Takeaways

### 8.1 This is Real Production SSL

This manual certificate setup is exactly how enterprise companies configure
HTTPS when:

- They use paid certificates from providers like GoDaddy, DigiCert, or Sectigo
- They do not use Let's Encrypt or Certbot
- The IT or DevOps team manages certificates centrally for the organization

### 8.2 The Difference from Certbot

With Certbot, the tool automatically generates the certificate, private key, and
handles renewal. With manual SSL:

- The IT team provides the certificate files
- You configure Nginx to point to those files manually
- Renewal is handled by the IT team when the certificate expires

### 8.3 One-Line Summary

The IT team provided a manual SSL certificate (certificate + private key + CA
bundle). These three files are placed on the server and referenced in Nginx to
enable enterprise-grade HTTPS without using any automated certificate tools.

---

*Document prepared by Vicky | April 2026*
*Context: Setting up HTTPS on AWS EC2 using company-provided SSL certificate files*
