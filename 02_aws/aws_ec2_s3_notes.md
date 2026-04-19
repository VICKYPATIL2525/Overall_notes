# AWS EC2 & S3 Notes

**Platform:** Amazon Web Services  
**Services:** EC2 & S3  
**Type:** Personal Reference

---

# Part 1 — Amazon EC2 (Elastic Compute Cloud)

## What is EC2
Amazon EC2 is a service that lets you rent virtual computers in the cloud. Instead of buying physical hardware, you spin up a virtual server on AWS in minutes, run your applications on it, and pay only for the time it runs.

## Why Use EC2
- **No hardware to manage** — AWS handles all underlying infrastructure
- **Scalable on demand** — increase or decrease size at any time
- **Pay only for what you use** — stop the instance and stop being charged
- **Full control** — root access, install anything, configure freely
- **Global availability** — launch in any AWS region worldwide

## Key Concepts

**AMI (Amazon Machine Image)** — A pre-built OS template. Think of it as the installer. Ubuntu Server LTS is the most common choice.

**Instance Type** — Hardware spec of the VM. Format: `family.size` e.g. `t2.micro`, `t3.medium`. `t2.micro` is free tier eligible.

**Key Pair** — SSH authentication keys. AWS stores the public key on the server, you keep the `.pem` private key. Generated once, cannot be re-downloaded.

**Security Group** — Virtual firewall controlling inbound/outbound traffic by port and IP.

**Elastic IP** — Static public IP that doesn't change on restart. Free while associated with a running instance.

**EBS** — The hard disk attached to the instance. Persists independently — data survives stops.

**Region / AZ** — Region is geographic location (e.g. `ap-south-1`). Each region has multiple Availability Zones (isolated data centres).

**VPC** — Virtual private network. Use the default VPC for basic setups.

## Creating an EC2 Instance — Every Field

| Field | What It Is | What to Select |
|---|---|---|
| Name and Tags | Label for the instance | Any meaningful name |
| AMI | OS template | Ubuntu Server (latest LTS) |
| Instance Type | CPU + RAM config | `t2.micro` for free tier |
| Key Pair | SSH auth key | Create new → RSA → `.pem` |
| VPC | Virtual network | Default VPC |
| Subnet | Sub-network | Default |
| Auto-assign Public IP | Public internet access | Enable |
| Security Group | Firewall rules | Add SSH port 22 at minimum |
| Storage | EBS disk size | 8GB default, increase as needed |

## Instance States

| State | Meaning |
|---|---|
| Running | Active, accessible, being charged |
| Stopped | Shut down, storage preserved, not charged for compute |
| Terminated | Permanently deleted, all data lost, irreversible |

## Common Use Cases
- Hosting web applications or API servers
- Running Streamlit or ML apps
- Running background jobs or scripts
- Hosting a database
- Running Docker containers

---

# Part 2 — Amazon S3 (Simple Storage Service)

## What is S3
Amazon S3 is AWS's cloud object storage. Store any file — documents, images, datasets, model files, backups — and access them from anywhere. Unlike EBS, S3 is independent of any specific server.

## Why Use S3
- **Unlimited storage** — no cap on how much you store
- **11 nines durability** — 99.999999999% — files replicated across multiple data centres
- **Accessible anywhere** — via URL, console, CLI, or SDK
- **Cost effective** — much cheaper than EBS for file storage
- **Works with every AWS service** — EC2, Lambda, SageMaker, and more

## How S3 Works
S3 is organised into **buckets** (top-level containers). Inside buckets you store **objects** (files). Every object has a unique **key** (its path). Access URL format: `https://bucket-name.s3.region.amazonaws.com/object-key`

## Key Concepts

**Bucket** — Top-level container. Globally unique name. Belongs to one region.

**Object** — Any file stored in S3. Up to 5TB per object.

**Object Key** — Unique path/identifier for the object e.g. `data/2024/report.csv`

**Bucket Policy** — JSON document defining access permissions for the bucket.

**Public vs Private** — All buckets are private by default. You control what is publicly accessible.

**Storage Classes** — Standard (frequent access), Standard-IA (infrequent), Glacier (archival). Use Standard for most cases.

**Versioning** — Keeps all versions of every object. Enables restore on accidental delete/overwrite.

**Presigned URL** — Temporary time-limited URL for secure access to a private object.

## Creating an S3 Bucket — Every Field

**Bucket Name** — Globally unique, lowercase, 3–63 chars, hyphens allowed. Cannot be changed after creation. Example: `my-project-data-bucket`

**AWS Region** — Choose the same region as your EC2 instance to avoid transfer costs.

**Object Ownership** — Leave as ACLs disabled (recommended) for most use cases.

**Block Public Access** — All checked by default = fully private. Uncheck only if you need public access.

**Bucket Versioning** — Enable for important data. Leave disabled for temp/regeneratable files.

**Default Encryption** — Leave as SSE-S3 (AWS-managed keys) unless you have compliance requirements.

## AWS CLI Commands for S3

| Action | Command |
|---|---|
| Upload file | `aws s3 cp file.txt s3://bucket-name/` |
| Upload folder | `aws s3 cp ./folder s3://bucket-name/folder/ --recursive` |
| List contents | `aws s3 ls s3://bucket-name/` |
| Download file | `aws s3 cp s3://bucket-name/file.txt ./` |
| Delete file | `aws s3 rm s3://bucket-name/file.txt` |

## EC2 + S3 Together

Give EC2 access to S3 via an **IAM Role** — never hardcode credentials.

**Setup:** IAM → Roles → Create Role → EC2 as trusted entity → attach `AmazonS3FullAccess` → create. Then: EC2 instance → Actions → Security → Modify IAM Role → attach role.

## EC2 vs S3 — When to Use Which

| Need | Use |
|---|---|
| Run an application | EC2 |
| Store and retrieve files | S3 |
| Temporary working storage | EC2 EBS |
| Long-term persistent storage | S3 |
| Share files between servers | S3 |
| Host a static website | S3 |
| Store large datasets cheaply | S3 |

## Common Use Cases for S3
- Storing ML model files loaded by EC2 at startup
- Storing user uploaded files from a web app
- Keeping datasets for training or processing
- Hosting static website files
- Storing logs and backups
- Sharing files between multiple EC2 instances
