# GitHub Actions CI/CD

This guide explains how to configure a production CI/CD pipeline for a Dockerized backend deployed on AWS EC2.

The pipeline uses:

- GitHub Actions
- GitHub Container Registry (GHCR)
- Docker
- Docker Compose
- SSH
- AWS EC2

The goal is:

```text
Developer
    |
    | git push origin main
    v
GitHub
    |
    v
GitHub Actions
    |
    +----------------------+
    |                      |
    v                      v
   CI                    Build
    |                      |
    |                      v
    |                     GHCR
    |                      |
    +----------+-----------+
               |
               v
          SSH to EC2
               |
               v
       docker compose pull
               |
               v
       docker compose up
               |
               v
          New API live
```

---

# 1. Production Architecture

The production server uses:

```text
AWS EC2
    |
    +-- /home/ubuntu/app
    |
    +-- Docker
    |
    +-- Docker Compose
    |
    +-- Nginx
    |
    +-- HTTPS
```

The application runs in Docker:

```text
127.0.0.1:3000
```

Nginx exposes:

```text
80
443
```

GitHub Actions deploys the Docker image to the EC2 server.

---

# 2. Production Branch

The production branch is:

```text
main
```

Deployment happens when code is pushed to:

```text
main
```

Pull requests to `main` run CI validation but do not deploy.

---

# 3. CI/CD Flow

The workflow is:

```text
Pull Request
    |
    v
Validate
    |
    +-- npm ci
    +-- lint
    +-- typecheck
    +-- build
    +-- Docker build
```

For a push to `main`:

```text
Push to main
    |
    v
Validate
    |
    v
Build production Docker image
    |
    v
Push image to GHCR
    |
    v
SSH to EC2
    |
    v
Upload compose.yaml
    |
    v
Upload production .env
    |
    v
Login to GHCR
    |
    v
docker compose pull
    |
    v
docker compose up --no-build
```

---

# 4. Why Use GHCR?

GitHub Container Registry stores Docker images associated with GitHub repositories.

Instead of building the application directly on EC2:

```text
EC2
  |
  +-- git pull
  +-- npm install
  +-- npm build
  +-- docker build
```

the recommended CI/CD approach is:

```text
GitHub Actions
  |
  +-- test
  +-- build
  +-- docker build
  +-- push image
        |
        v
      GHCR
        |
        v
       EC2
        |
        +-- docker pull
        +-- docker compose up
```

This keeps builds out of the production server.

---

# 5. EC2 Production Directory

The existing production application directory is:

```text
/home/ubuntu/app
```

Do not create a second deployment directory unless there is a specific reason.

The deployment configuration uses:

```text
DEPLOY_USER=ubuntu
DEPLOY_PATH=/home/ubuntu/app
```

---

# 6. Dedicated Deployment SSH Key

Do not use the normal personal EC2 `.pem` key for GitHub Actions.

Create a separate SSH key specifically for CI/CD.

Example on your local machine:

```bash
ssh-keygen -t ed25519 -C "github-actions-deploy"
```

Save it as:

```text
~/.ssh/github-actions-deploy
```

This creates:

```text
~/.ssh/github-actions-deploy
~/.ssh/github-actions-deploy.pub
```

---

# 7. Add the Public Key to EC2

Display the public key:

```bash
cat ~/.ssh/github-actions-deploy.pub
```

Copy it.

On EC2:

```bash
nano ~/.ssh/authorized_keys
```

Add the public key as a new line.

Save the file.

Set permissions:

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

---

# 8. Test the Deployment SSH Key

From the local machine:

```bash
ssh -i ~/.ssh/github-actions-deploy ubuntu@<ELASTIC_IP>
```

The connection should succeed.

Important:

The private key:

```text
~/.ssh/github-actions-deploy
```

must never be committed to GitHub.

---

# 9. GitHub Environment

Create a GitHub Environment named:

```text
production
```

Go to:

```text
GitHub Repository
→ Settings
→ Environments
→ New environment
```

Create:

```text
production
```

Deployment secrets should be stored in this environment.

---

# 10. GitHub Environment Secrets

Add the following secrets to:

```text
production
```

### DEPLOY_HOST

Value:

```text
<ELASTIC_IP>
```

Example:

```text
13.202.130.86
```

---

### DEPLOY_USER

Value:

```text
ubuntu
```

---

### DEPLOY_PATH

Value:

```text
/home/ubuntu/app
```

---

### DEPLOY_SSH_PRIVATE_KEY

Value:

The complete private key from:

```text
~/.ssh/github-actions-deploy
```

It should include:

