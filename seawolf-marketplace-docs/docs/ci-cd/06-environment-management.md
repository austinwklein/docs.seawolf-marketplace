# Environment Management

## Three-environment strategy

Seawolf Marketplace uses three separate environments to support development, testing, and production:

```
Developer's Machine (local development)
    ↓ (git push to test branch)
test.swe.ajklein.io (rapid iteration, experimental features)
    ↓ (git push to dev branch)
dev.swe.ajklein.io (pre-production validation, team review)
    ↓ (git push to prod branch)
swe.ajklein.io (production)
```

## Environment isolation

Each environment is completely isolated:

| Component | Test | Dev | Prod                  |
|-----------|------|-----|-----------------------|
| **Git branch** | test | dev | prod                  |
| **URL** | test.swe.ajklein.io | dev.swe.ajklein.io | swe.ajklein.io        |
| **Directory** | /opt/.../test/ | /opt/.../dev/ | /opt/.../prod/        |
| **Backend port** | 3000 | 3001 | 3002                  |
| **PM2 process** | backend-test | backend-dev | backend-prod          |
| **Database** | test | dev | seawolfmarketplace    |
| **Deploy user** | deploy-test | deploy-dev | deploy-prod           |
| **Backend user** | deploy-test | deploy-dev | deploy-prod           |
| **Nginx config** | 11-test_* | 12-dev_* | 20-swe_*              |
| **SSL cert** | swe.ajklein.io wildcard | swe.ajklein.io wildcard | swe.ajklein.io (main) |

**Key point**: Deploying to one environment should never affects the others.

## Environment promotion workflow

### Step 1: Develop and test locally

1. Clone repository: `git clone ...`
2. Create feature branch: `git checkout -b feature/my-feature`
3. Make code changes
4. Test locally with `npm start` for frontend and backend
5. Commit changes: `git commit -m "..."`

### Step 2: Push to test branch

1. Push to test branch: `git push origin feature/my-feature:test`
2. Go to GitHub Actions UI
3. Select **Angular Deployment** workflow
4. Click **Run workflow**
5. Select:
   - Branch: `test`
   - Environment: `Test Site`
6. Wait for deployment (2-5 minutes)
7. Visit **test.swe.ajklein.io** and verify changes
8. Check for errors in logs

### Step 3: Merge to dev branch

Once test environment looks good:

1. Merge test into dev: `git checkout dev && git merge test`
2. Push to dev: `git push origin dev`
3. Manually trigger deployment (same steps as above, but select `dev` branch and `Dev Site`)
4. Visit **dev.swe.ajklein.io** for team review
5. Address any feedback or issues

### Step 4: Merge to prod branch

When dev is approved:

1. Merge dev into prod: `git checkout prod && git merge dev`
2. Push to prod: `git push origin prod`
3. Manually trigger deployment to prod (select `prod` branch and `Prod Site`)
4. Monitor for issues

## Environment variables

Each environment has its own `.env` file:

```bash
# /opt/seawolf-marketplace/test/backend/.env
NODE_ENV=test
DATABASE_URL=postgresql://user:password@localhost/test
PORT=3000

# /opt/seawolf-marketplace/dev/backend/.env
NODE_ENV=dev
DATABASE_URL=postgresql://user:password@localhost/dev
PORT=3001

# /opt/seawolf-marketplace/prod/backend/.env
NODE_ENV=prod
DATABASE_URL=postgresql://user:password@localhost/seawolfmarketplace
PORT=3002
```

**Important**: `.env` files are **NOT tracked in Git** (in `.gitignore`). They must be manually created/maintained on the VPS.

### Setting up environment variables

On first deployment to a new VPS:

```bash
# Create directories
mkdir -p /opt/seawolf-marketplace/{test,dev,prod}/backend

# Create .env files for each environment
cat > /opt/seawolf-marketplace/test/backend/.env << EOF
NODE_ENV=test
DATABASE_URL=postgresql://postgres:password@localhost/test
PORT=3000
EOF

cat > /opt/seawolf-marketplace/dev/backend/.env << EOF
NODE_ENV=dev
DATABASE_URL=postgresql://postgres:password@localhost/dev
PORT=3001
EOF

cat > /opt/seawolf-marketplace/prod/backend/.env << EOF
NODE_ENV=prod
DATABASE_URL=postgresql://postgres:password@localhost/seawolfmarketplace
PORT=3002
EOF

# Set correct permissions
chmod 600 /opt/seawolf-marketplace/*/backend/.env
chown deploy-{test,dev,prod}:deploy-{test,dev,prod} /opt/seawolf-marketplace/*/backend/.env
```

### Updating environment variables

If backend configuration needs to change (e.g., adding a new API key):

1. SSH into the VPS
2. Edit the appropriate `.env` file:
   ```bash
   nano /opt/seawolf-marketplace/prod/backend/.env
   ```
3. Update the variable
4. Restart the PM2 process to load new variables:
   ```bash
   /opt/utils/pm2.restart --target prod
   ```

## Databases

Three separate PostgreSQL databases exist:

