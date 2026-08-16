# SSL / HTTPS Setup

This guide explains how to enable HTTPS for a production backend running on AWS EC2 with Nginx.

The setup uses:

- Let's Encrypt
- Certbot
- Nginx
- A DNS subdomain
- AWS EC2

The goal is to make the API available through:

```text
https://api.<DOMAIN>
```

---

# 1. Architecture

Before SSL:

```text
Client
   |
   | HTTP :80
   v
DNS
   |
   v
EC2
   |
   v
Nginx
   |
   v
127.0.0.1:3000
   |
   v
Docker API
```

After SSL:

```text
Client
   |
   | HTTPS :443
   v
DNS
   |
   v
EC2
   |
   v
Nginx
   |
   | TLS termination
   |
   v
127.0.0.1:3000
   |
   v
Docker API
```

Nginx handles HTTPS while the Node.js application continues running internally on port `3000`.

---

# 2. Prerequisites

Before running Certbot, make sure:

- EC2 is running
- Elastic IP is configured
- Security Group allows TCP 80
- Security Group allows TCP 443
- Nginx is installed and running
- API DNS record points to the EC2 Elastic IP
- HTTP API is already working

Test:

```bash
curl http://api.<DOMAIN>/health
```

Expected:

```text
Hello, I am alive
```

---

# 3. Verify DNS

Check:

```bash
dig api.<DOMAIN>
```

or:

```bash
nslookup api.<DOMAIN>
```

The API hostname should resolve to the EC2 Elastic IP.

Example:

```text
api.example.com → <ELASTIC_IP>
```

If DNS does not resolve correctly, fix DNS before continuing.

---

# 4. Install Certbot

Update packages:

```bash
sudo apt update
```

Install Certbot and the Nginx plugin:

```bash
sudo apt install certbot python3-certbot-nginx -y
```

Verify:

```bash
certbot --version
```

---

# 5. Run Certbot

Run:

```bash
sudo certbot --nginx -d api.<DOMAIN>
```

Certbot will:

1. Verify domain ownership
2. Obtain a Let's Encrypt certificate
3. Configure Nginx
4. Enable HTTPS
5. Normally configure HTTP → HTTPS redirection

Follow the prompts.

If Certbot asks whether to redirect HTTP traffic to HTTPS, choose the redirect option.

---

# 6. Successful Certificate Deployment

A successful deployment will report that the certificate was obtained and deployed for the domain.

The certificate files are normally stored under:

```text
/etc/letsencrypt/live/api.<DOMAIN>/
```

Important files include:

```text
fullchain.pem
privkey.pem
```

Do not expose or commit the private key.

---

# 7. Test HTTPS

Run:

```bash
curl https://api.<DOMAIN>/health
```

Expected:

```text
Hello, I am alive
```

You can also open:

```text
https://api.<DOMAIN>/health
```

in a browser.

The browser should show a valid HTTPS connection.

---

# 8. Test HTTP Redirect

Run:

```bash
curl -I http://api.<DOMAIN>/health
```

Expected behavior:

```text
HTTP/1.1 301 Moved Permanently
Location: https://api.<DOMAIN>/health
```

The exact status may vary depending on the Nginx configuration.

The important point is that HTTP should redirect to HTTPS.

---

# 9. Check Nginx Configuration

After Certbot changes the Nginx configuration:

```bash
sudo nginx -t
```

Expected:

```text
syntax is ok
test is successful
```

Check the generated configuration:

```bash
sudo nginx -T
```

---

# 10. Check Nginx Status

```bash
sudo systemctl status nginx
```

If necessary:

```bash
sudo systemctl reload nginx
```

Avoid restarting Nginx unnecessarily for normal configuration changes.

---

# 11. Check Certificate Information

Run:

```bash
sudo certbot certificates
```

This shows:

- Certificate name
- Domain
- Expiration date
- Certificate path
- Private key path

Example:

```text
Certificate Name: api.<DOMAIN>
Domains: api.<DOMAIN>
```

---

# 12. Test Automatic Renewal

Let's Encrypt certificates are short-lived, so automatic renewal is important.

Do not manually renew the certificate every time.

Test the renewal process with:

```bash
sudo certbot renew --dry-run
```

