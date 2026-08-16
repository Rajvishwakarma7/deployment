# Docker Deployment

This guide explains how to containerize and deploy a Node.js backend application on AWS EC2 using Docker and Docker Compose.

The setup is designed for production and can also be reused for other projects.

---

# 1. Deployment Architecture

The production architecture is:

```text
Developer
    |
    v
GitHub
    |
    v
GitHub Actions
    |
    v
GHCR
    |
    | Docker image
    v
AWS EC2
    |
    v
Docker Compose
    |
    v
Node.js API
    |
    v
Nginx
    |
    v
HTTPS
```

The application itself runs inside Docker.

Nginx handles public HTTP/HTTPS traffic.

The Node.js application is kept private on:

```text
127.0.0.1:3000
```

---

# 2. Prerequisites

The EC2 server should already have:

- Ubuntu
- Docker
- Docker Compose
- Git
- Nginx
- Security Group configured
- Elastic IP configured

Verify Docker:

```bash
docker --version
```

Verify Docker Compose:

```bash
docker compose version
```

Test Docker:

```bash
docker run --rm hello-world
```

Expected:

```text
Hello from Docker!
```

---

# 3. Create Application Directory

Use a simple application directory.

```bash
mkdir -p ~/app
```

Enter it:

```bash
cd ~/app
```

The production application path used in this deployment is:

```text
/home/ubuntu/app
```

---

# 4. Clone the Repository

Clone the backend repository into the application directory:

```bash
git clone <GITHUB_REPOSITORY_URL> .
```

Example:

```bash
git clone https://github.com/<GITHUB_OWNER>/<REPOSITORY>.git .
```

Verify:

```bash
ls
```

---

# 5. Check Git Status

```bash
git status
```

Expected:

```text
nothing to commit, working tree clean
```

---

# 6. Select the Production Branch

If production uses `main`:

```bash
git checkout main
```

Verify:

```bash
git branch
```

Expected:

```text
* main
```

If another branch is intentionally used for a particular environment, configure it accordingly.

---

# 7. Create the Production Environment File

Check whether the repository provides an example environment file:

```bash
find . -maxdepth 2 -type f \
  \( -name ".env.example" -o -name ".env.sample" -o -name "*.env.example" \)
```

Create the production environment file:

```bash
nano .env
```

Add the required production environment variables.

Do not copy secrets directly into source code.

---

# 8. Protect the Environment File

Set restrictive permissions:

```bash
chmod 600 .env
```

Verify that Git ignores the file:

```bash
git check-ignore .env
```

Expected:

```text
.env
```

The production `.env` must never be committed to Git.

---

# 9. Environment Variables

Typical backend variables may include:

```text
NODE_ENV
PORT

DATABASE_URI
DB_NAME

JWT_SECRET_KEY

AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
AWS_REGION
AWS_BUCKET_NAME

EMAIL_HOST
EMAIL_PORT
EMAIL_USER
EMAIL_PASS
EMAIL_FROM

FRONTEND_BASE_URL

RAZORPAY_KEY_ID
RAZORPAY_KEY_SECRET
RAZORPAY_WEBHOOK_SECRET

MSG91_AUTH_KEY
MSG91_TEMPLATE_ID

OTP_BYPASS_ENABLED
OTP_BYPASS_CODE
```

Only include variables actually required by the application.

Never commit real production credentials.

---

# 10. Dockerfile

Create a production-ready multi-stage Dockerfile.

Example:

```dockerfile
# syntax=docker/dockerfile:1

# ---------- Stage 1: Builder ----------
FROM node:22-alpine AS build

ENV HUSKY=0

WORKDIR /app

COPY package.json package-lock.json ./
RUN npm ci

COPY tsconfig.json ./
COPY src ./src
COPY views ./views

RUN npm run build


# ---------- Stage 2: Production ----------
FROM node:22-alpine AS production

ENV NODE_ENV=production \
    PORT=3000 \
    HUSKY=0

WORKDIR /app

COPY package.json package-lock.json ./

RUN npm pkg delete scripts.prepare \
    && npm ci --omit=dev \
    && npm cache clean --force

COPY --from=build --chown=node:node /app/dist ./dist

RUN mkdir -p /app/logs /app/uploads \
    && chown -R node:node /app/logs /app/uploads

USER node

EXPOSE 3000

HEALTHCHECK --interval=30s --timeout=5s --start-period=20s --retries=3 \
  CMD node -e "fetch('http://127.0.0.1:' + process.env.PORT + '/health').then(r => { if (!r.ok) process.exit(1) }).catch(() => process.exit(1))"

CMD ["node", "dist/index.js"]
```

