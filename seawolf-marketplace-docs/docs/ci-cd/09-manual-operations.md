# Manual Operations and Commands

## Overview

While deployments are normally triggered through GitHub Actions, sometimes you need to manually manage processes on the VPS. This guide covers common manual operations.

## SSH access to VPS

### Initial SSH setup

```bash
# From your local machine, copy your public key
cat ~/.ssh/id_rsa.pub

# SSH into VPS (first time)
ssh -i ~/.ssh/id_rsa -p 2222 deploy-prod@swe.ajklein.io

# Or use the domain if available
ssh -i ~/.ssh/id_rsa -p 2222 austin@swe.ajklein.io
```

**Note**: Replace `2222` with the actual SSH port, `deploy-prod` with your user, and `swe.ajklein.io` with the VPS hostname/IP.

### SSH shortcuts (optional)

Add to `~/.ssh/config` on your local machine:

```
Host seawolf-prod
    HostName swe.ajklein.io
    User deploy-prod
    IdentityFile ~/.ssh/id_rsa
    Port 2222

Host seawolf-dev
    HostName swe.ajklein.io
    User deploy-dev
    IdentityFile ~/.ssh/id_rsa
    Port 2222

Host seawolf-test
    HostName swe.ajklein.io
    User deploy-test
    IdentityFile ~/.ssh/id_rsa
    Port 2222
```

Then you can simply:

```bash
ssh seawolf-prod
ssh seawolf-dev
ssh seawolf-test
```

## Viewing process status

### Check all processes

```bash
pm2 list
```

Output shows:
```
┌─────┬──────────────┬─────────┬─────────┬──────────┬─────────┬──────────────┐
│ id  │ name         │ version │ mode    │ pid      │ status  │ restart      │
├─────┼──────────────┼─────────┼─────────┼──────────┼─────────┼──────────────┤
│ 0   │ backend-test │ N/A     │ fork    │ 12345    │ online  │ 0            │
│ 1   │ backend-dev  │ N/A     │ fork    │ 12346    │ online  │ 0            │
│ 2   │ backend-prod │ N/A     │ fork    │ 12347    │ online  │ 0            │
└─────┴──────────────┴─────────┴─────────┴──────────┴─────────┴──────────────┘
```

**Status meanings**:
- `online`: Process is running
- `stopped`: Process was stopped manually
- `errored`: Process crashed
- `stopped-unexpected`: Process stopped unexpectedly

### Monitor processes in real-time

```bash
pm2 monit
```

Shows live CPU, memory, and request counts. Press `q` to exit.

### Get detailed process info

```bash
pm2 show backend-prod
```

Shows:
- Process details (name, pid, memory, CPU)
- Restart count
- Recent logs
- File watching status

## Managing processes

### Restart a process

```bash
pm2 restart backend-prod          # Restart one
pm2 restart backend-test backend-dev  # Restart multiple
/opt/utils/pm2.restart --target prod  # Using utility script
```

### Stop a process

```bash
pm2 stop backend-prod

# Stop all
pm2 stop all
```

**Note**: Stopping takes the website down. Use for maintenance or emergency stopping.

### Start a process

```bash
pm2 start backend-prod

# Or start all
pm2 start all

# Or manually start from index.js
NODE_ENV=prod pm2 start /opt/seawolf-marketplace/prod/backend/index.js \
  --name backend-prod \
  --cwd /opt/seawolf-marketplace/prod/backend
```

### Delete a process

```bash
pm2 delete backend-prod

# This removes it from PM2's list but doesn't delete files
```

## Viewing logs

### View live logs

```bash
pm2 logs backend-prod
```

Streams logs as they happen. Press `Ctrl+C` to exit.

### View recent logs (non-streaming)

```bash
pm2 logs backend-prod --lines 50 --nostream

# View error logs only
pm2 logs backend-prod --err --lines 50 --nostream

# View specific number of lines
pm2 logs backend-prod --lines 200 --nostream
```

