# GitHub Actions Workflow

## Workflow file location

```
repository-root/
└── .github/
    └── workflows/
        └── angular-deploy.yml
```

## Triggering a deployment

### Via GitHub UI

1. Go to your GitHub repository
2. Click **Actions** tab
3. Select **Angular Deployment** workflow from the left sidebar
4. Click **Run workflow** button
5. Select options:
   - **Branch**: Which code branch to deploy (`test`, `dev`, or `prod`)
   - **Environment**: Where to deploy (`Test Site`, `Dev Site`, or `Prod Site`)
6. Click **Run workflow**

## Workflow inputs

The workflow accepts two required input parameters:

### `branch` (dropdown)

- **Options**: `test`, `dev`, `prod`
- **Default**: `test`
- **Purpose**: Specifies which Git branch to check out and deploy
- **Requirement**: Must match the branch name in your repository

### `environment` (dropdown)

- **Options**: `Test Site`, `Dev Site`, `Prod Site`
- **Default**: `Test Site`
- **Purpose**: Specifies deployment target and environment
- **Behind the scenes**: Gets mapped to `test`, `dev`, or `prod` for internal use

## Workflow steps explained

### 1. Map environment choices to variables

```yaml
- name: Map environment choices to variables
  id: map_environment
  run: |
    case "${{ inputs.environment }}" in
      "Test Site") echo "environment=test" >> $GITHUB_OUTPUT ;;
      "Dev Site") echo "environment=dev" >> $GITHUB_OUTPUT ;;
      "Prod Site") echo "environment=prod" >> $GITHUB_OUTPUT ;;
    esac
```

**Purpose**: Converts human-friendly environment names (`Test Site`) to internal variable names (`test`)

**Output**: Stores result in `steps.map_environment.outputs.environment` for use in later steps

### 2. Checkout code

```yaml
- name: Checkout code
  uses: actions/checkout@v4
```

**Purpose**: Clones the repository to the GitHub Actions runner

**Note**: This always checks out the branch specified in the workflow input via the GitHub Actions UI

### 3. Deploy via SSH

```yaml
- name: Deploy via SSH
  uses: appleboy/ssh-action@v1.0.3
  with:
    host: ${{ secrets.HOST }}
    username: deploy-${{ steps.map_environment.outputs.environment }}
    key: ${{ secrets.SSH_PRIVATE_KEY }}
    port: ${{ secrets.PORT }}
    script: |
      # [deployment script contents]
```

**Purpose**: Establishes SSH connection to VPS and executes the deployment script

**Parameters**:
- `host`: VPS hostname/IP from GitHub secret
- `username`: `deploy-test`, `deploy-dev`, or `deploy-prod` based on environment
- `key`: Private SSH key for authentication (stored in GitHub secrets)
- `port`: Non-standard SSH port (stored in GitHub secret for security)
- `script`: Bash script that runs on the VPS (see "Deployment script" section below)

## Deployment script execution

The deployment script runs on the VPS with these main phases:

### Variables setup

```bash
APP_NAME=seawolf-marketplace
ENV=${{ steps.map_environment.outputs.environment }}
DEPLOY_DIR=/opt/$APP_NAME/$ENV
DB_DIR="$DEPLOY_DIR/database"
WEB_ROOT=/usr/share/nginx/SWE/${ENV}_root
REPO_URL=ssh://git@github.com/${{ github.repository }}.git
BRANCH=${{ inputs.branch }}
FRONTEND_DIR="$DEPLOY_DIR/frontend"
BACKEND_DIR="$DEPLOY_DIR/backend"
SERVICE_NAME="backend-$ENV"
```

These variables define paths and names used throughout deployment.

### Git operations

```bash
cd "$DEPLOY_DIR"
if [ ! -d .git ]; then
  git clone -b "$BRANCH" "$REPO_URL" .
else
  git fetch origin
  git checkout -f "$BRANCH"
  git reset --hard origin/"$BRANCH"
fi
```

**Purpose**: Clone repository on first deployment, or update existing code

**Behaviors**:
- **First time**: Clones the specified branch into the deploy directory
- **Subsequent times**: Fetches latest commits and hard resets to remote branch
- **Result**: Working directory always matches the latest commit on the specified branch

### Database initialization (idempotent)

```bash
if [ -d "$DB_DIR/sql_scripts" ]; then
  cd "$DB_DIR"
  psql -U postgres < sql_scripts/create_database.sql
  psql -U postgres -d $ENV < sql_scripts/create_tables.sql
  psql -U postgres -d $ENV < sql_scripts/create_functions.sql
  psql -U postgres -d $ENV < sql_scripts/create_indexes.sql
  psql -U postgres -d $ENV < sql_scripts/create_mock_data.sql
fi
```

**Purpose**: Set up database schema and mock data for new environments

**Idempotency**: Scripts are designed to check for existence before creating (e.g., `IF NOT EXISTS`), so they can run safely multiple times

**Files required**: 
- `sql_scripts/create_database.sql`
- `sql_scripts/create_tables.sql`
- `sql_scripts/create_functions.sql`
- `sql_scripts/create_indexes.sql`
- `sql_scripts/create_mock_data.sql`