| Database             | Purpose                   | Data                                     |
|----------------------|---------------------------|------------------------------------------|
| `test`               | Testing, rapid iteration  | Test data, can be reset                  |
| `dev`                | Pre-production validation | Data mirrors prod structure              |
| `seawolfmarketplace` | Production                | Live data, student listings and accounts |

### Database connection

Each backend connects via environment variable `DATABASE_URL`:

```
postgresql://username:password@host:port/database_name
```

Example:
```
postgresql://postgres:my_password@127.0.0.1:5432/seawolfmarketplace
```

### Database initialization

On first deployment, SQL scripts create the schema:

```bash
/opt/seawolf-marketplace/{env}/database/sql_scripts/
├── create_database.sql       # Creates the database
├── create_tables.sql         # Creates user, listing, history, etc. tables
├── create_functions.sql      # Creates stored procedures (if any)
├── create_indexes.sql        # Creates indexes for performance
└── create_mock_data.sql      # Populates with test data
```

These scripts run **only if the database/tables don't already exist** (idempotent). Subsequent deployments skip this.

### Manual database access

Connect to a specific database:

```bash
# Connect to test database
psql -U postgres -d test

# Connect to prod database
psql -U postgres -d seawolfmarketplace

# List all databases
psql -U postgres -l

# List tables in current database
\dt
```

### Database backup

**Currently**: No automated backups (acceptable for class project, risky for production).

**Manual backup**:

```bash
# Backup a database
pg_dump -U postgres seawolfmarketplace > seawolfmarketplace_backup.sql

# Backup all databases
pg_dumpall -U postgres > all_databases_backup.sql

# Restore from backup
psql -U postgres -d seawolfmarketplace < seawolfmarketplace_backup.sql
```

## Git branch strategy

### Branch naming

- `test`: Always deployable, latest experimental code
- `dev`: Pre-production, team-reviewed and tested code
- `prod`: Production

### Merging strategy

```
feature-branch   →   test    →    dev    →    prod
       ↓               ↓           ↓            ↓
    (local)        (testing)  (pre-release)  (public)
```

1. **Feature branch**: Develop and test locally
2. **test branch**: Push and deploy to test environment, verify
3. **dev branch**: Merge from test, deploy to dev, team review
4. **prod branch**: Merge from dev, deploy to prod, final version

### Preventing accidental deployments

**Best practice**: Only Jacob (project manager) should have permission to merge into `dev` and `prod` branches.

This can be enforced in GitHub:
1. Go to Repository → Settings → Branches
2. Add branch protection rules for `dev` and `prod`
3. Require pull request reviews before merge
4. Dismiss stale reviews when new commits are pushed

## Traffic and data flow

### User traffic

- Production users access **swe.ajklein.io** → served by `backend-prod` and `seawolfmarketplace` database
- Team testing uses **test.swe.ajklein.io** and **dev.swe.ajklein.io**
- Each environment has separate user accounts and listings

**Note**: Users cannot see or interact with test/dev listings in production.

### Code flow during deployment

```
GitHub (test branch)
    → GitHub Actions runner
    → VPS (deploy-test user)
    → /opt/seawolf-marketplace/test/
    → Build frontend & backend
    → Update PM2 process
    → Update Nginx symlink
    → Changes live at test.swe.ajklein.io
```

## Troubleshooting by environment

### Test environment issues

Usually safe to fix quickly since it's for experimentation:

```bash
# Redeploy from test branch:
# Go to GitHub Actions, re-run workflow with test branch/Test Site

# Or manually:
cd /opt/seawolf-marketplace/test/
git fetch origin
git reset --hard origin/test
# Re-build and restart PM2 manually
```

### Dev environment issues

Pre-production, so more caution needed. Notify the team before making changes.

### Production issues

**Critical**. Follow incident response:

1. **Stop the bleeding**: If a bug is causing errors, `pm2 stop backend-prod` to take site down briefly
2. **Investigate**: Check logs and determine root cause
3. **Fix**: Either hotfix in prod branch or rollback to previous version
4. **Deploy**: Once fixed, test in test/dev first, then redeploy to prod
5. **Post-mortem**: Discuss what went wrong and how to prevent it, test thoroughly in test -> dev -> prod

## Environment-specific configurations

### Frontend configuration

Angular build includes environment-specific configs:

```
/frontend/src/environments/
├── environment.ts           # Dev/test default
├── environment.prod.ts      # Production
└── environment.test.ts      # Test
```

The backend also recognizes `NODE_ENV`:

```javascript
if (process.env.NODE_ENV === 'production') {
  // Production-specific code
} else {
  // Dev/test code
}
```

## Monitoring environment health

### Check all environments at once

```bash
# View all PM2 processes
pm2 list

# View all Nginx virtual hosts
sudo nginx -t

# Check database connections
psql -U postgres -d seawolfmarketplace -c "SELECT version();"
psql -U postgres -d dev -c "SELECT version();"
psql -U postgres -d test -c "SELECT version();"
```

### Monitor specific environment

```bash
# Test environment logs
pm2 logs backend-test --lines 50

# Test environment errors
/opt/utils/pm2_errors.view --target test

# Dev environment status
/opt/utils/pm2_errors.view --target dev

# Prod database connectivity
psql -U postgres -d seawolfmarketplace -c "SELECT COUNT(*) FROM users;"
```
