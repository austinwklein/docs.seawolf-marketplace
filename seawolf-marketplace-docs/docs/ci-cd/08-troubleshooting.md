# Troubleshooting Guide

## General troubleshooting approach

1. **Check workflow logs** on GitHub Actions for step-by-step details
2. **Check PM2 logs** on the VPS for backend errors
3. **Check Nginx logs** for frontend/proxy issues
4. **Check PostgreSQL logs** for database issues
5. **SSH into VPS** and manually verify processes/files

## Deployment workflow failures

### Workflow stuck in running state

**Symptom**: GitHub Actions shows "in progress" for hours

**Causes**:
- GitHub Actions runner lost connection
- Deployment script hanging waiting for input
- Long network timeout

**Solution**:
1. Go to GitHub Actions
2. Click the stuck workflow
3. Click **Cancel workflow**
4. Re-run the workflow

### SSH authentication failed

**Symptom**: `ssh: Permission denied (publickey)`

**Causes**:
- `SSH_PRIVATE_KEY` secret is wrong or corrupted
- SSH port in `PORT` secret is incorrect
- Host IP/domain in `HOST` secret is wrong
- SSH key permissions are incorrect on VPS

**Solution**:

```bash
# On local machine, verify key format
cat ~/.ssh/id_rsa | head -5
# Should show: -----BEGIN RSA PRIVATE KEY-----

# Check if key works locally
ssh -i ~/.ssh/id_rsa -p 2222 deploy-test@swe.ajklein.io
# (adjust port as needed)

# On VPS, check authorized_keys
cat ~/.ssh/authorized_keys | wc -l
# Should be non-zero

# Verify key permissions
ls -la ~/.ssh/
# Should see: authorized_keys (644 or 600), id_rsa (600)
```

### Git clone failed

**Symptom**: `fatal: could not read Username`

**Cause**: GitHub SSH key not configured on VPS

**Solution**:

```bash
# On VPS as deploy-{env} user, set up GitHub SSH key
ssh -T git@github.com
# If fails, add your public key to GitHub:
# https://github.com/settings/keys

# Or use HTTPS instead (in workflow, change REPO_URL):
REPO_URL=https://github.com/${{ github.repository }}.git
# (requires GitHub token in workflow)
```

### Frontend build failed

**Symptom**: `ERROR in src/app/...ts(...)` in workflow logs

**Causes**:
- TypeScript compilation error in Angular code
- Angular component import error
- Template syntax error

**Solution**:

1. Read the error message carefully:
   ```
   ERROR in src/app/pages/listing/listing.component.ts(42,5): 
   error TS2322: Type 'string' is not assignable to type 'number'
   ```

2. Fix the error in your code (line 42 of listing.component.ts)

3. Test locally:
   ```bash
   cd frontend
   npm ci
   ng build
   ```

4. Commit and push to `test` branch

5. Re-run workflow

### Backend build failed

**Symptom**: `error TS####: ...` in workflow logs

**Causes**:
- TypeScript syntax error
- Missing import statement
- Type mismatch

**Solution**:

1. Read the error:
   ```
   error TS2307: Cannot find module './db/connection'
   ```

2. Fix the import in your code

3. Test locally:
   ```bash
   cd backend
   npm ci
   npx tsc --noEmit  # Check without writing files
   ```

4. Commit, push, and re-run workflow

### Database initialization failed

**Symptom**: `ERROR: relation "users" already exists` or `psql: error: ...`

**Causes**:
- SQL syntax error in script
- File not found in repository
- PostgreSQL permission issue

**Solution**:

1. Check that `database/sql_scripts/` directory exists in your repo
2. Verify SQL syntax in scripts
3. On VPS, manually test:
   ```bash
   cd /opt/seawolf-marketplace/prod/database
   psql -U postgres -d seawolf_prod < sql_scripts/create_tables.sql
   ```
4. Fix any errors and re-run deployment

### PM2 restart failed

**Symptom**: `Error: spawn ENOENT` or `Process failed to start`

**Causes**:
- Backend `index.js` doesn't exist or is invalid
- Port already in use
- Missing dependencies

**Solution**:

```bash
# On VPS, check if index.js exists
ls -la /opt/seawolf-marketplace/prod/backend/index.js

# Check if port is available
lsof -i :3002

# Manually start and see error
cd /opt/seawolf-marketplace/prod/backend
NODE_ENV=prod node index.js

# Or check PM2 logs
pm2 logs backend-prod --err --lines 50
```

