# Overall Notes

A personal knowledge base covering cloud infrastructure, Linux deployment, automation, and AI frameworks.

---

## AWS

| File | Description |
|---|---|
| [01_ec2_guide.md](aws/01_ec2_guide.md) | How to create an EC2 instance on AWS, understand key concepts like AMI, instance types, security groups, and elastic IP, and manage instance states. |
| [02_mobaxterm_ssh_guide.md](aws/02_mobaxterm_ssh_guide.md) | How to connect to your EC2 server using a PEM file through MobaXterm (Windows) or SSH terminal (Mac/Linux). |
| [03_s3_guide.md](aws/03_s3_guide.md) | How to create S3 buckets, control access using IAM roles and policies, upload/download files via CLI and Python boto3. |
| [04_ssl_https_setup.md](aws/04_ssl_https_setup.md) | How to set up HTTPS on your server using SSL certificate files provided by the IT team, without Certbot. |

---

## Linux Deployment

| File | Description |
|---|---|
| [01_systemd_app_service.md](01_systemd_app_service.md) | How to run any app as a background service that auto-starts on reboot and auto-restarts on crash. |
| [02_nginx_setup.md](02_nginx_setup.md) | How to install Nginx, understand where config files live, and write the server block to accept and route traffic. |
| [03_nginx_reverse_proxy.md](03_nginx_reverse_proxy.md) | How to forward traffic from port 80 to your app's internal port so users never see the port number in the URL. |

---

## Automation

| File | Description |
|---|---|
| [n8n-automation/](n8n-automation/) | How to run n8n using Docker, expose it publicly using Ngrok, and connect Google services like Gmail and Sheets via OAuth. |

---

## Other

| File | Description |
|---|---|
| [05_git_clone_private_repo.md](05_git_clone_private_repo.md) | How to clone a private GitHub repository using a Personal Access Token for both main and specific branches. |