```text
-----BEGIN OPENSSH PRIVATE KEY-----
...
-----END OPENSSH PRIVATE KEY-----
```

Never commit this key.

---

### SSH_KNOWN_HOSTS

Generate this from a trusted machine:

```bash
ssh-keyscan -H <ELASTIC_IP>
```

Example:

```bash
ssh-keyscan -H 13.202.130.86
```

Copy the output into:

```text
SSH_KNOWN_HOSTS
```

This prevents SSH from blindly trusting an unknown server host key.

---

### PRODUCTION_ENV

This secret contains the complete production `.env` file.

Example:

```text
NODE_ENV=production
PORT=3000
DATABASE_URI=...
JWT_SECRET_KEY=...
...
```

Do not commit:

```text
.env
```

to GitHub.

The workflow uploads this secret to:

```text
/home/ubuntu/app/.env
```

---

### GHCR_USERNAME

Use the GitHub username that owns/accesses the package.

Example:

```text
<github-username>
```

Keep the username in the secret instead of hardcoding it in deployment commands.

---

### GHCR_PULL_TOKEN

Create a GitHub token that can read the private GHCR package.

The token needs package read permission.

For a classic Personal Access Token, the relevant permission is:

```text
read:packages
```

Do not give write/delete permissions to the EC2 pull token unless they are actually required.

Store it as:

```text
GHCR_PULL_TOKEN
```

---

# 11. GitHub Token for Publishing

The workflow uses:

```yaml
permissions:
  contents: read
```

at the workflow level.

The deployment job overrides this with:

```yaml
permissions:
  contents: read
  packages: write
```

This allows the GitHub Actions job to publish the Docker image to GHCR using:

```text
GITHUB_TOKEN
```

The workflow does not need a separate personal token to push the image.

---

# 12. Important Difference Between GHCR Tokens

There are two different authentication operations.

### GitHub Actions → GHCR

Used to push the image:

```text
GITHUB_TOKEN
```

with:

```yaml
packages: write
```

### EC2 → GHCR

Used to pull the private image:

```text
GHCR_USERNAME
GHCR_PULL_TOKEN
```

with package read permission.

Conceptually:

```text
GitHub Actions
      |
      | GITHUB_TOKEN
      | packages: write
      v
     GHCR
      ^
      |
      | GHCR_PULL_TOKEN
      | read package
      |
     EC2
```

---

# 13. Repository Name and GHCR Naming

Docker image repository names must be lowercase.

GitHub usernames/repository names can contain uppercase characters, so do not blindly use:

```text
ghcr.io/${{ github.repository }}
```

if the resulting owner/repository value can contain uppercase characters.

Use a lowercase repository path when constructing the image name.

For example:

```yaml
env:
  IMAGE_REPOSITORY: ghcr.io/${{ github.repository_owner }}/${{ github.event.repository.name }}
```

Before using it in Docker tags, normalize it to lowercase.

A reliable approach is to create the image name in a shell step:

```yaml
- name: Set image name
  id: image
  shell: bash
  run: |
    IMAGE_REPOSITORY="ghcr.io/${GITHUB_REPOSITORY,,}"
    echo "image=$IMAGE_REPOSITORY" >> "$GITHUB_OUTPUT"
```

Then:

```yaml
${{ steps.image.outputs.image }}:${{ github.sha }}
```

This avoids errors such as:

```text
repository name must be lowercase
```

---

# 14. Workflow File

Create:

```text
.github/workflows/ci-cd.yml
```

Example:

