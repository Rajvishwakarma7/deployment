# Nginx Setup

This guide explains how to configure Nginx as a reverse proxy for a Dockerized Node.js backend running on AWS EC2.

The configuration keeps the Node.js application private and exposes the API through a domain/subdomain.

---

# 1. Architecture

The production request flow is:

```text
Client
   |
   | https://api.<DOMAIN>
   v
DNS
   |
   v
Elastic IP
   |
   v
EC2
   |
   v
Nginx :80 / :443
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

Nginx is the public-facing reverse proxy.

The Node.js application does not need to be directly exposed to the internet.

---

# 2. Prerequisites

Before configuring Nginx, make sure:

- EC2 is running
- Elastic IP is configured
- Security Group allows ports 80 and 443
- Docker container is running
- Application works locally on EC2
- DNS A record points the API subdomain to the Elastic IP

Test the application:

```bash
curl http://127.0.0.1:3000/health
```

Expected:

```text
Hello, I am alive
```

---

# 3. Install Nginx

On Ubuntu:

```bash
sudo apt update
```

Install:

```bash
sudo apt install nginx -y
```

Check the version:

```bash
nginx -v
```

Check the service:

```bash
sudo systemctl status nginx
```

If necessary:

```bash
sudo systemctl start nginx
```

Enable Nginx on boot:

```bash
sudo systemctl enable nginx
```

---

# 4. DNS Configuration

Create an A record with your DNS provider.

Example:

```text
Type:
A

Name:
api

Value:
<ELASTIC_IP>

