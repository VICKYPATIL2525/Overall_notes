# EC2 PEM File & MobaXterm Setup

**Platform:** AWS EC2 / Ubuntu  
**Category:** Cloud Infrastructure / Remote Access  
**Type:** Personal Reference

---

## Objective

Launch an AWS EC2 Ubuntu instance, create and download the PEM key pair during setup, store it safely, and use it in MobaXterm to establish a secure SSH connection to the server.

---

## Overview

**What is EC2**  
Amazon EC2 is a virtual server provided by AWS. It runs in the cloud and can host applications, APIs, or any server-side workload. You access it remotely over SSH using a key pair for authentication.

**What is a PEM File**  
A PEM file is a private key file AWS generates when you create a key pair during EC2 setup. It is downloaded once and never shown again. It acts as your password to SSH into the server.

**What is MobaXterm**  
MobaXterm is a Windows application that provides an SSH client, terminal, and file browser in one tool. It supports `.pem` files directly — no conversion needed.

**PEM vs PPK**  
AWS generates keys in `.pem` format. PuTTY requires `.ppk` format (convert using PuTTYgen). MobaXterm supports `.pem` natively.

**Static vs Dynamic IP**  
By default EC2 gets a new public IP on every restart. An Elastic IP is a permanent static IP that never changes across reboots.

---

## Tools & Technologies

AWS EC2 · Ubuntu · PEM Key Pair · MobaXterm · SSH · Elastic IP

---

## Prerequisites

| Requirement | Detail |
|---|---|
| AWS Account | Active account with EC2 access permissions |
| MobaXterm | Installed on your Windows machine |
| PEM File | Downloaded during EC2 instance creation |
| EC2 Instance | Running Ubuntu, port 22 open in Security Group |

---

## Part 1 — Creating the EC2 Instance & Key Pair

**Step 01 — Open EC2 in the AWS Console**  
Log in to AWS, search for EC2, and open the dashboard.

**Step 02 — Launch a New Instance**  
Click the orange **Launch Instance** button.

**Step 03 — Configure the Instance**

| Setting | Value |
|---|---|
| Name | Any meaningful name |
| AMI | Ubuntu Server (latest LTS) |
| Instance Type | `t2.micro` for free tier |
| Storage | 8GB default or as needed |

**Step 04 — Create the Key Pair**  
Select **Create new key pair**, choose RSA type, select `.pem` format, give it a name, and click **Create key pair**. The file downloads automatically. This is downloaded once — store it safely.

**Step 05 — Configure the Security Group**

| Rule | Port | Source |
|---|---|---|
| SSH | 22 | Your IP (recommended) |
| Custom TCP | 8501 | Your IP (for Streamlit) |

**Step 06 — Launch the Instance**  
Review settings and click **Launch Instance**. Wait for status to show **Running**.

**Step 07 — Store the PEM File**  
Move the downloaded PEM file to a dedicated folder e.g. `C:\Users\vicky\aws-keys\`. Never commit it to version control.

**Step 08 — Copy the Public IP**  
In the EC2 dashboard, click your instance and copy the **Public IPv4 address**.

---

## Part 2 — Connecting via MobaXterm

**Step 01 — Open MobaXterm and click Session** (top-left corner)

**Step 02 — Select SSH** as the session type (first option)

**Step 03 — Enter connection details**

| Field | Value |
|---|---|
| Remote Host | Public IPv4 of your EC2 instance |
| Specify Username | Check the box, enter `ubuntu` |
| Port | 22 |

**Step 04 — Attach the PEM File**
1. Click the **Advanced SSH settings** tab
2. Check **Use private key**
3. Click the folder icon and navigate to your PEM file
4. Select the file — leave all other settings as default

**Step 05 — Click OK**  
MobaXterm connects. On first connection, accept the fingerprint prompt. Success looks like: `ubuntu@ip-xxx-xxx-xxx-xxx:~$`

**Step 06 — Reconnecting**  
The session is saved in the left sidebar. Double-click to reconnect. If the IP changed, right-click → Edit session → update the Remote Host field.

---

## Optional — Elastic IP Setup

**Step 01 —** Go to **EC2 → Elastic IPs** → click **Allocate Elastic IP address**

**Step 02 —** Select the IP → **Actions → Associate Elastic IP address** → select your instance → confirm

> Elastic IP is free while associated with a running instance. Release it when not in use to avoid charges.

---

## Useful Commands Once Connected

| Action | Command |
|---|---|
| Check current directory | `pwd` |
| List files | `ls -la` |
| Check logged in user | `whoami` |
| Update server packages | `sudo apt update && sudo apt upgrade -y` |
| Check running services | `sudo systemctl list-units --type=service --state=running` |

---

## What This Setup Achieves

- EC2 Ubuntu instance created and running on AWS
- PEM key pair downloaded and stored securely
- MobaXterm session configured with IP, username, and key
- Secure SSH connection established to the remote server
- Session saved in MobaXterm for quick reconnection