```yaml
name: CI/CD

on:
  pull_request:
    branches: [main]

  push:
    branches: [main]

permissions:
  contents: read

concurrency:
  group: production-${{ github.ref }}
  cancel-in-progress: true

jobs:
  ci:
    name: Validate
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Node
        uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: npm

      - name: Install dependencies
        run: npm ci

      - name: Lint
        run: npm run lint

      - name: Check TypeScript types
        run: npm run typecheck

      - name: Build application
        run: npm run build

      - name: Build production image
        run: |
          docker build \
            --target production \
            --tag william-hazlitt-api:ci \
            .

  publish-and-deploy:
    name: Publish and deploy
    needs: ci

    if: github.event_name == 'push' && github.ref == 'refs/heads/main'

    runs-on: ubuntu-latest

    environment: production

    permissions:
      contents: read
      packages: write

    env:
      DEPLOY_HOST: ${{ secrets.DEPLOY_HOST }}
      DEPLOY_USER: ${{ secrets.DEPLOY_USER }}
      DEPLOY_PATH: ${{ secrets.DEPLOY_PATH }}
      DEPLOY_SSH_PRIVATE_KEY: ${{ secrets.DEPLOY_SSH_PRIVATE_KEY }}
      SSH_KNOWN_HOSTS: ${{ secrets.SSH_KNOWN_HOSTS }}
      PRODUCTION_ENV: ${{ secrets.PRODUCTION_ENV }}
      GHCR_USERNAME: ${{ secrets.GHCR_USERNAME }}
      GHCR_PULL_TOKEN: ${{ secrets.GHCR_PULL_TOKEN }}

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set lowercase image name
        id: image
        shell: bash
        run: |
          IMAGE_REPOSITORY="ghcr.io/${GITHUB_REPOSITORY,,}"

          echo "repository=$IMAGE_REPOSITORY" >> "$GITHUB_OUTPUT"
          echo "image=$IMAGE_REPOSITORY:${GITHUB_SHA}" >> "$GITHUB_OUTPUT"
          echo "latest=$IMAGE_REPOSITORY:latest" >> "$GITHUB_OUTPUT"

      - name: Log in to GHCR
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build and push image
        uses: docker/build-push-action@v6
        with:
          context: .
          target: production
          push: true
          tags: |
            ${{ steps.image.outputs.image }}
            ${{ steps.image.outputs.latest }}

      - name: Configure SSH
        shell: bash
        run: |
          mkdir -p ~/.ssh
          chmod 700 ~/.ssh

          printf '%s\n' "$DEPLOY_SSH_PRIVATE_KEY" > ~/.ssh/id_ed25519
          chmod 600 ~/.ssh/id_ed25519

          printf '%s\n' "$SSH_KNOWN_HOSTS" > ~/.ssh/known_hosts
          chmod 644 ~/.ssh/known_hosts

      - name: Upload deployment configuration
        shell: bash
        run: |
          ssh "$DEPLOY_USER@$DEPLOY_HOST" \
            "mkdir -p '$DEPLOY_PATH'"

          scp compose.yaml \
            "$DEPLOY_USER@$DEPLOY_HOST:$DEPLOY_PATH/compose.yaml"

          printf '%s' "$PRODUCTION_ENV" \
            | base64 \
            | ssh "$DEPLOY_USER@$DEPLOY_HOST" \
              "base64 --decode > '$DEPLOY_PATH/.env'"

          ssh "$DEPLOY_USER@$DEPLOY_HOST" \
            "chmod 600 '$DEPLOY_PATH/.env'"

      - name: Login to GHCR on EC2
        shell: bash
        run: |
          printf '%s' "$GHCR_PULL_TOKEN" \
            | ssh "$DEPLOY_USER@$DEPLOY_HOST" \
              "docker login ghcr.io \
                --username '$GHCR_USERNAME' \
                --password-stdin"

      - name: Deploy immutable image
        shell: bash
        env:
          IMAGE_NAME: ${{ steps.image.outputs.image }}
        run: |
          ssh "$DEPLOY_USER@$DEPLOY_HOST" \
            "cd '$DEPLOY_PATH' && \
             IMAGE_NAME='$IMAGE_NAME' docker compose pull && \
             IMAGE_NAME='$IMAGE_NAME' docker compose up \
               --detach \
               --remove-orphans \
               --no-build"

      - name: Verify deployment
        shell: bash
        run: |
          ssh "$DEPLOY_USER@$DEPLOY_HOST" \
            "cd '$DEPLOY_PATH' && \
             docker compose ps"
```

---

# 15. Docker Compose for CI/CD

The production `compose.yaml` should support both local builds and remote images.

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

The important part for CI/CD is:

```yaml
image: ${IMAGE_NAME:-<PROJECT_NAME>-api:latest}
```

Locally:

```text
IMAGE_NAME not set
        |
        v
<PROJECT_NAME>-api:latest
```

In production:

```text
IMAGE_NAME=<GHCR_IMAGE>:<COMMIT_SHA>
        |
        v
GHCR immutable image
```

---

# 16. Why `--no-build`?

The deployment command uses:

```bash
docker compose up --detach --remove-orphans --no-build
```

This tells Docker Compose:

```text
Do not build the application on EC2.
```

The image was already built by GitHub Actions and pushed to GHCR.

EC2 only needs to:

```text
pull → start
```

---

# 17. Why Use Commit SHA Tags?

The image is tagged with:

```text
${GITHUB_SHA}
```

Example:

```text
ghcr.io/example/backend:af9c6bf18d945dbb5be04227e667e595c3289f5a
```

This provides an immutable reference to the exact Git commit.

Deployment becomes:

