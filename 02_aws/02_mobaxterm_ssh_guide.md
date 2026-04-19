# MobaXterm & SSH Connection Guide

**Purpose:** Connect to your AWS EC2 instance securely using a PEM key file — via MobaXterm (Windows) or terminal (Mac/Linux).

> **Prerequisite:** You need a running EC2 instance and a `.pem` key file downloaded during instance creation — see [01_ec2_guide.md](01_ec2_guide.md).

---

## Table of Contents

1. [What is a PEM File?](#what-is-a-pem-file)
2. [What is MobaXterm?](#what-is-mobaxterm)
3. [PEM vs PPK](#pem-vs-ppk)
4. [Static vs Dynamic IP](#static-vs-dynamic-ip)
5. [Connecting via MobaXterm](#connecting-via-mobaxterm)
6. [Connecting via SSH Terminal](#connecting-via-ssh-terminal)
7. [Reconnecting After IP Change](#reconnecting-after-ip-change)
8. [Useful Commands Once Connected](#useful-commands-once-connected)
9. [What This Setup Achieves](#what-this-setup-achieves)

---

## What is a PEM File?

A PEM file is a **private key** that AWS generates when you create a key pair during EC2 setup.

- Downloaded **once** during EC2 instance creation — never shown again
- Acts as your password to SSH into the server
- `.pem` = Privacy Enhanced Mail — standard text-based key format used in Linux
- Content is Base64 encoded between `-----BEGIN RSA PRIVATE KEY-----` markers

**Store it safely:** Move the downloaded PEM file to a dedicated folder on your machine:
```
C:\Users\your-name\aws-keys\
```

**Never:**
- Commit it to Git or any version control
- Share it over email or chat
- Leave it in your Downloads folder

**If you lose the PEM file:**
You cannot recover it from AWS. You must create a new key pair and either launch a new instance or manually replace the public key on the existing server.

---

## What is MobaXterm?

MobaXterm is a Windows application that provides:
- SSH client (connect to remote servers)
- Built-in terminal
- File browser (drag and drop files to/from server)
- Supports `.pem` files directly — no conversion needed

Download: [mobaxterm.mobatek.net](https://mobaxterm.mobatek.net)

---

## PEM vs PPK

| Format | Used By | Notes |
|---|---|---|
| `.pem` | MobaXterm, Mac/Linux terminal, AWS default | No conversion needed |
| `.ppk` | PuTTY (Windows) | Convert from `.pem` using PuTTYgen |

MobaXterm supports `.pem` natively — no conversion needed.

---

## Static vs Dynamic IP

By default, EC2 gets a **new public IP every time it restarts**. This breaks saved MobaXterm sessions and DNS records.

An **Elastic IP** is a permanent static IP that never changes across reboots — assign one to avoid this.

> See [01_ec2_guide.md → Elastic IP Setup](01_ec2_guide.md#elastic-ip-setup) for setup steps.

---

## Connecting via MobaXterm

**Step 01 —** Open MobaXterm → click **Session** (top-left corner)

**Step 02 —** Select **SSH** as the session type (first option)

**Step 03 —** Enter connection details:

| Field | Value |
|---|---|
| Remote Host | Public IPv4 of your EC2 instance |
| Specify Username | Check the box → enter `ubuntu` |
| Port | 22 |

**Step 04 — Attach the PEM File**
1. Click the **Advanced SSH settings** tab
2. Check **Use private key**
3. Click the folder icon → navigate to your `.pem` file → select it
4. Leave all other settings as default

**Step 05 —** Click **OK**

On first connection, MobaXterm will show a fingerprint prompt — click **Accept**.

Success looks like:
```
ubuntu@ip-xxx-xxx-xxx-xxx:~$
```

**Step 06 —** The session is saved in the left sidebar — double-click to reconnect anytime.

---

## Connecting via SSH Terminal

If you are on Mac/Linux or prefer the terminal:

```bash
# Step 1 — Set correct permissions on PEM file (required)
# SSH refuses to use a key that is readable by others
chmod 400 /path/to/your-key.pem

# Step 2 — Connect
ssh -i /path/to/your-key.pem ubuntu@your-ec2-public-ip
```

Example:
```bash
chmod 400 ~/aws-keys/mykey.pem
ssh -i ~/aws-keys/mykey.pem ubuntu@54.123.45.67
```

On first connection, type `yes` to accept the fingerprint.

---

## Reconnecting After IP Change

If your instance restarted and got a new IP (no Elastic IP set up):

**MobaXterm:**
Right-click the saved session in the sidebar → **Edit session** → update the **Remote Host** field with the new IP → click OK → reconnect.

**Terminal:**
Just run the SSH command again with the new IP.

> To avoid this permanently — set up an Elastic IP. See [01_ec2_guide.md](01_ec2_guide.md#elastic-ip-setup).

---

## Useful Commands Once Connected

| Action | Command |
|---|---|
| Check current directory | `pwd` |
| List files | `ls -la` |
| Check logged in user | `whoami` |
| Update server packages | `sudo apt update && sudo apt upgrade -y` |
| Check disk usage | `df -h` |
| Check memory usage | `free -h` |
| Check running services | `sudo systemctl list-units --type=service --state=running` |

---

## What This Setup Achieves

- Secure SSH connection to your EC2 instance using PEM key authentication
- MobaXterm session saved for quick reconnection
- File browser available to drag and drop files to/from the server
- Ready to install packages, deploy apps, and manage the server