### Symlink update failed

**Symptom**: `Permission denied` when updating symlink

**Causes**:
- `/usr/share/nginx/SWE/` directory doesn't exist
- Wrong file permissions
- Deploy user lacks write permission

**Solution**:

```bash
# On VPS, check permissions
ls -la /usr/share/nginx/SWE/

# Create directory if needed
sudo mkdir -p /usr/share/nginx/SWE

# Set correct ownership and permissions
sudo chown -R deploy-prod:deploy-prod /usr/share/nginx/SWE
sudo chmod -R 755 /usr/share/nginx/SWE

# Verify
ls -la /usr/share/nginx/SWE/
```

## Runtime issues (after deployment)

### Website returns 500 error

**Symptom**: Browser shows "500 Internal Server Error"

**Causes**:
- Backend process crashed
- Unhandled exception in Node.js
- Database connection failed

**Solution**:

```bash
# On VPS, check backend status
pm2 list

# Check logs
pm2 logs backend-prod --err --lines 100

# Restart backend
pm2 restart backend-prod

# Check database connectivity
psql -U postgres -d seawolf_prod -c "SELECT 1;"

# Check if port is accessible
curl http://127.0.0.1:3002/api/health
```

### API requests fail with 502 Bad Gateway

**Symptom**: Browser shows "502 Bad Gateway"

**Cause**: Nginx can't reach backend on specified port

**Solution**:

```bash
# On VPS, verify backend is running
pm2 list | grep backend-prod

# Check port is listening
ss -tlnp | grep 3002

# Verify Nginx proxy configuration
sudo nginx -t

# Check Nginx error log
sudo tail -f /var/log/nginx/error.log

# Manually test connection
curl http://127.0.0.1:3002/
```

### Frontend doesn't load (404 errors)

**Symptom**: Page loads but shows "Cannot GET /"

**Causes**:
- Symlink points to wrong directory
- Build output files don't exist
- Nginx root directive is wrong

**Solution**:

```bash
# On VPS, check symlink
ls -la /usr/share/nginx/SWE/prod_root

# Should point to valid build directory:
# prod_root -> /opt/seawolf-marketplace/prod/frontend/dist/frontend/browser

# Verify build files exist
ls -la /opt/seawolf-marketplace/prod/frontend/dist/frontend/browser/

# Check Nginx config
sudo nginx -t

# Check Nginx access log
sudo tail -f /var/log/nginx/access.log | grep prod
```

### Frontend pages return 404 after navigation

**Symptom**: Works fine until user navigates, then 404

**Causes**: Angular SPA routing issue (Angular router not handling route)

**Solution**: This is typically an Angular/frontend issue, not CI/CD. Check that:
1. Routes are defined in `app.routing.ts` or `app.routes.ts`
2. Nginx is serving `index.html` for all routes (SPA requirement)

Nginx should be configured to serve `index.html` for any non-API route:

```nginx
location / {
  try_files $uri $uri/ /index.html;
}
```

### Database connection refused

**Symptom**: `Error: connect ECONNREFUSED 127.0.0.1:5432`

**Causes**:
- PostgreSQL not running
- Wrong host/port in `DATABASE_URL`
- `DATABASE_URL` not set in `.env`

**Solution**:

```bash
# On VPS, check PostgreSQL is running
systemctl status postgresql

# Start if needed
sudo systemctl start postgresql

# Verify connection string in .env
cat /opt/seawolf-marketplace/prod/backend/.env | grep DATABASE_URL

# Test connection manually
psql -U postgres -d seawolf_prod

# Check PostgreSQL logs
sudo tail -f /var/log/postgresql/postgresql*.log
```

### New code changes not showing up

**Symptom**: Deployed code but old version still running

**Causes**:
- Frontend files not updated
- Backend process still running old code
- Symlink not updated

**Solution**:

```bash
# On VPS, verify new files exist
ls -la /opt/seawolf-marketplace/prod/frontend/dist/frontend/browser/
# Check modification time

# Check symlink target
ls -la /usr/share/nginx/SWE/prod_root

# Restart PM2 to force load new code
pm2 restart backend-prod

# Clear browser cache (Ctrl+Shift+Delete)
# Or use incognito window

# Force refresh (Ctrl+F5 or Cmd+Shift+R)
```