---

# 11. Why Multi-Stage Builds?

The Dockerfile uses two stages:

```text
build
  |
  +-- install dependencies
  +-- compile TypeScript
  +-- generate dist
        |
        v
production
  |
  +-- install production dependencies only
  +-- copy dist
  +-- run as node user
```

This keeps the final production image smaller and avoids shipping development dependencies unnecessarily.

---

# 12. Run as a Non-Root User

The production image uses:

```dockerfile
USER node
```

This is a security best practice.

The Node.js application should not normally run as root inside the container.

---

# 13. Docker Compose

Create:

```text
compose.yaml
```

Example:

```yaml
services:
  api:
    build:
      context: .
      dockerfile: Dockerfile
      target: production

    image: ${IMAGE_NAME:-<PROJECT_NAME>-api:latest}

    container_name: <PROJECT_NAME>-api

    restart: unless-stopped

    init: true

    env_file:
      - .env

    environment:
      NODE_ENV: production
      PORT: 3000

    ports:
      - "127.0.0.1:${APP_PORT:-3000}:3000"

    volumes:
      - api_logs:/app/logs
      - api_uploads:/app/uploads

    read_only: true

    tmpfs:
      - /tmp:size=64m,mode=1777

    security_opt:
      - no-new-privileges:true

    cap_drop:
      - ALL

volumes:
  api_logs:
  api_uploads:
```

Replace:

```text
<PROJECT_NAME>
```

with the actual project name.

---

# 14. Important Docker Compose Settings

### Restart policy

```yaml
restart: unless-stopped
```

If the container crashes or the EC2 server restarts, Docker will automatically attempt to restart the application.

---

### Localhost port binding

```yaml
ports:
  - "127.0.0.1:3000:3000"
```

This keeps the application private.

Nginx communicates with the application locally.

---

### Read-only filesystem

```yaml
read_only: true
```

This reduces the ability of the application to modify the container filesystem.

Writable locations are provided through:

```yaml
volumes:
  - api_logs:/app/logs
  - api_uploads:/app/uploads
```

and:

```yaml
tmpfs:
  - /tmp:size=64m,mode=1777
```

---

### Drop Linux capabilities

```yaml
cap_drop:
  - ALL
```

This reduces unnecessary Linux capabilities available to the container.

---

### No new privileges

```yaml
security_opt:
  - no-new-privileges:true
```

This helps prevent processes inside the container from gaining additional privileges.

---

# 15. Build the Docker Image Locally on EC2

For the initial manual deployment:

```bash
cd ~/app
```

Build:

```bash
docker compose build
```

Start:

```bash
docker compose up -d
```

Or:

```bash
docker compose up --detach
```

---

# 16. Check Container Status

```bash
docker compose ps
```

Expected:

```text
NAME
<PROJECT_NAME>-api

STATUS
Up ... (healthy)
```

The important part is:

```text
healthy
```

---

# 17. Check Application Health

Because the application is bound to localhost:

```bash
curl http://127.0.0.1:3000/health
```

Expected:

```text
Hello, I am alive
```

The exact response depends on the application's health endpoint.

---

# 18. View Docker Logs

From the application directory:

```bash
cd ~/app
```

Follow logs:

```bash
docker compose logs -f api
```

Last 100 lines:

```bash
docker compose logs --tail=100 api
```

All services:

```bash
docker compose logs
```

Important:

`docker compose logs` looks for `compose.yaml` in the current directory.

If you run it from:

```text
/home/ubuntu
```

instead of:

```text
/home/ubuntu/app
```

you may see:

```text
no configuration file provided
```

Use:

```bash
cd ~/app
```

first.

---

# 19. Restart the Application

```bash
docker compose restart api
```

---

# 20. Stop the Application

```bash
docker compose stop
```

---

# 21. Start the Application

```bash
docker compose start
```

---

# 22. Remove Containers

To stop and remove the containers:

