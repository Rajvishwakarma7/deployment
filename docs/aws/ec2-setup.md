# AWS EC2 Setup

This guide explains how to create and prepare an AWS EC2 instance for hosting a production backend application.

The instructions are intentionally generic so they can be reused for different projects.

---

## Architecture

```text
Developer
    |
    | GitHub
    v
GitHub Actions
    |
    v
AWS EC2
    |
    +-- Docker
    |     |
    |     +-- Node.js API
    |
    +-- Nginx
    |
    +-- HTTPS / SSL
```

---

# 1. AWS Account

Make sure you have an AWS account with permission to create:

- EC2 instances
- Security Groups
- Elastic IPs
- IAM roles
- CloudWatch resources

For production workloads, avoid using the AWS root account for day-to-day operations.

Prefer an IAM user or role with only the permissions required for the task.

---

# 2. Open EC2

Go to:

```text
AWS Console
→ EC2
→ Instances
→ Launch instances
```

---

# 3. Create EC2 Instance

Click:

```text
Launch instance
```

Give the instance a meaningful name.

Example:

```text
<PROJECT_NAME>-production
```

---

# 4. Choose Operating System

Recommended:

```text
Ubuntu Server 24.04 LTS
```

Architecture:

```text
64-bit (x86)
```

Ubuntu works well with:

- Docker
- Nginx
- Certbot
- GitHub Actions
- AWS CloudWatch

---

# 5. Choose Instance Type

For a small or medium Node.js API, start with an appropriate low-cost instance such as:

```text
t3.micro
```

or an equivalent instance available in your AWS region/account.

Choose a larger instance if the application requires more:

- CPU
- RAM
- Concurrent processing
- Background workers

Do not over-provision the server initially.

Monitor the application and scale when required.

---

# 6. Create SSH Key Pair

Under:

```text
Key pair
```

Create a new key pair.

Example:

```text
<PROJECT_NAME>-prod-key
```

Recommended:

```text
Key pair type:
ED25519

Private key format:
.pem
```

Download the `.pem` file and store it securely.

Example local location:

```text
~/Downloads/<PROJECT_NAME>-prod-key.pem
```

---

# 7. Protect the Private Key

On your local machine:

```bash
chmod 400 ~/Downloads/<PROJECT_NAME>-prod-key.pem
```

Verify:

```bash
ls -l ~/Downloads/<PROJECT_NAME>-prod-key.pem
```

The private key must never be:

- Committed to Git
- Uploaded to GitHub
- Shared with other developers
- Stored inside the application repository

---

# 8. Configure Network Settings

When launching the instance, create or select a Security Group.

Initial inbound rules:

| Type | Port | Source |
|---|---:|---|
| SSH | 22 | Your IP only |
| HTTP | 80 | `0.0.0.0/0` |
| HTTPS | 443 | `0.0.0.0/0` |

### SSH

Do not expose SSH to everyone unless there is a specific reason.

Prefer:

```text
22 → My IP
```

This restricts SSH access to your current public IP.

### HTTP

```text
80 → 0.0.0.0/0
```

HTTP needs to be publicly reachable for web traffic and SSL certificate setup.

### HTTPS

```text
443 → 0.0.0.0/0
```

This allows users to access the production API over HTTPS.

---

# 9. Do NOT Expose the Node.js Port

If the Node.js application runs on:

```text
3000
```

do not add:

```text
3000 → 0.0.0.0/0
```

Instead, Nginx should act as the public reverse proxy.

Recommended architecture:

```text
Internet
   |
   | HTTPS :443
   v
 Nginx
   |
   | localhost:3000
   v
Node.js / Docker
```

Docker should preferably bind the application port to localhost:

```yaml
ports:
  - "127.0.0.1:${APP_PORT:-3000}:3000"
```

This prevents direct public access to the Node.js container.

---

# 10. Storage

For a normal Node.js API, the default root EBS volume is usually sufficient.

For example:

```text
20–30 GB
```

Choose the size based on:

- Docker images
- Application files
- Logs
- Uploaded files
- Database requirements

If the application stores large files, consider using Amazon S3 instead of relying on the EC2 root disk.

---

# 11. File System

For a standard backend deployment, the default root file system is sufficient.

There is normally no need to create an additional file system during the initial EC2 setup unless the application has a specific storage requirement.

---

# 12. Launch Instance

