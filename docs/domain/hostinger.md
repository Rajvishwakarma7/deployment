# Hostinger DNS & Domain Setup

This guide documents how to manage the project's domain and DNS records from Hostinger while hosting the frontend on Vercel and the backend on AWS EC2.

For this project, Hostinger remains the DNS provider.

---

# 1. Architecture

```text
                    Hostinger DNS
                         |
             +-----------+-----------+
             |                       |
             v                       v
        Frontend                  Backend API
        Vercel                    AWS EC2
             |                       |
             v                       v
   williamhazlitt.in        api.williamhazlitt.in
   www.williamhazlitt.in            |
                                     v
                                   Nginx
                                     |
                                     v
                              Docker :3000
```

Hostinger manages the DNS records.

Vercel hosts the frontend.

AWS EC2 hosts the backend.

---

# 2. Important Domain Naming

Do not write:

```text
https.williamhazlitt.com
```

`https://` is a protocol, not part of the hostname.

Correct examples:

```text
https://williamhazlitt.in
https://www.williamhazlitt.in
https://api.williamhazlitt.in
```

The actual hostname is:

```text
williamhazlitt.in
www.williamhazlitt.in
api.williamhazlitt.in
```

---

# 3. DNS Management

Log in to Hostinger.

Go to:

```text
Domains
    |
    v
Your Domain
    |
    v
DNS / Nameservers
    |
    v
Manage DNS Records
```

All DNS records for the application are managed here.

---

# 4. Root Domain → Vercel

The root domain:

```text
williamhazlitt.in
```

is used for the frontend.

In Vercel, add:

```text
williamhazlitt.in
```

Vercel will show the DNS configuration required for the domain.

Add the exact records provided by Vercel in Hostinger DNS.

Do not guess the values.

---

# 5. WWW → Vercel

Add:

```text
www.williamhazlitt.in
```

to the Vercel project if the `www` version should also work.

Configure the DNS record in Hostinger according to the exact value shown by Vercel.

A common setup is a CNAME for:

```text
www
```

pointing to the Vercel target provided by Vercel.

Always use the current value shown in the Vercel Domains page.

---

# 6. API Subdomain → AWS EC2

The backend uses:

```text
api.williamhazlitt.in
```

Create a DNS record in Hostinger for:

```text
api
```

The record points to the EC2 public/Elastic IP.

Example:

```text
Type: A
Name: api
Value: <EC2_ELASTIC_IP>
TTL: default
```

This results in:

```text
api.williamhazlitt.in
        |
        v
EC2 Elastic IP
        |
        v
Nginx
        |
        v
Docker container :3000
```

Use an Elastic IP for production rather than relying on an EC2 public IP that may change after an instance lifecycle event.

---

# 7. Current Project Example

For the William Hazlitt project:

```text
Frontend:
https://williamhazlitt.in

Frontend:
https://www.williamhazlitt.in

Backend:
https://api.williamhazlitt.in

Backend health:
https://api.williamhazlitt.in/health
```

The frontend is hosted on Vercel.

The backend is hosted on AWS EC2.

Hostinger manages the DNS.

---

# 8. DNS Record Overview

The final DNS configuration conceptually looks like:

```text
Host                  Type       Destination
---------------------------------------------------------
@                     Vercel     Vercel-provided value
www                   CNAME      Vercel-provided value
api                   A          EC2 Elastic IP
```

The exact Vercel destination values can change.

Always copy the current values shown in:

```text
Vercel
→ Project
→ Settings
→ Domains
```

---

# 9. DNS Propagation

After changing DNS records, the change may not appear immediately everywhere.

Check the domain:

```bash
nslookup williamhazlitt.in
```

Check the API:

```bash
nslookup api.williamhazlitt.in
```

You can also use:

```bash
dig williamhazlitt.in
```

and:

```bash
dig api.williamhazlitt.in
```

---

# 10. Verify Frontend

Open:

```text
https://williamhazlitt.in
```

and:

```text
https://www.williamhazlitt.in
```

Both should resolve to the Vercel frontend if both are configured.

---

# 11. Verify Backend

Open:

```text
https://api.williamhazlitt.in/health
```

Expected:

```text
Hello, I am alive 🦊
```

If this works:

```text
Browser
   |
   v
Hostinger DNS
   |
   v
EC2
   |
   v
Nginx
   |
   v
Docker
   |
   v
Node.js
```

the DNS and backend routing are working.

---

