# AWS Security Group

This guide explains how to configure the EC2 Security Group for a production backend.

A Security Group acts as a virtual firewall for the EC2 instance.

The goal is to expose only the ports that are actually required.

---

# 1. Recommended Production Rules

For a typical production Node.js backend using Nginx and HTTPS:

| Type | Protocol | Port | Source | Purpose |
|---|---|---:|---|---|
| SSH | TCP | 22 | Your IP only | Server administration |
| HTTP | TCP | 80 | `0.0.0.0/0` | HTTP and SSL certificate setup |
| HTTPS | TCP | 443 | `0.0.0.0/0` | Public HTTPS traffic |

Do **not** expose the Node.js application port publicly.

For example, do not add:

```text
TCP 3000 → 0.0.0.0/0
```

---

# 2. Open the Security Group

Go to:

```text
AWS Console
→ EC2
→ Security Groups
```

Select the Security Group attached to the production EC2 instance.

You can also find it from:

```text
EC2
→ Instances
→ Select your instance
→ Security
→ Security groups
```

---

# 3. Configure Inbound Rules

Open:

```text
Inbound rules
→ Edit inbound rules
```

Add the following.

---

## SSH

```text
Type:
SSH

Protocol:
TCP

Port:
22

Source:
My IP
```

This normally creates a rule similar to:

```text
22 / TCP / <YOUR_PUBLIC_IP>/32
```

This means only your current public IP can connect through SSH.

Example:

```text
22 → 203.0.113.50/32
```

Do not normally use:

```text
0.0.0.0/0
```

for production SSH access.

---

# 4. HTTP

Add:

```text
Type:
HTTP

Protocol:
TCP

Port:
80

Source:
Anywhere IPv4
```

Equivalent:

```text
80 / TCP / 0.0.0.0/0
```

HTTP is needed because:

- Nginx listens on port 80
- HTTP requests may be redirected to HTTPS
- Certbot/Let's Encrypt may use HTTP validation

---

# 5. HTTPS

Add:

```text
Type:
HTTPS

Protocol:
TCP

Port:
443

Source:
Anywhere IPv4
```

Equivalent:

```text
443 / TCP / 0.0.0.0/0
```

This allows users to access the production API through HTTPS.

Example:

```text
https://api.<DOMAIN>
```

---

# 6. Do NOT Add Port 3000

The Node.js application may run internally on:

```text
3000
```

But this port should not be publicly accessible.

The recommended architecture is:

```text
Internet
    |
    | HTTPS :443
    v
Elastic IP
    |
    v
EC2
    |
    v
Nginx
    |
    | 127.0.0.1:3000
    v
Docker Container
    |
    v
Node.js API
```

Docker Compose should bind the application port to localhost:

```yaml
ports:
  - "127.0.0.1:${APP_PORT:-3000}:3000"
```

This means:

```text
127.0.0.1:3000
```

is accessible from the EC2 server itself, but not directly from the public internet.

---

# 7. Outbound Rules

For a normal backend server, keep the default outbound rule unless your application has a specific networking requirement.

Typical default:

```text
All traffic
Destination:
0.0.0.0/0
```

The application may need outbound access to services such as:

- MongoDB
- AWS services
- Razorpay
- Email providers
- MSG91
- GitHub Container Registry
- Package/image registries
- External APIs

If you later need tighter outbound restrictions, configure them based on the application's actual requirements.

Do not unnecessarily restrict outbound traffic during the initial setup.

---

# 8. IPv6

If your server is not using IPv6, you do not need to add IPv6 inbound rules.

For example, these are not required unless IPv6 is intentionally configured:

```text
::/0
```

Keep the configuration simple and use IPv4 where appropriate.

---

# 9. Verify the Rules

Your final inbound rules should look similar to:

```text
Inbound Rules

SSH
22
My IP

HTTP
80
0.0.0.0/0

HTTPS
443
0.0.0.0/0
```

No public:

```text
3000
```

---

# 10. Test SSH

From your local machine:

```bash
ssh -i ~/Downloads/<PROJECT_NAME>-prod-key.pem ubuntu@<ELASTIC_IP>
```

If the connection works, SSH is configured correctly.

---

# 11. Test HTTP/HTTPS

After Nginx and the domain are configured:

```bash
curl -I http://api.<DOMAIN>
```

Expected behavior in a HTTPS production setup:

```text
HTTP/1.1 301 Moved Permanently
Location: https://api.<DOMAIN>/...
```

Then test HTTPS:

```bash
curl https://api.<DOMAIN>/health
```

Expected application response:

```text
Hello, I am alive
```

---

# 12. Test That Port 3000 Is Not Public

From a machine outside the EC2 server, do not expect:

```text
http://<ELASTIC_IP>:3000
```

to be publicly accessible.

This is intentional.

The public entry point should be:

```text
https://api.<DOMAIN>
```

not:

```text
http://<ELASTIC_IP>:3000
```

---

# 13. Security Best Practices

### SSH

Prefer:

```text
22 → Your IP only
```

If your public IP changes, update the Security Group rule.

If multiple administrators need SSH access, add their specific IPs instead of opening SSH to the entire internet.

---

### Web Traffic

Public:

```text
80
443
```

is normal for a web/API server.

---

### Application Ports

Keep internal application ports private.

Examples:

```text
3000
8080
5000
```

should not normally be exposed directly when Nginx is being used as the reverse proxy.

---

# 14. Production Architecture

The final network flow should be:

```text
Client
  |
  | HTTPS :443
  v
DNS
  |
  v
Elastic IP
  |
  v
EC2 Security Group
  |
  +---- SSH :22
  |       |
  |       +---- Your IP only
  |
  +---- HTTP :80
  |
  +---- HTTPS :443
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

---

# 15. Common Mistakes

Avoid these configurations:

### Do not expose SSH publicly

```text
22 → 0.0.0.0/0
```

unless there is a specific operational requirement.

### Do not expose Node.js directly

```text
3000 → 0.0.0.0/0
```

### Do not use the EC2 public IP as the permanent application URL

Prefer:

```text
api.<DOMAIN>
```

with DNS pointing to the Elastic IP.

### Do not open every port

Only expose ports required by the application.

---

# 16. Security Group Checklist

Before moving to the next deployment step:

- [ ] SSH 22 allows only required administrator IPs
- [ ] HTTP 80 is open
- [ ] HTTPS 443 is open
- [ ] Port 3000 is not publicly exposed
- [ ] Default outbound access is reviewed
- [ ] Security Group is attached to the correct EC2 instance
- [ ] SSH connection works
- [ ] Public web traffic reaches Nginx

---

# Next Step

After the Security Group is configured, continue with:

[Docker Deployment](../backend/docker-deployment.md)