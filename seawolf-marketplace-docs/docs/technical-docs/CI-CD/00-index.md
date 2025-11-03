# CI/CD Pipeline Documentation Index

Welcome to the Seawolf Marketplace CI/CD Pipeline documentation. This guide covers everything from deployment architecture to manual troubleshooting.

## Quick Links

### For Quick Reference

- **[CI/CD Overview](01-cicd-overview.md)** - What is this pipeline and why do we use it?
- **[Quick Troubleshooting](08-troubleshooting.md#deployment-workflow-failures)** - Something went wrong?
- **[Manual Operations](09-manual-operations.md)** - How do I check status or restart services?

### For Learning the System

1. **[CI/CD Overview](01-cicd-overview.md)** - Start here for the big picture
2. **[Deployment Architecture](02-deployment-architecture.md)** - Understand the system design
3. **[GitHub Actions Workflow](03-github-actions-workflow.md)** - How deployments are triggered
4. **[Environment Management](06-environment-management.md)** - How test/dev/prod are set up

### For Specific Tasks

- **Deploying code**: See [GitHub Actions Workflow](03-github-actions-workflow.md#triggering-a-deployment)
- **Troubleshooting failed deployment**: See [Troubleshooting](08-troubleshooting.md#deployment-workflow-failures)
- **Restarting a process**: See [Manual Operations](09-manual-operations.md#managing-processes)
- **Viewing logs**: See [Manual Operations](09-manual-operations.md#viewing-logs)
- **Checking database**: See [Manual Operations](09-manual-operations.md#database-operations)

## Documentation Structure

### 1. Overview & Architecture

```
01-cicd-overview.md
├─ What is the pipeline?
├─ Why manual trigger?
├─ Key technologies
├─ Promotion path (test → dev → prod)
└─ Security model

02-deployment-architecture.md
├─ System diagram
├─ Directory structure
├─ Deployment users & permissions
├─ Environment variables
└─ Nginx proxying
```

### 2. Deployment Process

```
03-github-actions-workflow.md
├─ How to trigger deployment
├─ Workflow inputs & parameters
├─ Step-by-step explanation
├─ Build process details
└─ Troubleshooting workflow errors

07-build-pipeline.md
├─ Frontend build (Angular)
│  ├─ ng build process
│  ├─ Build output
│  └─ Common errors
├─ Backend build (TypeScript)
│  ├─ tsc compilation
│  ├─ tsconfig.json
│  └─ Common errors
└─ Database initialization
   ├─ SQL scripts
   └─ Idempotency
```

### 3. Runtime Management

```
04-process-management-pm2.md
├─ What is PM2?
├─ Process lifecycle
├─ PM2 utility scripts
│  ├─ Restart
│  ├─ View errors
│  └─ Flush logs
└─ Troubleshooting PM2

05-nginx-configuration.md
├─ Nginx overview
├─ Virtual hosts (test/dev/prod)
├─ Frontend serving
├─ API proxying
└─ SSL/TLS configuration

06-environment-management.md
├─ Three-environment strategy
├─ Promotion workflow
├─ Environment variables
├─ Databases
└─ Traffic flow
```

### 4. Support & Operations

```
08-troubleshooting.md
├─ Workflow failures (SSH, git, builds)
├─ Runtime issues (500 errors, 502 errors)
├─ Logs and diagnostics
└─ Emergency procedures

09-manual-operations.md
├─ SSH access
├─ Process management (manual restart)
├─ Viewing logs and status
├─ Database operations
├─ File operations
└─ Common workflows & checklists
```

## Key Concepts

### Environments

| Environment | URL | Purpose | Branch |
|---|---|---|---|
| **Test** | test.swe.ajklein.io | Rapid iteration, experimental | test |
| **Dev** | dev.swe.ajklein.io | Pre-production validation | dev |
| **Prod** | swe.ajklein.io | Public-facing, graded | prod |

Each environment is completely isolated with separate processes, databases, and code.

### Deployment flow

```
1. Developer commits code
   ↓
2. Push to test/dev/prod branch
   ↓
3. Manually trigger GitHub Actions (via web UI)
   ↓
4. Select branch & environment
   ↓
5. Workflow runs on GitHub Actions runner
   ├─ Checkout code
   ├─ SSH to VPS
   ├─ Update code
   ├─ Build frontend (ng build)
   ├─ Build backend (tsc)
   ├─ Initialize database (if needed)
   ├─ Restart PM2 process
   └─ Update Nginx symlink
   ↓
6. Changes live (2-5 minutes)
```

### Technology stack

| Layer | Technology | Port |
|-------|-----------|------|
| **Frontend** | Angular v20 | served via Nginx |
| **Backend** | Express.js (Node.js) | 3000/3001/3002 |
| **Process Manager** | PM2 | manages backend |
| **Web Server** | Nginx | proxies requests |
| **Database** | PostgreSQL | localhost:5432 |

## Common Tasks

### Deploy new code

1. Push to test branch: `git push origin feature:test`
2. Go to GitHub → Actions → Angular Deployment
3. Click "Run workflow"
4. Select branch=`test`, environment=`Test Site`
5. Wait 2-5 minutes
6. Visit test.swe.ajklein.io to verify

See: [Triggering Deployments](03-github-actions-workflow.md#triggering-a-deployment)

### Check if deployment succeeded

```bash
# Via GitHub Actions UI:
# - Green checkmark = success
# - Red X = failure
# - Click for detailed logs

# Via VPS:
pm2 list                              # Check processes
pm2 logs backend-prod --lines 20      # Check recent logs
curl https://swe.ajklein.io/api/ping  # Test API
```

See: [Manual Operations](09-manual-operations.md#viewing-process-status)

### Restart a process manually

```bash
/opt/utils/pm2.restart --target prod
# or
pm2 restart backend-prod
```

See: [Process Management](04-process-management-pm2.md#pm2-utility-scripts)

### View error logs

```bash
/opt/utils/pm2_errors.view --target prod --lines 100
# or
pm2 logs backend-prod --err --lines 100 --nostream
```

See: [Viewing Logs](09-manual-operations.md#viewing-logs)

### Check database

```bash
psql -U postgres -d seawolf_prod
> SELECT COUNT(*) FROM listings;
> \dt  # list tables
> \q  # quit
```

See: [Database Operations](09-manual-operations.md#database-operations)

### Troubleshoot deployment failure

1. Check GitHub Actions UI for error message
2. Identify which step failed (git, build, deployment)
3. Read error message carefully
4. Fix code or configuration
5. Re-run deployment

See: [Troubleshooting Guide](08-troubleshooting.md)

## Checklists

### Pre-deployment

- [ ] Code tested locally
- [ ] No TypeScript errors (`npm run lint` or `ng build`)
- [ ] Changes pushed to test branch
- [ ] Ready to deploy?

### Post-deployment

- [ ] Workflow shows green checkmark
- [ ] Website loads at environment URL
- [ ] API calls work (check Network tab in DevTools)
- [ ] No errors in PM2 logs

### Promoting to production

- [ ] Tested in test environment ✓
- [ ] Tested in dev environment ✓
- [ ] Team approved changes
- [ ] Ready for grading/public release
- [ ] Jacob approved deployment

## Support

### Who to contact

| Issue | Contact |
|-------|---------|
| Deployment/infrastructure | Austin |
| Frontend code | Wa |
| UI/UX/styling | Hunter |
| Project management | Jacob |

### Resources

- **GitHub**: https://github.com/seawolf-marketplace
- **Documentation**: This guide
- **Project Plan**: See `Summit_Project_Plan.pdf`
- **System Architecture**: See `System_Architecture.pdf`

## Frequently asked questions

**Q: How do I deploy code?**
A: See [Triggering Deployments](03-github-actions-workflow.md#triggering-a-deployment). Use GitHub Actions UI for manual trigger.

**Q: What if deployment fails?**
A: See [Troubleshooting](08-troubleshooting.md). Check GitHub Actions logs for error message.

**Q: Can I deploy directly to production?**
A: Yes, but we recommend testing in test/dev first to catch issues early.

**Q: How long does deployment take?**
A: Typically 2-5 minutes. Depends on build times and network speed.

**Q: How do I check if my code is running?**
A: See [Viewing Process Status](09-manual-operations.md#viewing-process-status). Use `pm2 list` on VPS.

**Q: Can I manually restart a process?**
A: Yes, use `/opt/utils/pm2.restart --target prod` or `pm2 restart backend-prod`.

**Q: Where are logs?**
A: See [Viewing Logs](09-manual-operations.md#viewing-logs). Use PM2 logs or utility scripts.

**Q: What if PM2 keeps crashing?**
A: See [Troubleshooting PM2](08-troubleshooting.md#pm2-restart-failed). Check error logs for root cause.

**Q: How do I check the database?**
A: See [Database Operations](09-manual-operations.md#database-operations). Use psql command-line tool.

## Document versions

| Date | Author | Version | Notes |
|------|--------|---------|-------|
| 2025-11-02 | Austin | 1.0 | Initial CI/CD documentation |

## Next steps

1. **Read** [CI/CD Overview](01-cicd-overview.md) to understand the big picture
2. **Review** [GitHub Actions Workflow](03-github-actions-workflow.md) to see how deployments work
3. **Bookmark** [Troubleshooting](08-troubleshooting.md) for when things go wrong
4. **Save** [Manual Operations](09-manual-operations.md) for common tasks
5. **Ask questions** if anything is unclear

---

**Last updated**: November 2, 2025  
**Maintained by**: Austin (Infrastructure Lead)  
**For issues/questions**: Contact Austin or see [Support](#support)
