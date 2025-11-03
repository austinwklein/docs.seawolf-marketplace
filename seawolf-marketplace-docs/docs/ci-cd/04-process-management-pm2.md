# Process Management (PM2)

## What is PM2?

PM2 is a Node.js process manager that:

- Keeps the backend Node.js process running 24/7
- Automatically restarts the process if it crashes
- Manages multiple processes (one per environment: test, dev, prod)
- Provides logging and monitoring capabilities
- Persists process state across server reboots

## Why PM2?

- **Lightweight**: No system administration overhead
- **User-level**: Can be run by non-root users (security benefit)
- **Simple**: Easy to configure and manage
- **Reliable**: Battle-tested in production

**Alternative**: systemd service files (more complex but more powerful)

## PM2 processes for Seawolf Marketplace

Three backend processes run simultaneously on the VPS:

| Process | Environment | Port | Command |
|---------|-------------|------|---------|
| `backend-test` | Test | 3000 | `pm2 start /opt/seawolf-marketplace/test/backend/index.js --name backend-test` |
| `backend-dev` | Dev | 3001 | `pm2 start /opt/seawolf-marketplace/dev/backend/index.js --name backend-dev` |
| `backend-prod` | Prod | 3002 | `pm2 start /opt/seawolf-marketplace/prod/backend/index.js --name backend-prod` |

Each process:
- Runs the compiled Express.js backend (`index.js`)
- Listens on a unique port
- Reads configuration from `.env` file in its directory
- Logs to PM2's log files
- Can be restarted independently

## PM2 process lifecycle

### Starting a process

During deployment, PM2 starts a new process:

```bash
NODE_ENV=test pm2 start /opt/seawolf-marketplace/test/backend/index.js \
  --name backend-test \
  --cwd /opt/seawolf-marketplace/test/backend
```

**Flags**:
- `--name`: Human-readable name for the process
- `--cwd`: Working directory (where to run from)
- `NODE_ENV`: Environment variable (passed to Node.js)

### Stopping a process

Before redeploying, the old process is stopped gracefully:

```bash
pm2 stop backend-test
```

**Behavior**: Sends SIGTERM signal to process, allowing it to clean up connections

### Deleting a process

After stopping, the process is removed from PM2's list:

```bash
pm2 delete backend-test
```

**Behavior**: Removes the process definition; subsequent reboots won't restart it

### Restarting a process

To restart without redeploying:

```bash
pm2 restart backend-test
```

**Behavior**: Stops the old process and starts a new one with the same configuration

### Auto-restart on crash

If a Node.js process crashes, PM2 automatically restarts it. Log in to check why:

```bash
# View most recent logs
pm2 logs backend-prod --lines 50

# View error logs
/opt/utils/pm2.errors.view --target prod
```

## PM2 state persistence

### Saving state

After starting or restarting processes, state is saved:

```bash
pm2 save
```

**Effect**: PM2 stores the process list in a state file

### Restoring state after reboot

When the VPS reboots:

```bash
pm2 startup    # Enable PM2 to start on system boot
pm2 resurrect  # Restore previous process list after boot
```

This is done once during VPS setup. After that, PM2 automatically restarts all processes listed in the saved state.

## Viewing PM2 status

### List all processes

```bash
pm2 list
```

Output shows:
- Process name
- PID (process ID)
- Status (online, stopped, errored)
- CPU usage
- Memory usage
- Uptime

### View logs

```bash
# View live logs for a specific process
pm2 logs backend-prod

# View 50 most recent lines (non-streaming)
pm2 logs backend-prod --lines 50 --nostream

# View error logs
pm2 logs backend-prod --err --lines 50 --nostream
```

### Monitor all processes

```bash
# Interactive dashboard with live updates
pm2 monit
```

Press `q` to exit.

## PM2 utility scripts

Helper scripts in `/opt/utils/` simplify common operations  
All scripts default to all environments but will accept --target to specify {test|dev|prod}
### View errors

```bash
./pm2.errors.view                  # View errors for all processes
./pm2.errors.view --target prod    # View errors for specific environment
./pm2.errors.view -n 100           # View last 100 lines
./pm2.errors.view -t dev -n 50     # Combine options
```

Located at: `/opt/utils/pm2.errors.view`

### Flush logs