```bash
docker compose down
```

Named volumes are not removed by the command above.

Be careful with:

```bash
docker compose down -v
```

This removes the Compose volumes and may delete application data stored in those volumes.

Do not use `-v` in production unless you intentionally want to delete the volumes.

---

# 23. Check Docker Resources

Running containers:

```bash
docker ps
```

All containers:

```bash
docker ps -a
```

Images:

```bash
docker images
```

Volumes:

```bash
docker volume ls
```

Docker disk usage:

```bash
docker system df
```

---

# 24. Initial Production Flow

For the first manual deployment:

```text
EC2
 |
 v
~/app
 |
 +-- Dockerfile
 +-- compose.yaml
 +-- .env
 +-- source code
 |
 v
docker compose build
 |
 v
docker compose up -d
 |
 v
Health check
 |
 v
Nginx
 |
 v
HTTPS
```

---

# 25. Production CI/CD Flow

After GitHub Actions is configured, EC2 should not build the application on every deployment.

The recommended flow is:

```text
Developer
    |
    | git push origin main
    v
GitHub Actions
    |
    +-- npm ci
    +-- lint
    +-- typecheck
    +-- build
    +-- Docker build
    |
    v
GHCR
    |
    | immutable image
    v
EC2
    |
    +-- docker compose pull
    |
    +-- docker compose up --no-build
    |
    v
New version live
```

This keeps application builds inside CI rather than consuming EC2 CPU and memory.

---

# 26. Immutable Image Tags

For CI/CD, use the Git commit SHA as the production image tag.

Example:

```text
ghcr.io/<github-owner>/<repository>:<commit-sha>
```

Example:

```text
ghcr.io/example/backend:af9c6bf18d945dbb5be04227e667e595c3289f5a
```

This allows a deployment to identify the exact source commit used to build the image.

A `latest` tag can also be maintained for convenience, but production deployment should preferably use the immutable SHA tag.

---

# 27. GHCR Authentication

The EC2 server needs permission to pull private images from GitHub Container Registry.

The deployment process will:

```bash
docker login ghcr.io
```

Then:

```bash
docker compose pull
```

The GitHub Actions deployment credentials should be stored as GitHub Environment Secrets.

Never commit GHCR credentials to the repository.

---

# 28. Docker Security Checklist

- [ ] Multi-stage Dockerfile
- [ ] Production dependencies only in final image
- [ ] Application runs as non-root
- [ ] Health check configured
- [ ] `restart: unless-stopped`
- [ ] Node.js port bound to localhost
- [ ] Read-only container filesystem where practical
- [ ] Temporary writable filesystem configured
- [ ] Linux capabilities dropped
- [ ] `no-new-privileges` enabled
- [ ] Production `.env` not committed
- [ ] Docker image uses immutable production tags
- [ ] EC2 does not publicly expose port 3000

---

# 29. Troubleshooting

### Container is not running

```bash
docker compose ps
```

Then:

```bash
docker compose logs --tail=100 api
```

---

### Health check is unhealthy

Check:

```bash
docker inspect <PROJECT_NAME>-api --format='{{json .State.Health}}'
```

Then check application logs:

```bash
docker compose logs --tail=100 api
```

---

### Port already in use

Check:

```bash
sudo ss -lntp | grep :3000
```

Also:

```bash
docker ps
```

---

### Compose file not found

If you see:

```text
no configuration file provided
```

go to the application directory:

```bash
cd ~/app
```

Then:

```bash
docker compose ps
```

---

# 30. Deployment Checklist

Before moving to Nginx:

- [ ] Repository cloned into `/home/ubuntu/app`
- [ ] Correct production branch selected
- [ ] `.env` created
- [ ] `.env` permissions set to `600`
- [ ] `.env` ignored by Git
- [ ] Docker installed
- [ ] Docker Compose installed
- [ ] Dockerfile created
- [ ] `compose.yaml` created
- [ ] Docker image builds successfully
- [ ] Container starts successfully
- [ ] Container is healthy
- [ ] `/health` works on `127.0.0.1:3000`
- [ ] Port 3000 is not publicly exposed
- [ ] Container uses `restart: unless-stopped`

---

# Next Step

After Docker is working, continue with:

[Nginx Setup](./nginx.md)