```text
Git commit
     |
     v
Docker image
     |
     v
GHCR
     |
     v
EC2
```

This makes deployments easier to trace and roll back.

---

# 18. `latest` Tag

The workflow may also push:

```text
:latest
```

Example:

```text
ghcr.io/example/backend:latest
```

This is convenient for humans and tooling.

However, production deployment should use:

```text
:<COMMIT_SHA>
```

rather than relying only on:

```text
:latest
```

because `latest` changes over time.

---

# 19. Environment Secret Handling

The production environment is stored in:

```text
GitHub Environment Secret:
PRODUCTION_ENV
```

The workflow uploads it to:

```text
/home/ubuntu/app/.env
```

Then:

```bash
chmod 600 /home/ubuntu/app/.env
```

This means the production `.env` does not need to exist in Git.

---

# 20. Deployment SSH Security

GitHub Actions uses:

```text
DEPLOY_SSH_PRIVATE_KEY
```

The server stores the matching public key in:

```text
/home/ubuntu/.ssh/authorized_keys
```

The private key exists only inside the GitHub Actions runner during deployment.

The workflow also uses:

```text
SSH_KNOWN_HOSTS
```

instead of:

```text
StrictHostKeyChecking=no
```

This is safer because the server host key is explicitly trusted.

---

# 21. Deployment Sequence

When a developer runs:

```bash
git push origin main
```

GitHub Actions executes:

```text
1. Checkout source
        |
2. Install dependencies
        |
3. Lint
        |
4. TypeScript check
        |
5. Build application
        |
6. Build Docker image
        |
7. Push image to GHCR
        |
8. Connect to EC2
        |
9. Upload compose.yaml
        |
10. Upload production .env
        |
11. Login to GHCR
        |
12. Pull exact image
        |
13. Start container
        |
14. Verify container status
```

---

# 22. Pull Request Behavior

For:

```bash
git push origin feature/my-feature
```

followed by a pull request to `main`:

```text
CI runs
```

but:

```text
Deployment does NOT run
```

This is controlled by:

```yaml
if: github.event_name == 'push' &&
    github.ref == 'refs/heads/main'
```

---

# 23. Main Branch Behavior

For:

```bash
git push origin main
```

the pipeline runs:

```text
CI
 |
 v
Build
 |
 v
GHCR
 |
 v
EC2
```

A successful push to `main` therefore results in a production deployment.

Make sure repository permissions and branch protection rules match the team's deployment policy.

---

# 24. Concurrency

The workflow uses:

```yaml
concurrency:
  group: production-${{ github.ref }}
  cancel-in-progress: true
```

This prevents multiple overlapping deployments for the same branch.

For example, if several commits are pushed quickly, an older in-progress workflow can be cancelled in favor of the newer deployment.

Review this behavior if your team requires every commit to be deployed independently.

---

# 25. GitHub Actions Node Runtime Warnings

GitHub Actions may display warnings related to the Node.js runtime used internally by an action.

For example:

```text
Node 20 is being deprecated...
```

This may refer to the runtime used by the GitHub Action itself, not the Node.js version used to build the application.

The application's Node version is controlled separately:

```yaml
uses: actions/setup-node@v4
with:
  node-version: 22
```

Do not add insecure compatibility environment variables just to hide a warning unless there is a specific requirement.

Keep GitHub Actions dependencies updated.

---

# 26. Docker Build Cache

A Docker build may fail if a GitHub Actions cache backend is configured incorrectly.

For example:

```text
Cache export is not supported for the docker driver.
```

If this occurs, the simplest production workflow is to remove:

```yaml
cache-from: type=gha
cache-to: type=gha,mode=max
```

from `docker/build-push-action`.

The image can still be built and pushed normally.

Caching can be reintroduced later using a correctly configured Buildx driver.

---

# 27. Common GHCR Error

If Docker reports:

```text
repository name must be lowercase
```

the GHCR image repository contains uppercase characters.

Docker requires the repository path to be lowercase.

Use:

```bash
${GITHUB_REPOSITORY,,}
```

when constructing the image repository in Bash.

Example:

```bash
IMAGE_REPOSITORY="ghcr.io/${GITHUB_REPOSITORY,,}"
```

---

# 28. Deployment Verification

After deployment, SSH into EC2:

```bash
ssh ubuntu@<ELASTIC_IP>
```

Go to:

```bash
cd ~/app
```

Check:

```bash
docker compose ps
```

Expected:

```text
Up ... (healthy)
```

Check logs:

```bash
docker compose logs --tail=100 api
```

Check local health:

```bash
curl http://127.0.0.1:3000/health
```