A successful result should indicate that simulated renewal succeeded.

Example:

```text
Congratulations, all simulated renewals succeeded
```

This does not actually replace the production certificate.

It only tests the renewal process.

---

# 13. Check Certbot Renewal Timer

On Ubuntu, Certbot commonly uses a systemd timer.

Check:

```bash
systemctl list-timers | grep certbot
```

You can also check:

```bash
sudo systemctl status certbot.timer
```

If the timer exists and is active, automatic renewal is configured.

---

# 14. Do Not Put SSL Certificates in Git

Never commit:

```text
privkey.pem
fullchain.pem
cert.pem
chain.pem
```

Do not copy them into:

```text
src/
```

or:

```text
docs/
```

or:

```text
.github/
```

They should remain on the server under:

```text
/etc/letsencrypt/
```

---

# 15. Certificate Security

The private key is especially sensitive:

```text
/etc/letsencrypt/live/api.<DOMAIN>/privkey.pem
```

Do not:

- Commit it to GitHub
- Put it in `.env`
- Send it through chat
- Store it in source code
- Upload it to public storage

Certbot and Nginx manage the certificate files directly on the server.

---

# 16. HTTP and HTTPS Ports

The EC2 Security Group should allow:

```text
HTTP
80
0.0.0.0/0
```

and:

```text
HTTPS
443
0.0.0.0/0
```

Port:

```text
3000
```

should remain private.

---

# 17. Final Network Flow

The final production request path is:

```text
Browser / API Client
        |
        | HTTPS :443
        v
api.<DOMAIN>
        |
        v
Elastic IP
        |
        v
EC2 Security Group
        |
        v
Nginx
        |
        | reverse proxy
        v
127.0.0.1:3000
        |
        v
Docker Container
        |
        v
Node.js API
```

---

# 18. Verify the Complete Setup

Run these commands.

### HTTPS

```bash
curl https://api.<DOMAIN>/health
```

### HTTP redirect

```bash
curl -I http://api.<DOMAIN>/health
```

### Certificate

```bash
sudo certbot certificates
```

### Renewal test

```bash
sudo certbot renew --dry-run
```

### Nginx configuration

```bash
sudo nginx -t
```

### Docker

```bash
cd ~/app
docker compose ps
```

### Application logs

```bash
docker compose logs --tail=100 api
```

---

# 19. Troubleshooting

## Certificate request fails

Check DNS:

```bash
dig api.<DOMAIN>
```

Check Nginx:

```bash
sudo nginx -t
```

Check HTTP access:

```bash
curl http://api.<DOMAIN>/health
```

Make sure port 80 is allowed in the Security Group.

---

## HTTPS returns 502

Check the application:

```bash
cd ~/app
docker compose ps
```

Then:

```bash
curl http://127.0.0.1:3000/health
```

If the local health check fails, fix the Docker application first.

Then check Nginx errors:

```bash
sudo tail -f /var/log/nginx/error.log
```

---

## Browser says connection is not secure

Check:

```bash
sudo certbot certificates
```

Then:

```bash
curl -I https://api.<DOMAIN>/health
```

Also verify that the URL is using:

```text
https://
```

instead of:

```text
http://
```

---

# 20. SSL Checklist

- [ ] DNS points to Elastic IP
- [ ] Port 80 is open
- [ ] Port 443 is open
- [ ] Nginx is running
- [ ] HTTP API works
- [ ] Certbot installed
- [ ] Certificate obtained
- [ ] Certificate deployed to Nginx
- [ ] HTTPS API works
- [ ] HTTP redirects to HTTPS
- [ ] `sudo nginx -t` succeeds
- [ ] `sudo certbot renew --dry-run` succeeds
- [ ] Certbot renewal timer is active
- [ ] SSL private key is not committed to Git

---

# 21. Production Result

After completing this setup, the backend should be accessible through:

```text
https://api.<DOMAIN>/health
```

The application itself remains private:

```text
127.0.0.1:3000
```

Public users access:

```text
HTTPS :443
```

Nginx handles TLS and proxies the request to the Dockerized Node.js application.

---

# Next Step

After HTTPS is configured, continue with:

[GitHub Actions CI/CD](../cicd/github-actions.md)