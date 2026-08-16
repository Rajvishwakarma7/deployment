# AWS Elastic IP

This guide explains how to allocate and associate an Elastic IP address with an AWS EC2 instance.

An Elastic IP provides a stable public IPv4 address for a production EC2 server.

---

## Why Use an Elastic IP?

A normal EC2 public IPv4 address can change when the instance is stopped and started.

A production backend should use a stable IP because DNS records point to that address.

Recommended architecture:

```text
Domain
   |
   v
DNS
   |
   v
Elastic IP
   |
   v
EC2 Instance
```

Example:

```text
api.<DOMAIN>
      |
      v
<ELASTIC_IP>
      |
      v
EC2
```

---

# 1. Open Elastic IPs

Go to:

```text
AWS Console
→ EC2
→ Network & Security
→ Elastic IPs
```

---

# 2. Allocate Elastic IP

Click:

```text
Allocate Elastic IP address
```

For the resource type, use:

```text
Public IPv4 address pool:
Amazon's IPv4 address pool
```

Then click:

```text
Allocate
```

AWS will allocate a new Elastic IP.

Example:

```text
<ELASTIC_IP>
```

---

# 3. Associate the Elastic IP

Select the newly allocated Elastic IP.

Then choose:

```text
Actions
→ Associate Elastic IP address
```

For:

```text
Resource type:
Instance
```

Select your production EC2 instance.

Example:

```text
<PROJECT_NAME>-production
```

Then click:

```text
Associate
```

---

# 4. Verify the Association

Go to:

```text
EC2
→ Instances
→ Select your instance
```

Check:

```text
Public IPv4 address
```

It should now show the Elastic IP.

Example:

```text
<ELASTIC_IP>
```

You can also check:

```text
Elastic IP address
```

in the instance details.

---

# 5. Test SSH

From your local machine:

```bash
ssh -i ~/Downloads/<PROJECT_NAME>-prod-key.pem ubuntu@<ELASTIC_IP>
```

Example:

```bash
ssh -i ~/Downloads/my-project-prod-key.pem ubuntu@203.0.113.10
```

If SSH works, the Elastic IP is correctly associated.

---

# 6. Update DNS

When configuring the backend API domain, create an A record at your DNS provider.

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

This creates:

```text
api.<DOMAIN>
```

pointing to the EC2 server.

For example:

```text
api.example.com
      |
      v
<ELASTIC_IP>
```

---

# 7. Verify DNS

After saving the DNS record, check from your local machine:

```bash
dig api.<DOMAIN>
```

Or:

```bash
nslookup api.<DOMAIN>
```

You should see the Elastic IP in the response.

You can also use:

```bash
ping api.<DOMAIN>
```

Note: Ping may fail if ICMP is not allowed by the Security Group. DNS resolution can still be correct.

---

# 8. Important: Do Not Change the EC2 Application Port

The Elastic IP does not mean the Node.js port should be publicly exposed.

Keep the production architecture:

```text
Internet
   |
   | 80 / 443
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
Docker
   |
   v
Node.js API
```

Do not add port 3000 to the Security Group.

---

# 9. Elastic IP and Security Group

The Elastic IP does not replace the Security Group.

Both are required:

```text
Elastic IP
    ↓
Provides stable public IP

Security Group
    ↓
Controls allowed traffic
```

Recommended production inbound rules:

| Type | Port | Source |
|---|---:|---|
| SSH | 22 | Your IP only |
| HTTP | 80 | `0.0.0.0/0` |
| HTTPS | 443 | `0.0.0.0/0` |

---

# 10. Important AWS Cost Consideration

AWS may charge for public IPv4 addresses, including Elastic IP usage, depending on the current AWS pricing rules.

Check the current AWS public IPv4 pricing before creating unused addresses.

Avoid allocating Elastic IPs that are not associated with a running resource.

When an Elastic IP is no longer required, release it.

---

# 11. Releasing an Elastic IP

Only release the Elastic IP when it is no longer needed.

Go to:

```text
EC2
→ Network & Security
→ Elastic IPs
```

Select the address.

If it is associated with an instance:

```text
Actions
→ Disassociate Elastic IP address
```

Then:

```text
Actions
→ Release Elastic IP addresses
```

Do not release a production Elastic IP unless you are intentionally changing the production IP and DNS configuration.

---

# 12. Production Checklist

Before moving to the next step:

- [ ] Elastic IP allocated
- [ ] Elastic IP associated with the production EC2 instance
- [ ] EC2 shows the Elastic IP as its public IPv4 address
- [ ] SSH works using the Elastic IP
- [ ] DNS A record can point to the Elastic IP
- [ ] Node.js port 3000 is not publicly exposed
- [ ] Security Group allows only required ports
- [ ] Unused Elastic IPs are not left allocated

---

# Next Step

After the Elastic IP is configured, continue with:

[Security Group](./security-group.md)