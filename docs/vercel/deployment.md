# Vercel Deployment

This guide explains how to deploy a frontend application to Vercel and connect it with a custom domain.

The setup can be reused for React, Next.js, and other supported frontend projects.

---

# 1. Deployment Architecture

The frontend is hosted on Vercel.

```text
Developer
    |
    v
GitHub
    |
    v
Vercel
    |
    v
Frontend
    |
    v
Custom Domain
```

The backend can be hosted separately on AWS EC2.

Example architecture:

```text
Frontend
    |
    | HTTPS
    v
Vercel
    |
    | API requests
    v
https://api.example.com
    |
    v
AWS EC2
    |
    v
Nginx
    |
    v
Docker
    |
    v
Node.js API
```

---

# 2. Prerequisites

You should have:

- GitHub repository
- Frontend project
- Vercel account
- Domain name
- Production backend API URL if the frontend communicates with a backend

Make sure the frontend works locally before deploying.

---

# 3. Push Frontend to GitHub

From the frontend project:

```bash
git status
```

Add files:

```bash
git add .
```

Commit:

```bash
git commit -m "initial frontend deployment"
```

Push:

```bash
git push origin main
```

Use the production branch intended for deployment.

---

# 4. Create a Vercel Project

Open Vercel and choose:

```text
Add New Project
```

Import the GitHub repository.

Select the frontend repository.

Vercel automatically detects many common frameworks.

For example:

```text
Next.js
React
Vite
```

Review the detected settings before deploying.

---

# 5. Configure Build Settings

Vercel normally detects these automatically.

Common examples:

### Next.js

```text
Framework Preset: Next.js
Build Command: next build
Output Directory: Automatically detected
Install Command: npm install
```

### Vite

```text
Framework Preset: Vite
Build Command: npm run build
Output Directory: dist
Install Command: npm install
```

Do not change settings unnecessarily if Vercel has detected the project correctly.

---

# 6. Environment Variables

Production frontend environment variables should be configured in Vercel.

Go to:

```text
Project
    |
    v
Settings
    |
    v
Environment Variables
```

Example:

```text
VITE_API_URL=https://api.example.com
```

For Next.js, public browser variables normally use:

```text
NEXT_PUBLIC_API_URL=https://api.example.com
```

Use the variable prefix required by the frontend framework.

Do not commit production secrets to GitHub.

---

# 7. Important Frontend Environment Rule

Frontend variables are often bundled into browser JavaScript.

Therefore:

```text
Do not put private secrets in frontend environment variables.
```

Never expose:

```text
DATABASE_PASSWORD
JWT_SECRET
AWS_SECRET_ACCESS_KEY
RAZORPAY_KEY_SECRET
```

Frontend environment variables should contain only values that are safe for the browser to receive.

---

# 8. Deploy the Project

Click:

```text
Deploy
```

Vercel will:

```text
Clone GitHub repository
        |
        v
Install dependencies
        |
        v
Build application
        |
        v
Deploy
```

After deployment, Vercel provides a URL similar to:

```text
https://project-name.vercel.app
```

Open it and verify the application.

---

# 9. Automatic Deployments

When the GitHub repository is connected to Vercel:

```text
git push origin main
        |
        v
Vercel detects change
        |
        v
Build
        |
        v
Deploy
```

This means developers normally do not need to manually upload the frontend.

---

# 10. Production and Preview Deployments

A common workflow is:

```text
main
 |
 +----> Production deployment
```

Feature branches or pull requests can create preview deployments:

```text
feature branch
      |
      v
Vercel Preview
```

This allows changes to be tested before merging into production.

---

# 11. Custom Domain

Go to:

```text
Vercel
    |
    v
Project
    |
    v
Settings
    |
    v
Domains
```

Add your domain.

Example:

```text
example.com
```

You can also add:

```text
www.example.com
```

Vercel will show the DNS records that need to be configured.

---

# 12. DNS Configuration

If your domain DNS is managed by Hostinger, keep DNS management there.

For example:

```text
Domain
   |
   v
Hostinger DNS
   |
   +----> Vercel
   |
   +----> Backend API / EC2
```

Do not move DNS providers unless there is a reason to do so.

Add the exact DNS records provided by Vercel.

Do not guess DNS values.

---

# 13. Root Domain and WWW

If both are required:

```text
example.com
www.example.com
```

Add both to the Vercel project.

Then configure the DNS records according to Vercel's instructions.

Choose one as the canonical domain and redirect the other if required.

---

# 14. Backend API Configuration

If the frontend communicates with an API, configure the production API URL.

Example:

```text
https://api.example.com
```

For Vite:

```text
VITE_API_URL=https://api.example.com
```

For Next.js:

```text
NEXT_PUBLIC_API_URL=https://api.example.com
```

The frontend should not use:

```text
http://127.0.0.1:3000
```

or:

```text
http://localhost:3000
```

in production.

---

# 15. CORS

The backend must allow requests from the production frontend domain.

Example:

```text
https://example.com
```

If using a `www` domain:

```text
https://www.example.com
```

Avoid allowing every origin in production unless there is a specific requirement.

---

# 16. HTTPS

Vercel automatically provides HTTPS for Vercel deployments and configured custom domains.

After DNS configuration, verify:

```text
https://example.com
```

The browser should show a secure HTTPS connection.

---

# 17. Deployment Verification

After deployment, check:

```text
Frontend:
https://example.com
```

Check the backend API separately:

```text
https://api.example.com/health
```

Then verify that the frontend can communicate with the backend.

Use the browser Developer Tools:

```text
Developer Tools
    |
    +-- Network
    |
    +-- Console
```

Check for:

- API errors
- CORS errors
- 404 errors
- 500 errors
- Environment variable problems

---

# 18. Redeploy

Normally, pushing a new commit is enough:

```bash
git add .
git commit -m "update frontend"
git push origin main
```

Vercel automatically starts a new deployment.

You can also manually redeploy a previous deployment from the Vercel dashboard when needed.

---

# 19. Rollback

If a production deployment has a problem:

```text
Vercel
  |
  v
Deployments
  |
  v
Select previous working deployment
```

Use Vercel's deployment controls to restore a known-good version.

Before rollback, check the deployment logs to understand the failure.

---

# 20. Build Logs

If deployment fails, open:

```text
Vercel
    |
    v
Project
    |
    v
Deployments
    |
    v
Failed Deployment
    |
    v
Build Logs
```

Common causes:

- Dependency installation failure
- TypeScript errors
- Build errors
- Missing environment variables
- Incorrect framework configuration
- Incorrect Node.js version

---

# 21. Production Checklist

- [ ] GitHub repository connected
- [ ] Correct production branch selected
- [ ] Framework detected correctly
- [ ] Build command verified
- [ ] Output directory verified where applicable
- [ ] Production environment variables configured
- [ ] No private secrets exposed to frontend
- [ ] Custom domain added
- [ ] DNS records configured
- [ ] HTTPS working
- [ ] Production API URL configured
- [ ] Backend CORS configured
- [ ] Frontend tested
- [ ] Production deployment verified
- [ ] Preview deployments tested when needed

---

# 22. Related Documentation

AWS EC2:

```text
../aws/ec2-setup.md
```

Security Group:

```text
../aws/security-group.md
```

Docker:

```text
../backend/docker-deployment.md
```

Nginx:

```text
../backend/nginx.md
```

SSL / HTTPS:

```text
../backend/ssl-https.md
```

GitHub Actions:

```text
../cicd/github-actions.md
```

CloudWatch:

```text
../aws/cloudwatch.md
```

S3:

```text
../aws/s3.md
```