# CI/CD Pipeline Overview

## What is this pipeline?

The Seawolf Marketplace uses a **manual-trigger, GitHub Actions-based deployment pipeline** to automate building, testing, and deploying the application to three separate environments: test, dev, and prod.

The pipeline orchestrates:

- **Code checkout** from GitHub
- **Frontend build** (Angular compilation to static assets)
- **Backend build** (TypeScript compilation to JavaScript)
- **Database initialization** (schema setup for new environments)
- **Process management** (PM2 restart/configuration)
- **Static file serving** (Nginx symlink updates)

## Why manual trigger?

For a class project, **manual deployment gives the team explicit control** over when code goes live. This prevents accidental deployments and ensures team alignment before production deployments.

Automatic deployment on commit could be added in the future if desired.

## Key technologies

| Component | Technology | Purpose |
|-----------|-----------|---------|
| **Automation** | GitHub Actions | Orchestrates deployment workflow |
| **Version Control** | Git | Tracks code changes by branch |
| **Frontend Build** | Angular CLI (ng build) | Compiles TypeScript/HTML/SCSS to static files |
| **Backend Build** | TypeScript Compiler (tsc) | Compiles TypeScript to JavaScript |
| **Process Manager** | PM2 | Runs and restarts Node.js backend process |
| **Web Server** | Nginx | Serves frontend static files and proxies `/api/` to backend |
| **Database** | PostgreSQL | Persists user data, listings, history, etc. |

## Three-environment promotion path

```
Developer's Machine (local)
    ↓
test branch / test.swe.ajklein.io (rapid iteration)
    ↓
dev branch / dev.swe.ajklein.io (pre-production validation)
    ↓
prod branch / swe.ajklein.io (graded by instructors)
```

Each environment has:
- Separate Git branch
- Separate backend Node.js process (PM2)
- Separate Nginx virtual host
- Separate PostgreSQL database
- Separate SSL certificate (shared for test/dev)

## Deployment workflow at a glance

```
1. Developer pushes code to branch (test/dev/prod)
2. Developer manually triggers workflow from GitHub Actions UI
3. Workflow selects deployment environment and branch
4. GitHub Actions runner:
   - Checks out code from the branch
   - SSHes into VPS as deploy-{environment} user
   - Clones or updates code from GitHub
   - Builds frontend (npm ci → ng build)
   - Builds backend (npm ci → tsc)
   - Initializes database (if needed)
   - Restarts PM2 backend process
   - Updates Nginx symlink to new build
   - Sets correct file permissions
5. Changes go live on server (2-5 minutes)
```

## Security model

**Principle: Least privilege**

- GitHub Actions only has access to deploy via **SSH keys for specific deploy users** (`deploy-test`, `deploy-dev`, `deploy-prod`)
- These deploy users **only have permission** to modify code in their respective directories
- No root access or password logins to VPS
- SSH uses asymmetric keys, not passwords
- Each environment is isolated from others

If GitHub secrets are compromised, an attacker can only deploy code, not access other systems or data.

## Monitoring deployments

All deployments are logged with:
- Timestamp of deployment
- Environment (test/dev/prod)
- Branch deployed
- Build success/failure status
- PM2 process status
- Frontend/backend build outputs

View logs in the GitHub Actions UI or check PM2 logs on the server (see [Manual Operations](09-manual-operations.md)).