Check public HTTPS:

```bash
curl https://api.<DOMAIN>/health
```

---

# 29. Rollback

Because production images use commit SHA tags, rollback can be performed by selecting a previous known-good image.

Example:

```text
ghcr.io/example/backend:<PREVIOUS_COMMIT_SHA>
```

On EC2:

```bash
cd ~/app
```

Then:

```bash
IMAGE_NAME="ghcr.io/example/backend:<PREVIOUS_COMMIT_SHA>" \
docker compose pull
```

Start:

```bash
IMAGE_NAME="ghcr.io/example/backend:<PREVIOUS_COMMIT_SHA>" \
docker compose up -d --no-build
```

Verify:

```bash
docker compose ps
```

Then:

```bash
curl http://127.0.0.1:3000/health
```

The exact rollback process should be tested before relying on it during an incident.

---

# 30. GitHub Secrets Checklist

GitHub Environment:

```text
production
```

Secrets:

- [ ] `DEPLOY_HOST`
- [ ] `DEPLOY_USER`
- [ ] `DEPLOY_PATH`
- [ ] `DEPLOY_SSH_PRIVATE_KEY`
- [ ] `SSH_KNOWN_HOSTS`
- [ ] `PRODUCTION_ENV`
- [ ] `GHCR_USERNAME`
- [ ] `GHCR_PULL_TOKEN`

---

# 31. Security Checklist

- [ ] Deployment uses a dedicated SSH key
- [ ] Private SSH key is stored only as a GitHub secret
- [ ] SSH public key is in EC2 `authorized_keys`
- [ ] `SSH_KNOWN_HOSTS` is configured
- [ ] Production `.env` is stored as a secret
- [ ] `.env` is not committed
- [ ] GHCR pull token has only required package access
- [ ] GitHub Actions uses `packages: write` only where needed
- [ ] EC2 does not expose port 3000 publicly
- [ ] Production image uses an immutable commit SHA
- [ ] Deployment does not build Docker images on EC2
- [ ] Deployment path is `/home/ubuntu/app`

---

# 32. Final CI/CD Architecture

```text
                  GitHub
                     |
                     |
              push to main
                     |
                     v
             GitHub Actions
                     |
          +----------+----------+
          |                     |
          v                     v
       Validate              Docker Build
          |                     |
          |                     v
          |                    GHCR
          |                     |
          +----------+----------+
                     |
                     v
                SSH to EC2
                     |
                     v
              /home/ubuntu/app
                     |
                     v
              docker login GHCR
                     |
                     v
              docker compose pull
                     |
                     v
        docker compose up --no-build
                     |
                     v
                 Docker API
                     |
                     v
                  Nginx
                     |
                     v
                   HTTPS
```

---

# 33. Final Deployment Checklist

### AWS

- [ ] EC2 running
- [ ] Elastic IP configured
- [ ] Security Group configured
- [ ] SSH restricted to required IPs
- [ ] HTTP 80 open
- [ ] HTTPS 443 open

### Docker

- [ ] Docker installed
- [ ] Docker Compose installed
- [ ] Dockerfile configured
- [ ] `compose.yaml` configured
- [ ] Health check configured
- [ ] `restart: unless-stopped`
- [ ] Application bound to localhost

### Nginx

- [ ] API DNS configured
- [ ] Nginx reverse proxy configured
- [ ] HTTP working
- [ ] HTTPS working
- [ ] SSL renewal tested

### GitHub

- [ ] Production environment created
- [ ] Deployment SSH key created
- [ ] Public key added to EC2
- [ ] `DEPLOY_HOST`
- [ ] `DEPLOY_USER`
- [ ] `DEPLOY_PATH`
- [ ] `DEPLOY_SSH_PRIVATE_KEY`
- [ ] `SSH_KNOWN_HOSTS`
- [ ] `PRODUCTION_ENV`
- [ ] `GHCR_USERNAME`
- [ ] `GHCR_PULL_TOKEN`

### CI/CD

- [ ] Workflow stored in `.github/workflows/ci-cd.yml`
- [ ] Pull requests run CI
- [ ] Push to `main` triggers deployment
- [ ] Docker image is pushed to GHCR
- [ ] Image repository is lowercase
- [ ] Commit SHA image tag is used
- [ ] EC2 pulls the image
- [ ] EC2 does not build the image
- [ ] Container starts successfully
- [ ] Health check passes
- [ ] Public HTTPS endpoint works

---

# Next Step

After GitHub Actions CI/CD is configured, continue with:

[CloudWatch Monitoring](../aws/cloudwatch.md)