Review the configuration:

```text
OS:
Ubuntu 24.04 LTS

Architecture:
64-bit x86

Instance type:
Appropriate instance size

Key pair:
Production SSH key

Security Group:
SSH 22 → Your IP
HTTP 80 → Public
HTTPS 443 → Public

Storage:
Appropriate EBS volume
```

Then click:

```text
Launch instance
```

---

# 13. Find the Public IP

After the instance is running:

```text
EC2
→ Instances
→ Select instance
```

Find:

```text
Public IPv4 address
```

Example:

```text
<EC2_PUBLIC_IP>
```

The public IP may change if the instance is stopped and started.

For production, use an Elastic IP.

See:

```text
docs/aws/elastic-ip.md
```

---

# 14. Connect to EC2

From your local Linux/macOS terminal:

```bash
ssh -i ~/Downloads/<PROJECT_NAME>-prod-key.pem ubuntu@<EC2_PUBLIC_IP>
```

Example:

```bash
ssh -i ~/Downloads/my-project-prod-key.pem ubuntu@203.0.113.10
```

For Ubuntu EC2 instances, the default SSH user is normally:

```text
ubuntu
```

---

# 15. First Connection

The first time you connect, SSH may show:

```text
The authenticity of host '<EC2_PUBLIC_IP>' can't be established.
```

Type:

```text
yes
```

You should then see an Ubuntu shell similar to:

```text
ubuntu@ip-xxx-xxx-xxx-xxx:~$
```

You are now connected to the EC2 server.

---

# 16. Verify the Server

Run:

```bash
whoami
```

Expected:

```text
ubuntu
```

Check the operating system:

```bash
cat /etc/os-release
```

Check the kernel/system:

```bash
uname -a
```

Check disk usage:

```bash
df -h
```

Check memory:

```bash
free -h
```

Check IP information:

```bash
hostname -I
```

---

# 17. Update Ubuntu

After the first login:

```bash
sudo apt update
```

Then:

```bash
sudo apt upgrade -y
```

If Ubuntu reports that a restart is required:

```bash
sudo reboot
```

Wait a few seconds and reconnect using SSH.

---

# 18. Production Deployment Principle

The EC2 instance should be treated as infrastructure.

The application should be deployed using:

```text
GitHub
   |
   v
GitHub Actions
   |
   v
Container Registry
   |
   v
EC2
```

Avoid manually copying application files to the production server for every release.

Docker and GitHub Actions will handle application deployment later.

---

# 19. Final Production Architecture

After completing the remaining deployment steps:

```text
                         Internet
                            |
                            v
                         <DOMAIN>
                            |
                       DNS Provider
                            |
                            v
                    <API_SUBDOMAIN>
                            |
                            v
                    Elastic IP / EC2
                            |
                            v
                          Nginx
                            |
                       HTTPS / SSL
                            |
                            v
                     localhost:3000
                            |
                            v
                     Docker Container
                            |
                            v
                       Node.js API
```

Monitoring will later be added:

```text
EC2
 |
 +-- Docker
 |
 +-- Nginx
 |
 +-- Node.js
 |
 +-- CloudWatch Agent
          |
          v
   CloudWatch Logs
```

---

# 20. Next Steps

After completing EC2 creation, continue in this order:

1. [Elastic IP](./elastic-ip.md)
2. [Security Group](./security-group.md)
3. [Docker Deployment](../backend/docker-deployment.md)
4. [Nginx Setup](../backend/nginx.md)
5. [SSL / HTTPS Setup](../backend/ssl-https.md)
6. [GitHub Actions + GHCR](../cicd/github-actions.md)
7. [CloudWatch Monitoring](./cloudwatch.md)

---

# Security Checklist

Before moving to the next step:

- [ ] Ubuntu 24.04 LTS selected
- [ ] Appropriate EC2 instance type selected
- [ ] SSH key pair created
- [ ] `.pem` file downloaded
- [ ] `.pem` permissions set to `400`
- [ ] SSH restricted to your IP
- [ ] HTTP 80 publicly accessible
- [ ] HTTPS 443 publicly accessible
- [ ] Node.js port 3000 is NOT publicly exposed
- [ ] Root storage size is appropriate
- [ ] EC2 instance launched successfully
- [ ] SSH connection tested
- [ ] Ubuntu packages updated
- [ ] Elastic IP configured for production