TTL:
Default / Automatic
```

This should create:

```text
api.<DOMAIN>
```

pointing to the EC2 Elastic IP.

Verify:

```bash
dig api.<DOMAIN>
```

or:

```bash
nslookup api.<DOMAIN>
```

The response should contain the Elastic IP.

---

# 5. Create Nginx Site Configuration

Nginx stores site configurations in:

```text
/etc/nginx/sites-available/
```

Create a configuration file:

```bash
sudo nano /etc/nginx/sites-available/api.<DOMAIN>
```

Example:

```nginx
server {
    listen 80;
    listen [::]:80;

    server_name api.<DOMAIN>;

    location / {
        proxy_pass http://127.0.0.1:3000;

        proxy_http_version 1.1;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Replace:

```text
api.<DOMAIN>
```

with the actual API hostname.

Example:

```text
api.example.com
```

---

# 6. Enable the Site

Create a symbolic link:

```bash
sudo ln -s /etc/nginx/sites-available/api.<DOMAIN> \
  /etc/nginx/sites-enabled/api.<DOMAIN>
```

---

# 7. Remove the Default Site

If the default Nginx site is not required:

```bash
sudo rm -f /etc/nginx/sites-enabled/default
```

This prevents the default Nginx page from handling requests that should go to your API configuration.

---

# 8. Test Nginx Configuration

Always test the configuration before reloading:

```bash
sudo nginx -t
```

Expected:

```text
syntax is ok
test is successful
```

Never skip this step after changing Nginx configuration.

---

# 9. Reload Nginx

If the configuration test succeeds:

```bash
sudo systemctl reload nginx
```

---

# 10. Test Nginx Locally

From the EC2 server:

```bash
curl -H "Host: api.<DOMAIN>" http://127.0.0.1/health
```

Expected:

```text
Hello, I am alive
```

This confirms:

```text
Nginx
  |
  v
127.0.0.1:3000
  |
  v
Docker API
```

---

# 11. Test Through the Public Domain

From your local computer:

```bash
curl http://api.<DOMAIN>/health
```

Expected:

```text
Hello, I am alive
```

If DNS and Nginx are correctly configured, the request should reach the EC2 server.

---

# 12. HTTP to HTTPS

After SSL is configured, HTTP should redirect to HTTPS.

Expected:

```text
http://api.<DOMAIN>/health
             |
             v
HTTP 301/308
             |
             v
https://api.<DOMAIN>/health
```

Test:

```bash
curl -I http://api.<DOMAIN>/health
```

Expected behavior:

```text
HTTP/1.1 301 Moved Permanently
Location: https://api.<DOMAIN>/health
```

The exact redirect status can vary depending on the Nginx/Certbot configuration.

---

# 13. Nginx Reverse Proxy Headers

These headers preserve useful client/request information:

```nginx
proxy_set_header Host $host;
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $scheme;
```

They allow the backend to understand information about:

- Original host
- Client IP
- Proxy chain
- Original HTTP/HTTPS protocol

---

# 14. Request Flow

A production request looks like:

```text
GET https://api.<DOMAIN>/users
          |
          v
       DNS
          |
          v
     Elastic IP
          |
          v
        Nginx
          |
          | proxy_pass
          v
127.0.0.1:3000
          |
          v
    Docker Container
          |
          v
     Node.js API
```

The client does not need to know that the Node.js application is running on port 3000.

---

# 15. Why Nginx?

Nginx provides a public reverse-proxy layer in front of the application.

It can handle:

- HTTP
- HTTPS
- SSL/TLS termination
- HTTP → HTTPS redirects
- Reverse proxying
- Request headers
- Rate limiting
- Access logging
- Error logging
- Multiple backend services
- Load balancing

For a single EC2 server, Nginx provides a clean separation between public web traffic and the internal application.

---

# 16. Nginx Logs

Nginx normally stores logs under:

```text
/var/log/nginx/
```

Access log:

```bash
sudo tail -f /var/log/nginx/access.log
```

Error log:

```bash
sudo tail -f /var/log/nginx/error.log
```

For the API site, logs may be configured as:

```text
/var/log/nginx/api.<DOMAIN>.access.log
/var/log/nginx/api.<DOMAIN>.error.log
```

depending on the site configuration.

---

# 17. Check Nginx Service

Check:

```bash
sudo systemctl status nginx
```

Restart if required:

```bash
sudo systemctl restart nginx
```

Prefer reload for normal configuration changes:

```bash
sudo systemctl reload nginx
```

Reload avoids unnecessarily stopping the running Nginx service.

---

# 18. Check Listening Ports

Use:

```bash
sudo ss -lntp
```

You should normally see Nginx listening on:

```text
:80
```

and after HTTPS setup:

```text
:443
```

Docker/Node.js should remain bound to:

```text
127.0.0.1:3000
```

---

# 19. Common Problems

## Nginx returns 404

Check:

```bash
sudo nginx -T
```

Verify that the correct `server_name` exists.

Also verify the enabled configuration:

```bash
ls -la /etc/nginx/sites-enabled/
```

---

## Nginx returns 502 Bad Gateway

Usually this means Nginx cannot reach the backend.

Check Docker:

```bash
cd ~/app
docker compose ps
```

Check:

```bash
curl http://127.0.0.1:3000/health
```

If this fails, fix the Docker application first.

Then check:

```bash
sudo tail -f /var/log/nginx/error.log
```

---

## Domain does not reach EC2

Check DNS:

```bash
dig api.<DOMAIN>
```

Confirm the returned IP is the EC2 Elastic IP.

Also verify Security Group:

```text
TCP 80 → 0.0.0.0/0
TCP 443 → 0.0.0.0/0
```

---

# 20. Important Security Rule

Do not expose:

```text
3000 → 0.0.0.0/0
```

The preferred setup is:

```text
Public
   |
   | 80 / 443
   v
 Nginx
   |
   | localhost
   v
Docker :3000
```

---

# 21. Nginx Checklist

- [ ] Nginx installed
- [ ] Nginx enabled on boot
- [ ] API DNS A record points to Elastic IP
- [ ] Site created in `sites-available`
- [ ] Site enabled in `sites-enabled`
- [ ] Default site removed if unnecessary
- [ ] `sudo nginx -t` succeeds
- [ ] Nginx reloaded
- [ ] Local proxy test succeeds
- [ ] Public HTTP API test succeeds
- [ ] Port 3000 remains private
- [ ] Nginx logs checked when troubleshooting

---

# Next Step

After Nginx is working over HTTP, continue with:

[SSL / HTTPS Setup](./ssl-https.md)