### View logs for all processes

```bash
pm2 logs

# Just errors
pm2 logs --err
```

### Using utility scripts

```bash
# View errors for all environments
/opt/utils/pm2_errors.view

# View errors for specific environment
/opt/utils/pm2_errors.view --target prod
/opt/utils/pm2_errors.view --target dev
/opt/utils/pm2_errors.view --target test

# View specific number of lines
/opt/utils/pm2_errors.view --target prod --lines 100
/opt/utils/pm2_errors.view -t test -n 50
```

### Flush logs (clean up)

```bash
# Flush all process logs
/opt/utils/pm2_logs.flush

# Flush specific environment
/opt/utils/pm2_logs.flush --target prod
/opt/utils/pm2_logs.flush -t dev

# Or manually with PM2
pm2 flush
```

## Database operations

### Connect to database

```bash
# Connect to test database
psql -U postgres -d seawolf_test

# Connect to prod database
psql -U postgres -d seawolf_prod

# List all databases
psql -U postgres -l

# List tables in current database
\dt

# Quit psql
\q
```

### Query the database

```bash
# Count users
psql -U postgres -d seawolf_prod -c "SELECT COUNT(*) FROM users;"

# View listings
psql -U postgres -d seawolf_prod -c "SELECT id, title, price FROM listings LIMIT 10;"

# Check if table exists
psql -U postgres -d seawolf_prod -c "SELECT EXISTS (SELECT 1 FROM information_schema.tables WHERE table_name = 'users');"
```

### Backup database

```bash
# Backup single database
pg_dump -U postgres seawolf_prod > seawolf_prod_backup.sql

# Backup to compressed file
pg_dump -U postgres seawolf_prod | gzip > seawolf_prod_backup.sql.gz

# Backup all databases
pg_dumpall -U postgres > all_databases_backup.sql
```

### Restore database

```bash
# Restore from backup
psql -U postgres -d seawolf_prod < seawolf_prod_backup.sql

# Restore from compressed backup
gunzip -c seawolf_prod_backup.sql.gz | psql -U postgres -d seawolf_prod
```

## File operations

### Check deployment directory

```bash
# View directory structure
ls -la /opt/seawolf-marketplace/

# Check size of each environment
du -sh /opt/seawolf-marketplace/{test,dev,prod}

# List files in prod backend
ls -la /opt/seawolf-marketplace/prod/backend/

# Check if build exists
ls -la /opt/seawolf-marketplace/prod/frontend/dist/
```

### Check symlinks

```bash
# List Nginx symlinks
ls -la /usr/share/nginx/SWE/

# Check where a symlink points
readlink /usr/share/nginx/SWE/prod_root

# Update a symlink manually
sudo ln -sfn /opt/seawolf-marketplace/prod/frontend/dist/frontend/browser /usr/share/nginx/SWE/prod_root

# Verify symlink now works
ls -la /usr/share/nginx/SWE/prod_root/
```

### Check file permissions

```bash
# Check who owns a directory
ls -la /opt/seawolf-marketplace/prod/

# Change ownership to deploy user
sudo chown -R deploy-prod:deploy-prod /opt/seawolf-marketplace/prod

# Change permissions to allow write
sudo chmod -R 755 /opt/seawolf-marketplace/prod

# Make files readable/writable by owner only (more secure)
sudo chmod -R 700 /opt/seawolf-marketplace/prod
```

## Git operations

### Manually update code

```bash
# SSH into VPS
ssh seawolf-prod

# Navigate to deployment directory
cd /opt/seawolf-marketplace/prod

# Fetch latest code from GitHub
git fetch origin

# Check current branch
git branch

# Switch branch if needed
git checkout main
# or
git checkout prod

# Update to latest commit on current branch
git reset --hard origin/prod

# Verify you have latest code
git log --oneline | head -5
```

### Check deployment status

