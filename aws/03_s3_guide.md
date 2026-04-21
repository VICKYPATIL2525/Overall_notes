# S3 Complete Guide

**Purpose:** Everything you need to understand, set up, and use Amazon S3 — from bucket creation to access control to integrating S3 with your EC2 app.

> **Related:** For connecting S3 to an app running on EC2 via IAM role — see [01_ec2_guide.md](01_ec2_guide.md).

---

## Table of Contents

1. [What is S3?](#what-is-s3)
2. [Why Use S3?](#why-use-s3)
3. [Key Concepts](#key-concepts)
4. [How S3 is Organised](#how-s3-is-organised)
5. [Creating an S3 Bucket](#creating-an-s3-bucket)
6. [Storage Classes — When to Use Which](#storage-classes--when-to-use-which)
7. [Access Control](#access-control)
   - [Bucket Policy](#bucket-policy)
   - [IAM Roles](#iam-roles)
   - [Presigned URLs](#presigned-urls)
   - [ACLs](#acls)
8. [When to Make Things Public vs Private](#when-to-make-things-public-vs-private)
9. [AWS CLI Commands for S3](#aws-cli-commands-for-s3)
10. [Using S3 from Your EC2 App (Python)](#using-s3-from-your-ec2-app-python)
11. [EC2 + S3 Together](#ec2--s3-together)
12. [EC2 vs S3 — When to Use Which](#ec2-vs-s3--when-to-use-which)
13. [Common Use Cases](#common-use-cases)

---

## What is S3?

Amazon S3 (Simple Storage Service) is AWS's cloud object storage. Store any file — documents, images, datasets, model files, videos, backups — and access them from anywhere via a URL, CLI, or SDK.

Unlike EBS (the disk attached to EC2), S3 is **completely independent of any server**. The files exist on their own and can be accessed by any service, any app, from anywhere.

---

## Why Use S3?

- **Unlimited storage** — no cap on how much you store
- **Extremely durable** — 99.999999999% (11 nines) durability — files are automatically replicated across multiple data centres
- **Accessible anywhere** — via URL, AWS Console, CLI, or SDK
- **Cost effective** — much cheaper than EBS for file storage
- **Works with every AWS service** — EC2, Lambda, SageMaker, CloudFront, and more
- **Fine-grained access control** — control exactly who can read, write, or delete

---

## Key Concepts

**Bucket**
Top-level container for your files. Like a root folder. The name must be globally unique across all AWS accounts worldwide. Belongs to one region.

**Object**
Any file stored in S3. Can be up to 5TB per object. Each object has a key, data, and metadata.

**Object Key**
The unique path/identifier for the object — e.g. `data/2024/report.csv`. S3 is flat (no real folders) but the `/` in keys creates the appearance of a folder structure.

**Bucket Policy**
A JSON document defining access permissions for the entire bucket — who can read, write, or delete.

**IAM Role**
Grants an AWS service (like EC2) permission to access S3 without hardcoding credentials. This is the correct and secure way to give your EC2 app access to S3.

**Presigned URL**
A temporary time-limited URL that grants access to a specific private object. Useful for sharing files securely without making them permanently public.

**Versioning**
Keeps all versions of every object. If you overwrite or delete a file, the old version is retained and can be restored. Enable for important or irreplaceable data.

**Access Control List (ACL)**
Object-level permissions. Mostly legacy — bucket policies are preferred for modern setups.

---

## How S3 is Organised

```
AWS Account
    └── Bucket: my-project-data          (globally unique name)
            ├── models/
            │     └── model_v1.pkl
            ├── data/
            │     ├── train.csv
            │     └── test.csv
            └── uploads/
                  └── user_file.pdf
```

Access URL format:
```
https://bucket-name.s3.region.amazonaws.com/object-key

Example:
https://my-project-data.s3.ap-south-1.amazonaws.com/models/model_v1.pkl
```

---

## Creating an S3 Bucket

Go to AWS Console → S3 → click **Create bucket**.

### Every Field Explained

**Bucket Name**
- Must be globally unique across all AWS accounts
- Lowercase only, 3–63 characters, hyphens allowed
- Cannot be changed after creation
- Example: `my-project-data-2025`

**AWS Region**
- Choose the same region as your EC2 instance to avoid cross-region transfer costs
- Example: `ap-south-1` (Mumbai)

**Object Ownership**
- Leave as **ACLs disabled** (recommended) for most use cases
- Means bucket policies and IAM control all access

**Block Public Access**
- All 4 options checked by default = fully private bucket
- Only uncheck if you specifically need public access (e.g. hosting static files)

**Bucket Versioning**
- Enable for important data you cannot regenerate
- Leave disabled for temp files, logs, or anything easily recreated

**Default Encryption**
- Leave as **SSE-S3** (AWS-managed keys) unless you have compliance requirements

---

## Storage Classes — When to Use Which

| Storage Class | Use When | Cost |
|---|---|---|
| **Standard** | Files accessed frequently (daily/weekly) | Highest |
| **Standard-IA** (Infrequent Access) | Files accessed monthly — backups, old datasets | Lower storage, higher retrieval |
| **Glacier Instant Retrieval** | Archival data accessed a few times a year | Very low storage |
| **Glacier Deep Archive** | Long-term archival — rarely accessed | Cheapest |
| **Intelligent-Tiering** | Unknown or changing access patterns — AWS auto-moves between tiers | Adds monitoring fee |

**For most projects:** Use **Standard** for active files and **Standard-IA** for older backups.

---

## Access Control

### Bucket Policy

A JSON document attached to the bucket that defines who can do what.

**Example — make a specific folder publicly readable:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::my-bucket/public/*"
    }
  ]
}
```

**Example — allow only your EC2 instance role to access:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::YOUR_ACCOUNT_ID:role/EC2S3AccessRole"
      },
      "Action": ["s3:GetObject", "s3:PutObject"],
      "Resource": "arn:aws:s3:::my-bucket/*"
    }
  ]
}
```

---

### IAM Roles

The correct way to give your EC2 app access to S3 — **no hardcoded credentials**.

**Setup:**
1. Go to **IAM → Roles → Create Role**
2. Trusted entity: **EC2**
3. Attach policy: `AmazonS3FullAccess` (or a custom restricted policy)
4. Give the role a name → create
5. Go to **EC2 → your instance → Actions → Security → Modify IAM Role**
6. Select the role you just created → save

Once the role is attached, your app running on that EC2 can access S3 automatically — no access keys needed in code.

---

### Presigned URLs

A presigned URL is a **temporary URL** that grants access to a specific private S3 object for a limited time.

**Use when:**
- You want to share a private file with someone without making it public
- Your app needs to let users download a file directly from S3
- You want uploads to go directly to S3 from the browser (presigned PUT URL)

**Generate via CLI:**
```bash
aws s3 presign s3://my-bucket/data/report.csv --expires-in 3600
```
This generates a URL valid for 1 hour (3600 seconds).

**Generate via Python (boto3):**
```python
import boto3

s3 = boto3.client('s3')
url = s3.generate_presigned_url(
    'get_object',
    Params={'Bucket': 'my-bucket', 'Key': 'data/report.csv'},
    ExpiresIn=3600
)
print(url)
```

---

### ACLs

Object-level permissions. Mostly legacy — prefer bucket policies for new setups. The main use is making a single object public:

```bash
aws s3api put-object-acl --bucket my-bucket --key file.txt --acl public-read
```

---

## When to Make Things Public vs Private

| Situation | Access Type | How |
|---|---|---|
| Static website files (HTML, CSS, JS) | Public | Bucket policy `s3:GetObject` for `*` |
| Shared assets (images, docs for all users) | Public | Bucket policy |
| App data, model files, datasets | Private + IAM Role | IAM Role on EC2 |
| Temporary file sharing | Private + Presigned URL | `generate_presigned_url` |
| Sensitive files (keys, credentials) | Never in S3 | Use AWS Secrets Manager instead |

---

## AWS CLI Commands for S3

First configure the CLI (one time):
```bash
aws configure
# Enter: Access Key ID, Secret Access Key, Region, output format
```

| Action | Command |
|---|---|
| List all buckets | `aws s3 ls` |
| List contents of a bucket | `aws s3 ls s3://bucket-name/` |
| Upload a file | `aws s3 cp file.txt s3://bucket-name/` |
| Upload to a specific folder | `aws s3 cp file.txt s3://bucket-name/folder/` |
| Upload entire folder | `aws s3 cp ./folder s3://bucket-name/folder/ --recursive` |
| Download a file | `aws s3 cp s3://bucket-name/file.txt ./` |
| Download entire folder | `aws s3 cp s3://bucket-name/folder/ ./local-folder/ --recursive` |
| Delete a file | `aws s3 rm s3://bucket-name/file.txt` |
| Sync local folder to S3 | `aws s3 sync ./folder s3://bucket-name/folder/` |
| Generate presigned URL | `aws s3 presign s3://bucket-name/file.txt --expires-in 3600` |

---

## Using S3 from Your EC2 App (Python)

Install boto3:
```bash
pip install boto3
```

If your EC2 has an IAM role attached, no credentials needed in code:

```python
import boto3

s3 = boto3.client('s3', region_name='ap-south-1')

# Upload a file
s3.upload_file('local_file.csv', 'my-bucket', 'data/local_file.csv')

# Download a file
s3.download_file('my-bucket', 'data/local_file.csv', 'downloaded_file.csv')

# Read a file directly into memory
response = s3.get_object(Bucket='my-bucket', Key='data/local_file.csv')
content = response['Body'].read().decode('utf-8')

# List objects in a folder
response = s3.list_objects_v2(Bucket='my-bucket', Prefix='data/')
for obj in response.get('Contents', []):
    print(obj['Key'])
```

---

## EC2 + S3 Together

The most common pattern — your app runs on EC2 and stores/retrieves files from S3.

```
User → EC2 (runs your app) → S3 (stores files)
```

**Setup checklist:**
1. Create an IAM role with S3 access
2. Attach the role to your EC2 instance
3. Use `boto3` in your app — no credentials needed in code
4. Your app can now read and write to S3 freely

**Why not hardcode AWS credentials in code?**
- Credentials in code get committed to Git accidentally
- Credentials are visible to anyone with server access
- IAM roles are automatically rotated by AWS — zero management
- IAM roles can be scoped to exactly what the app needs

---

## EC2 vs S3 — When to Use Which

| Need | Use |
|---|---|
| Run an application or script | EC2 |
| Store and retrieve files | S3 |
| Temporary working storage during processing | EC2 EBS |
| Long-term persistent file storage | S3 |
| Share files between multiple servers | S3 |
| Host a static website | S3 |
| Store large datasets cheaply | S3 |
| Store ML model files | S3 (load to EC2 at startup) |
| Store database files | EC2 EBS or RDS |
| Store secrets or credentials | AWS Secrets Manager (not S3) |

---

## Common Use Cases

- **ML model storage** — Store trained model files in S3, load them to EC2 at app startup
- **User file uploads** — App on EC2 receives uploads and stores them in S3
- **Dataset storage** — Keep training/test datasets in S3, pull to EC2 for processing
- **Static website hosting** — Host HTML/CSS/JS files directly from S3 with a public bucket
- **Log storage** — App on EC2 writes logs to S3 for long-term retention
- **Backup storage** — Automated backups of databases or config files pushed to S3
- **File sharing between servers** — Multiple EC2 instances read/write a shared S3 bucket
