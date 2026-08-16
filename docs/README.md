# Deployment Documentation

This repository contains reusable deployment, infrastructure, CI/CD, monitoring,
and third-party service documentation for our applications.

The documentation is written using generic placeholders so it can be reused
for different projects.

---

## Documentation

### Frontend

- [Vercel Deployment](./vercel/deployment.md)

---

### AWS

- [EC2 Setup](./aws/ec2-setup.md)
- [Elastic IP](./aws/elastic-ip.md)
- [Security Group](./aws/security-group.md)
- [CloudWatch Monitoring](./aws/cloudwatch.md)
- [S3](./aws/s3.md)

---

### Backend

- [Docker Deployment](./backend/docker-deployment.md)
- [Nginx Setup](./backend/nginx.md)
- [SSL / HTTPS Setup](./backend/ssl-https.md)

---

### CI/CD

- [GitHub Actions + GHCR](./cicd/github-actions.md)

---

### Services

- [MSG91 OTP](./services/msg91.md)
- [Razorpay](./services/razorpay.md)

---

## Recommended Deployment Order

For a new backend project, follow the documentation in this order:

1. [EC2 Setup](./aws/ec2-setup.md)
2. [Elastic IP](./aws/elastic-ip.md)
3. [Security Group](./aws/security-group.md)
4. [Docker Deployment](./backend/docker-deployment.md)
5. [Nginx Setup](./backend/nginx.md)
6. [SSL / HTTPS Setup](./backend/ssl-https.md)
7. [GitHub Actions + GHCR](./cicd/github-actions.md)
8. [CloudWatch Monitoring](./aws/cloudwatch.md)

---

## Important

Never commit sensitive information such as:

- `.env`
- AWS access keys
- GitHub personal access tokens
- SSH private keys
- API secrets
- Database passwords
- Payment gateway secrets

Use environment variables, GitHub Actions Secrets, AWS IAM, or the appropriate
secret-management mechanism instead.