```bash
# View last commit deployed
cd /opt/seawolf-marketplace/prod
git log --oneline | head -1

# View when it was deployed
git log --format="%h %an %ai %s" | head -1

# Compare with GitHub (if network access)
git log --oneline origin/prod | head -5
```

## Nginx operations

### Check Nginx configuration

```bash
# Test syntax
sudo nginx -t

# Output should be:
# nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
# nginx: configuration will be successful
```

### Reload Nginx (apply config changes without downtime)

```bash
# Graceful reload
sudo systemctl reload nginx

# Full restart (brief downtime)
sudo systemctl restart nginx

# Check status
sudo systemctl status nginx
```

### View Nginx logs

```bash
# Access log (all requests)
sudo tail -f /var/log/nginx/access.log

# Error log (problems)
sudo tail -f /var/log/nginx/error.log

# Specific number of lines
sudo tail -n 50 /var/log/nginx/error.log

# Search logs
sudo grep "502 Bad Gateway" /var/log/nginx/error.log | tail -10
```

### Test connectivity to backend

```bash
# Test if backend is responding
curl http://127.0.0.1:3002/

# Test API endpoint
curl http://127.0.0.1:3002/api/health

# Test with headers
curl -H "Content-Type: application/json" http://127.0.0.1:3002/api/users
```

## System operations

### Check disk usage

```bash
# Overall disk usage
df -h

# Usage of specific directory
du -sh /opt/seawolf-marketplace

# Find large files
find /opt -type f -size +100M
```

### Check memory and CPU

```bash
# Overall system stats
free -h        # Memory
top -n 1       # Top processes

# For PM2 processes
pm2 monit
```

### Check system uptime

```bash
uptime
```

### View system logs

```bash
# Recent system logs
sudo journalctl -n 50

# Logs since boot
sudo journalctl -b

# Logs for specific service
sudo journalctl -u nginx
sudo journalctl -u postgresql
```

## Utility scripts location

All utility scripts are in `/opt/utils/`:

```bash
# List available scripts
ls -la /opt/utils/

# Make scripts executable if needed
chmod +x /opt/utils/*.restart
chmod +x /opt/utils/pm2*.view
chmod +x /opt/utils/pm2*.flush
```

## Common workflows

### After a deployment

1. Check processes:
   ```bash
   pm2 list
   ```

2. Check backend logs:
   ```bash
   pm2 logs backend-prod --lines 20 --nostream
   ```

3. Test website:
   ```bash
   curl https://swe.ajklein.io/api/listings
   ```

4. If all good, proceed. If issues, check troubleshooting guide.

### Emergency shutdown

```bash
# Stop all processes
pm2 stop all

# Or stop web server
sudo systemctl stop nginx

# Or stop database
sudo systemctl stop postgresql
```

### Emergency restart

```bash
# Restart all processes
pm2 restart all

# Or restart specific services
sudo systemctl start nginx
sudo systemctl start postgresql
```

### Full system restart

```bash
# Reboot VPS
sudo reboot

# Wait for VPS to come back online (~1 minute)
# PM2 will auto-restart processes after reboot (if configured)

# Verify all running
pm2 list
```

## Monitoring checklist

Regularly check:

```bash
# Processes running?
pm2 list

# Any errors in logs?
pm2 logs --err --lines 100 --nostream

# Database connectivity?
psql -U postgres -d seawolf_prod -c "SELECT 1;"

# Disk usage?
df -h | grep /

# Memory usage?
free -h

# Nginx running?
sudo systemctl status nginx

# Certificate expiration?
sudo certbot certificates
```

## Emergency contacts

| Role | Responsibility |
|------|-----------------|
| Austin | Server infrastructure, PM2, database, deployments |
| Jacob | Deployment approval, code merging |
| Wa | Frontend/backend code issues |
| Hunter | UI/UX issues |

Reach out to relevant person for issues in their area.