# 12. DNS Does Not Handle HTTPS

Hostinger DNS only resolves the hostname to the destination.

For example:

```text
api.williamhazlitt.in
        |
        v
EC2 IP
```

HTTPS is handled separately.

For the backend:

```text
Client
   |
   | HTTPS :443
   v
Nginx
   |
   | HTTP localhost
   v
Docker :3000
```

The SSL certificate is installed on the EC2/Nginx server using Certbot.

For the Vercel frontend, Vercel manages HTTPS for the configured domain.

---

# 13. DNS + Nginx Relationship

DNS:

```text
api.williamhazlitt.in
        |
        v
EC2 IP
```

Nginx:

```text
api.williamhazlitt.in
        |
        v
127.0.0.1:3000
```

Docker:

```text
127.0.0.1:3000
        |
        v
Node.js API
```

DNS does not directly connect to Docker.

Each layer has a separate responsibility.

---

# 14. Do Not Point the API to Vercel

The backend API should not use the Vercel frontend DNS record.

Correct:

```text
williamhazlitt.in
        |
        v
Vercel
```

and:

```text
api.williamhazlitt.in
        |
        v
EC2
```

---

# 15. Do Not Point the Frontend to EC2

The frontend domain should resolve to Vercel.

The API subdomain should resolve to EC2.

Correct:

```text
Frontend:
williamhazlitt.in → Vercel

API:
api.williamhazlitt.in → EC2
```

---

# 16. Recommended Production Setup

Use:

```text
williamhazlitt.in
```

as the primary frontend domain.

Optionally support:

```text
www.williamhazlitt.in
```

Use:

```text
api.williamhazlitt.in
```

for the backend API.

This provides a clean separation:

```text
Frontend:
williamhazlitt.in

API:
api.williamhazlitt.in
```

---

# 17. Changing the EC2 IP

If using an Elastic IP:

```text
api.williamhazlitt.in
        |
        v
Elastic IP
        |
        v
EC2
```

The DNS record does not normally need to change when the EC2 instance is restarted.

If the Elastic IP itself is changed, update the Hostinger `api` A record.

---

# 18. Security Notes

- Keep DNS management centralized in Hostinger.
- Use an Elastic IP for the production EC2 server.
- Do not expose Docker port 3000 publicly.
- Allow public HTTP/HTTPS through Nginx.
- Keep SSH restricted to trusted IPs.
- Use HTTPS for frontend and API.
- Do not expose private services through DNS.
- Do not publish database ports publicly.

---

# 19. Troubleshooting

## Domain does not resolve

Check:

```bash
nslookup williamhazlitt.in
```

or:

```bash
dig williamhazlitt.in
```

Verify the DNS record in Hostinger.

---

## API domain does not resolve

Check:

```bash
nslookup api.williamhazlitt.in
```

Verify:

```text
api → EC2 Elastic IP
```

---

## API resolves but HTTPS fails

Check Nginx:

```bash
sudo nginx -t
```

Check Nginx:

```bash
sudo systemctl status nginx
```

Check the certificate:

```bash
sudo certbot certificates
```

---

## API returns 502

Check the Docker container:

```bash
cd /home/ubuntu/app
docker compose ps
```

Check logs:

```bash
docker compose logs --tail=100 api
```

Test the application directly:

```bash
curl http://127.0.0.1:3000/health
```

If this works but the domain returns 502, check the Nginx configuration.

---

# 20. Production Checklist

- [ ] Domain registered with Hostinger
- [ ] Hostinger is managing DNS
- [ ] Root domain configured for Vercel
- [ ] `www` configured for Vercel if required
- [ ] `api` A record points to EC2 Elastic IP
- [ ] DNS propagation verified
- [ ] Frontend opens through HTTPS
- [ ] API opens through HTTPS
- [ ] Nginx configured
- [ ] Certbot certificate installed
- [ ] Docker application running
- [ ] Docker port 3000 not publicly exposed
- [ ] EC2 Security Group configured
- [ ] Elastic IP attached to EC2

---

# 21. Related Documentation

Vercel:

```text
../vercel/deployment.md
```

EC2:

```text
./ec2-setup.md
```

Security Group:

```text
./security-group.md
```

Nginx:

```text
../backend/nginx.md
```

SSL / HTTPS:

```text
../backend/ssl-https.md
```

Docker:

```text
../backend/docker-deployment.md
```

CloudWatch:

```text
./cloudwatch.md
```