(Place these in your repository's `database/sql_scripts/` directory)

### Frontend build

```bash
cd "$FRONTEND_DIR"
npm cache verify
npm ci
if [ "$ENV" = "prod" ]; then
  ng build --configuration=production
else
  ng build
fi
```

**Purpose**: Build Angular SPA to static files

**Steps**:
1. Verify npm cache for corrupted packages
2. Clean install dependencies (more reliable than `npm install` in CI)
3. Compile TypeScript, bundle, optimize
4. Output to `dist/frontend/browser/`

**Production optimization**: Uses `--configuration=production` flag for prod, which enables:
- Ahead-of-Time (AOT) compilation
- Minification
- Tree-shaking
- Source map generation

### Backend build

```bash
cd "$BACKEND_DIR"
npm ci
npx tsc
```

**Purpose**: Compile TypeScript backend to JavaScript

**Steps**:
1. Clean install dependencies
2. Run TypeScript compiler (`tsc`) as specified in `tsconfig.json`
3. Output compiled JavaScript files to `dist/` (or as configured)

**Note**: Unlike frontend, prod and non-prod builds are identical. Optimization happens at runtime.

### Process management (PM2)

```bash
pm2 stop "$SERVICE_NAME" 2>/dev/null || true
pm2 delete "$SERVICE_NAME" 2>/dev/null || true
sleep 1

NODE_ENV=$ENV pm2 start "$BACKEND_DIR/index.js" \
  --name "$SERVICE_NAME" \
  --cwd "$BACKEND_DIR"

pm2 save
```

**Purpose**: Restart Node.js backend process with updated code

**Steps**:
1. Stop the running PM2 process (ignore errors if not running)
2. Delete the process definition
3. Wait 1 second for cleanup
4. Start a new process from the compiled `index.js`
5. Set `NODE_ENV` environment variable
6. Name the process `backend-test`, `backend-dev`, or `backend-prod`
7. Set working directory to the backend folder
8. Save PM2 state so process restarts after server reboot

### Symlink update

```bash
mkdir -p "$WEB_ROOT"
ln -sfn "$FRONTEND_DIR/dist/frontend/browser" "$WEB_ROOT"
```

**Purpose**: Point Nginx to the new frontend build

**Flags**:
- `-s`: Create symbolic link (not hard link)
- `-f`: Force replacement of existing symlink
- `-n`: Don't follow existing symlinks

**Result**: Nginx immediately serves the new static files without restart

### Permissions

```bash
chown -R $USER:$USER "$DEPLOY_DIR"
chown -R $USER:$USER "$WEB_ROOT"
chmod -R 755 "$DEPLOY_DIR"
chmod -R 755 "$WEB_ROOT"
```

**Purpose**: Ensure the deploy user can modify files in future deployments

**Note**: `$USER` is the deploy-{env} user (set by SSH)

### Verification

```bash
echo "Frontend: $(ls -la $FRONTEND_DIR/dist/frontend/browser | head -1)"
echo "Backend: $(pm2 list | grep $SERVICE_NAME)"
echo "Database: $(psql -U postgres -d $ENV -c 'SELECT COUNT(*) FROM listings;' 2>/dev/null || echo 'Database check skipped')"
echo "Deployment of $ENV completed successfully at $(date)"
```

**Purpose**: Verify deployment succeeded and log status

**Checks**:
- Frontend build directory exists and has files
- PM2 process is running
- Database is accessible and has data
- Timestamp of completion

## Workflow status and logs

After starting a deployment:

1. GitHub Actions displays workflow progress in real-time
2. Each step shows as **running** → **completed** or **failed**
3. Click on a step to view its output logs
4. If deployment fails, check the step where it failed (usually build or SSH steps)
5. Logs are retained for 90 days by GitHub

## Troubleshooting workflow failures

| Error | Likely cause | Solution |
|-------|--------------|----------|
| "SSH authentication failed" | Wrong SSH key or credentials | Verify `SSH_PRIVATE_KEY` secret contains correct private key |
| "Port connection refused" | Wrong SSH port | Verify `PORT` secret matches VPS SSH port |
| "Git clone failed" | SSH key not authorized for GitHub | Ensure deploy user has GitHub SSH key configured |
| "npm install failed" | Network issue or corrupted cache | Check node_modules permissions on VPS |
| "ng build failed" | TypeScript compilation error | Fix Angular build errors in code; check error message in logs |
| "tsc compilation failed" | TypeScript errors in backend | Fix TypeScript errors; check `tsconfig.json` settings |
| "PM2 start failed" | Port already in use or permission denied | Check if process already running; verify permissions |
| "Symlink update failed" | Permission denied on web root | Check file permissions and deploy user ownership |

## Environment variables during deployment

The workflow has access to:

```yaml
- ${{ github.repository }}        # repository owner/name
- ${{ inputs.branch }}            # user-selected branch
- ${{ inputs.environment }}       # user-selected environment
- ${{ secrets.HOST }}             # VPS hostname/IP
- ${{ secrets.PORT }}             # SSH port
- ${{ secrets.SSH_PRIVATE_KEY }}  # Private SSH key
```

These are substituted into the workflow before execution.