### High memory or CPU usage

**Symptom**: Server becomes slow, processes using lots of resources

**Causes**:
- Memory leak in backend code
- Infinite loop or inefficient algorithm
- Database query returning too much data

**Solution**:

```bash
# Monitor in real-time
pm2 monit

# Check logs for clues
pm2 logs backend-prod --lines 500 | tail -100

# Identify specific issue
# - If memory keeps growing: likely memory leak
# - If CPU stays high: likely inefficient code
# - If intermittent: might be specific slow query

# Temporary fix: restart the process
pm2 restart backend-prod

# Permanent fix: find and fix the bug in code
```

## SSL/TLS certificate issues

### Certificate expired or invalid

**Symptom**: Browser shows "SSL Certificate Error"

**Causes**:
- Let's Encrypt certificate expired
- Renewal failed
- Wrong certificate path in Nginx

**Solution**:

```bash
# On VPS, check certificate expiration
sudo certbot certificates

# Renew manually
sudo certbot renew --force-renewal

# Check Nginx config points to correct cert
sudo cat /etc/nginx/sites-enabled.d/20-swe_ajklein_io.conf | grep ssl_certificate

# Reload Nginx
sudo systemctl reload nginx

# Verify certificate is valid
echo | openssl s_client -connect swe.ajklein.io:443 2>/dev/null | grep -A2 "subject="
```

### Self-signed certificate warning

**Symptom**: Browser warns "Your connection is not private"

**Cause**: Using test certificate instead of production

**Solution**:
- In test/dev environments, this is expected
- Tell team: "This is a self-signed cert, click 'Advanced' and proceed"
- For production, ensure Let's Encrypt certificate is installed

## Checking logs

### GitHub Actions workflow logs

1. Go to **Actions** tab on GitHub
2. Select the workflow run
3. Click on a failed step to see output

### PM2 backend logs

```bash
# Live logs
pm2 logs backend-prod

# Last 50 lines
pm2 logs backend-prod --lines 50 --nostream

# Error logs only
pm2 logs backend-prod --err --lines 100 --nostream

# Using utility script
/opt/utils/pm2_errors.view --target prod --lines 100
```

### Nginx access/error logs

```bash
# Access log (all requests)
sudo tail -f /var/log/nginx/access.log

# Error log (problems)
sudo tail -f /var/log/nginx/error.log

# Search for specific error
sudo grep "error" /var/log/nginx/error.log | tail -20

# Filter by date/time
sudo tail -f /var/log/nginx/access.log | grep "12/Nov/2025"
```

### PostgreSQL logs

```bash
# PostgreSQL log file (location varies)
sudo tail -f /var/log/postgresql/postgresql*.log

# Or check systemd journal
sudo journalctl -u postgresql -f
```

## Emergency procedures

### Website is completely down

1. **Check PM2**: `pm2 list` - is backend running?
2. **Check Nginx**: `sudo systemctl status nginx` - is web server running?
3. **Check DB**: Can you connect? `psql -U postgres -d seawolf_prod`
4. **Restart services**:
   ```bash
   pm2 restart backend-prod
   sudo systemctl restart nginx
   sudo systemctl restart postgresql
   ```
5. **Check logs** to understand what happened
6. **If no fix works**: Rollback to previous version

### Rolling back to previous deployment

```bash
# Check available versions
cd /opt/seawolf-marketplace/prod
git log --oneline | head -10

# Switch to previous commit
git reset --hard <previous-commit-hash>

# Rebuild and restart
npm ci
ng build --configuration=production
cd backend
npm ci
npx tsc
pm2 restart backend-prod
```

### Getting help

If stuck:

1. Collect information:
   - Error messages from logs
   - Last deployment time
   - What code changed
   - What was working before

2. Check documentation (this guide)

3. Ask team members

4. Contact Austin (infrastructure lead) for server-level help

## Performance optimization

If deployment succeeds but performance is slow:

```bash
# Check build bundle size
du -sh /opt/seawolf-marketplace/prod/frontend/dist/

# Check backend memory
pm2 show backend-prod

# Check database query performance
psql -U postgres -d seawolf_prod
> EXPLAIN ANALYZE SELECT * FROM listings WHERE category = 'Books';

# Monitor network
nethogs -d 10  # Monitor for 10 seconds
```