```bash
./pm2.logs.flush                   # Flush logs for all processes
./pm2.logs.flush --target test     # Flush logs for specific environment
```

Located at: `/opt/utils/pm2.logs.flush`

Use this to clean up log files that grow too large.

### Restart processes

```bash
./pm2.restart                      # Restart all backend processes
./pm2.restart --target dev         # Restart specific environment
```

Located at: `/opt/utils/pm2.restart`

Use this to restart processes manually without redeploying code.

## Environment variables for PM2 processes

Each PM2 process reads environment variables from its `.env` file:

```bash
# /opt/seawolf-marketplace/test/backend/.env
NODE_ENV=test
DATABASE_URL=postgresql://user:password@localhost/seawolf_test
PORT=3000
```

When the process starts, PM2 loads these variables into the Node.js process. The backend code accesses them via `process.env.VARIABLE_NAME`.

**Important**: Environment variables are **NOT changed during deployment**. They persist across deployments. Only update `.env` files manually if configuration needs to change.

## Checking process health

### Process is not running

```bash
pm2 list
```

If process shows status `stopped` or `errored`:

```bash
# View why it stopped
pm2 logs backend-prod --err --lines 100

# Try to restart it
pm2 restart backend-prod

# If it crashes immediately, check the backend logs:
pm2 logs backend-prod --lines 100
```

### Port already in use

If startup fails with "Port already in use":

```bash
# Find what's using the port
lsof -i :3002

# Kill the old process
kill -9 <PID>

# Then restart PM2 process
pm2 restart backend-prod
```

### Memory leak or high CPU

Monitor process resources:

```bash
pm2 monit
```

Watch the CPU and memory columns. If they keep growing, there may be a bug in the backend code. Look at recent changes and check logs.

### Logs too large

PM2 logs can grow large over time. Clean them up:

```bash
/opt/utils/pm2.logs.flush --target prod
```

Or flush all:

```bash
/opt/utils/pm2.logs.flush
```

## PM2 files and locations

| File/Directory               | Purpose |
|------------------------------|---|
| `/opt/utils/pm2.restart`     | Restart utility script |
| `/opt/utils/pm2.errors.view` | View error logs script |
| `/opt/utils/pm2.logs.flush`  | Flush logs script |
| `~/.pm2/logs/`               | PM2 log files directory |
| `~/.pm2/dump.pm2`            | PM2 state persistence file |

Most files are in the home directory of the user running PM2 (usually the deploy user or root).

## Common PM2 commands

```bash
# List processes
pm2 list

# View logs
pm2 logs <process-name>
pm2 logs <process-name> --lines <number>
pm2 logs <process-name> --err

# Start/stop/restart
pm2 start <app.js>
pm2 stop <process-name>
pm2 restart <process-name>
pm2 delete <process-name>

# Monitoring
pm2 monit
pm2 show <process-name>

# State management
pm2 save
pm2 resurrect

# Cleanup
pm2 flush
pm2 kill
```

## Troubleshooting PM2 issues

| Issue | Solution |
|-------|----------|
| Process keeps crashing | Check `pm2 logs backend-prod --err` for error messages. Fix the bug in backend code and redeploy. |
| PM2 won't start | Check that Node.js is installed: `node --version`. Reinstall PM2 if needed: `npm install -g pm2`. |
| Can't access process | Verify you're running PM2 commands as the correct user. Each environment has its own user (`deploy-test`, `deploy-dev`, `deploy-prod`). |
| Logs not showing | Try `pm2 flush` to reset logs, then restart: `pm2 restart <process-name>`. |
| Process not restarting after reboot | Run `pm2 startup` and `pm2 save` to enable auto-restart. |

## Integration with deployment

During deployment, PM2 is used to:

1. **Stop** the old process (graceful shutdown)
2. **Delete** the old process definition
3. **Start** a new process with updated code
4. **Save** the new state

The deployment script handles this automatically:

```bash
pm2 stop "$SERVICE_NAME" 2>/dev/null || true
pm2 delete "$SERVICE_NAME" 2>/dev/null || true
sleep 1
NODE_ENV=$ENV pm2 start "$BACKEND_DIR/index.js" --name "$SERVICE_NAME" --cwd "$BACKEND_DIR"
pm2 save
```

This ensures zero downtime during redeployment (or minimal downtime, depending on how quickly the new process starts).
