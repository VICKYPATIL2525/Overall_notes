# EC2 Complete Guide

**Purpose:** Everything you need to launch and manage an EC2 instance on AWS — concepts, instance creation, security groups, and elastic IP.

> **Next Step:** Once your EC2 instance is running:
> - Connect to it via SSH — see [02_mobaxterm_ssh_guide.md](02_mobaxterm_ssh_guide.md)
> - Set up your app as a background service — see [01_systemd_app_service.md](../01_systemd_app_service.md)
> - Expose it via Nginx — see [02_nginx_setup.md](../02_nginx_setup.md)

---

## Table of Contents

1. [What is EC2?](#what-is-ec2)
2. [Why Use EC2?](#why-use-ec2)
3. [Key Concepts](#key-concepts)
4. [Creating an EC2 Instance](#creating-an-ec2-instance)
5. [Instance States](#instance-states)
6. [Elastic IP Setup](#elastic-ip-setup)
7. [Security Group Rules](#security-group-rules)
8. [Common Use Cases](#common-use-cases)

---

## What is EC2?

Amazon EC2 (Elastic Compute Cloud) is a service that lets you rent virtual computers in the cloud. Instead of buying physical hardware, you spin up a virtual server on AWS in minutes, run your applications on it, and pay only for the time it runs.

Think of it as **your own Linux server in the cloud** — you get full root access, install anything you want, and control it completely over SSH.

---

## Why Use EC2?

- **No hardware to manage** — AWS handles all underlying infrastructure
- **Scalable on demand** — increase or decrease size at any time
- **Pay only for what you use** — stop the instance and stop being charged for compute
- **Full control** — root access, install anything, configure freely
- **Global availability** — launch in any AWS region worldwide

---

## Key Concepts

**AMI (Amazon Machine Image)**
A pre-built OS template. Think of it as the installer disc. Ubuntu Server LTS is the most common choice for web apps and ML workloads.

**Instance Type**
The hardware spec of the VM. Format: `family.size` e.g. `t2.micro`, `t3.medium`.
- `t2.micro` — 1 vCPU, 1GB RAM — free tier eligible, good for light apps
- `t3.medium` — 2 vCPU, 4GB RAM — good for medium workloads
- `t3.large` — 2 vCPU, 8GB RAM — good for ML apps

**Key Pair**
SSH authentication keys. AWS stores the public key on the server, you keep the `.pem` private key file. Generated once — cannot be re-downloaded. If lost, you lose SSH access.

> For full detail on PEM files and SSH connection — see [02_mobaxterm_ssh_guide.md](02_mobaxterm_ssh_guide.md).

**Security Group**
Virtual firewall controlling inbound/outbound traffic by port and IP. You define which ports are open and to whom.

**Elastic IP**
A static public IP that doesn't change on restart. Without it, your EC2 gets a new IP every time it reboots — breaking saved connections and DNS records. Free while associated with a running instance.

**EBS (Elastic Block Store)**
The hard disk attached to the instance. Persists independently — data survives stops and restarts. Lost only on termination (unless configured otherwise).

**Region / Availability Zone**
Region is a geographic location (e.g. `ap-south-1` = Mumbai). Each region has multiple Availability Zones (isolated data centres). Choose the region closest to your users.

**VPC (Virtual Private Cloud)**
Virtual private network. Use the default VPC for basic setups.

---

## Creating an EC2 Instance

Go to AWS Console → EC2 → click **Launch Instance**.

### Every Field Explained

| Field | What It Is | What to Select |
|---|---|---|
| Name and Tags | Label for the instance | Any meaningful name |
| AMI | OS template | Ubuntu Server (latest LTS) |
| Instance Type | CPU + RAM config | `t2.micro` for free tier |
| Key Pair | SSH auth key | Create new → RSA → `.pem` format |
| VPC | Virtual network | Default VPC |
| Subnet | Sub-network | Default |
| Auto-assign Public IP | Public internet access | Enable |
| Security Group | Firewall rules | Add SSH port 22 at minimum |
| Storage | EBS disk size | 8GB default, increase as needed |

### Step by Step

**Step 01 —** Log in to AWS, search for EC2, open the dashboard.

**Step 02 —** Click the orange **Launch Instance** button.

**Step 03 —** Set Name, choose Ubuntu Server AMI, select instance type.

**Step 04 — Create the Key Pair**
Select **Create new key pair** → RSA type → `.pem` format → give it a name → click **Create key pair**.
The `.pem` file downloads automatically. **This is downloaded once — store it safely.** Never commit it to version control.

Store it in a dedicated folder: `C:\Users\your-name\aws-keys\`

**Step 05 —** Configure Security Group — add port 22 (SSH) at minimum.

**Step 06 —** Review and click **Launch Instance**. Wait for status to show **Running**.

**Step 07 —** Copy the **Public IPv4 address** from the instance details panel.

> Now connect to your instance — see [02_mobaxterm_ssh_guide.md](02_mobaxterm_ssh_guide.md).

---

## Instance States

| State | Meaning |
|---|---|
| Running | Active, accessible, being charged for compute |
| Stopped | Shut down, storage preserved, not charged for compute |
| Terminated | Permanently deleted, all data lost, irreversible |

> **Important:** Stopping an instance preserves your data (EBS). Terminating deletes everything. Always stop, never terminate unless you are certain.

---

## Elastic IP Setup

Without an Elastic IP, your EC2 instance gets a new public IP every time it restarts. This breaks:
- Saved MobaXterm sessions
- DNS records pointing to your domain
- Any hardcoded IP references

**Step 01 —** Go to **EC2 → Elastic IPs** → click **Allocate Elastic IP address** → confirm

**Step 02 —** Select the new IP → **Actions → Associate Elastic IP address** → select your instance → confirm

> Elastic IP is **free while associated with a running instance**. If you stop the instance or release the association without releasing the IP, you will be charged. Always release it when you no longer need it.

---

## Security Group Rules

Security Groups are the firewall for your EC2 instance. You add **inbound rules** to allow traffic on specific ports.

| Port | Protocol | Use |
|---|---|---|
| 22 | TCP | SSH access — to connect via MobaXterm or terminal |
| 80 | TCP | HTTP — for Nginx to receive web traffic |
| 443 | TCP | HTTPS — for SSL/TLS encrypted web traffic |
| 8085 | TCP | Custom app port — only if not using Nginx |

> **Best practice:** For port 22, restrict the source to **My IP** only — not `0.0.0.0/0`. This prevents unauthorized SSH attempts from the entire internet.

> If you are using Nginx as a reverse proxy (recommended), you only need ports 22, 80, and 443 open. Your app's internal port does not need to be exposed — see [02_nginx_setup.md](../02_nginx_setup.md).

---

## Common Use Cases

- Hosting web applications or API servers
- Running Python/ML apps (FastAPI, Flask, Streamlit, Gradio)
- Running background jobs or scheduled scripts
- Hosting a database (PostgreSQL, MySQL, MongoDB)
- Running Docker containers
- Acting as a VPN or proxy server
