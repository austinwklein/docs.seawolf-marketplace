# Deployment Architecture

## System diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                  GitHub Repository (Source of Truth)            │
│  (main repo with test, dev, prod branches)                      │
└────────────────────────────┬────────────────────────────────────┘
                             │ Developer pushes code
                             ↓
┌─────────────────────────────────────────────────────────────────┐
│              GitHub Actions (CI/CD Orchestration)               │
│  Workflow: .github/workflows/angular-deploy.yml                 │
│  Trigger: Manual workflow dispatch from UI                      │
└────────────────────────────┬────────────────────────────────────┘
                             │ SSH to VPS as deploy-{env} user
                             ↓
        ┌────────────────────────────────────────────────┐
        │     Hetzner VPS (Fedora 42 Server)             │
        │     Oregon data center                         │
        │                                                │
        ├──────────────────────────────────────────────┤
        │ /opt/seawolf-marketplace/                     │
        │  ├─ test/       (test environment)            │
        │  ├─ dev/        (dev environment)             │
        │  └─ prod/       (prod environment)            │
        │                                                │
        ├──────────────────────────────────────────────┤
        │ /usr/share/nginx/SWE/                         │
        │  ├─ test_root → /opt/.../test/frontend/...   │
        │  ├─ dev_root  → /opt/.../dev/frontend/...    │
        │  └─ prod_root → /opt/.../prod/frontend/...   │
        │                                                │
        ├──────────────────────────────────────────────┤
        │ Process Manager (PM2)                         │
        │  ├─ backend-test (port 3000)                  │
        │  ├─ backend-dev  (port 3001)                  │
        │  └─ backend-prod (port 3002)                  │
        │                                                │
        ├──────────────────────────────────────────────┤
        │ Web Server (Nginx)                            │
        │  ├─ test.swe.ajklein.io → port 3000           │
        │  ├─ dev.swe.ajklein.io  → port 3001           │
        │  └─ swe.ajklein.io      → port 3002           │
        │                                                │
        ├──────────────────────────────────────────────┤
        │ Database (PostgreSQL)                         │
        │  ├─ seawolf_test (test environment)           │
        │  ├─ seawolf_dev  (dev environment)            │
        │  └─ seawolf_prod (prod environment)           │
        └────────────────────────────────────────────────┘
```

## Deployment users and permissions

Each environment has a dedicated Linux user for SSH access:

| User | Permissions | Access |
|------|------------|--------|
| `deploy-test` | Read/write to `/opt/seawolf-marketplace/test/` only | Can deploy to test environment |
| `deploy-dev` | Read/write to `/opt/seawolf-marketplace/dev/` only | Can deploy to dev environment |
| `deploy-prod` | Read/write to `/opt/seawolf-marketplace/prod/` only | Can deploy to prod environment |

These users are created without shell access (login shell is `/usr/sbin/nologin`). They can only:
- Accept SSH key-based authentication
- Execute PM2 commands for their respective processes
- Read/write files in their respective directories

## Directory structure on VPS

```
/opt/seawolf-marketplace/
├── test/
│   ├── frontend/               (Angular source code)
│   │   └── dist/frontend/browser/  (build output)
│   ├── backend/                (Express/Node.js source)
│   ├── database/               (SQL init scripts)
│   └── .git/                   (Git repository)
│
├── dev/
│   └── [same structure as test]
│
└── prod/
    └── [same structure as test]

/usr/share/nginx/SWE/
├── test_root → /opt/seawolf-marketplace/test/frontend/dist/frontend/browser/
├── dev_root  → /opt/seawolf-marketplace/dev/frontend/dist/frontend/browser/
└── prod_root → /opt/seawolf-marketplace/prod/frontend/dist/frontend/browser/
```

**Why symlinks?**

Symlinks allow Nginx to serve frontend files from `/usr/share/nginx/SWE/` (Fedora default) while keeping application code in `/opt/` (Linux best practice). When a new build completes, we simply update the symlink to point to the latest build—no copying required.

## GitHub Actions secrets

Three secrets are stored in GitHub repository settings:

| Secret | Purpose | Value |
|--------|---------|-------|
| `HOST` | VPS hostname or IP | swe.ajklein.io (or IP) |
| `PORT` | SSH port (non-standard for security) | Custom port number |
| `SSH_PRIVATE_KEY` | SSH private key for deploy users | Private key of deploy-{env} users |

These secrets are passed to the GitHub Actions runner and used only during deployment. They are never logged or exposed in workflow output.

## Environment variables on VPS

Each environment has a `.env` file at:
- `/opt/seawolf-marketplace/{test|dev|prod}/.env`

These files are created manually and contain:
- `NODE_ENV={test|dev|prod}` (matches environment)
- `DATABASE_URL=postgresql://...` (connection string for that environment's database)
- Other backend configuration (API keys, ports, etc.)

The `.env` files are **NOT tracked in Git** (in `.gitignore`) and must be manually configured on the server. During deployment, the PM2 process reads these files.

## Deployment timeline

| Stage | Duration | Action |
|-------|----------|--------|
| Git operations | ~5-10s | Clone/fetch/checkout code from GitHub |
| Database init | ~2-5s | Run SQL scripts for schema (if needed) |
| Frontend build | ~30-60s | npm ci + ng build |
| Backend build | ~10-20s | npm ci + tsc |
| PM2 restart | ~5s | Stop old process, start new one |
| Symlink update | ~1s | Update Nginx symlink |
| Permissions | ~2s | Update file ownership/permissions |
| **Total** | **~2-5 minutes** | Entire deployment |

Actual time depends on npm cache, network speed, and build size.

## Nginx proxying

Nginx is configured to:

1. **Redirect HTTP to HTTPS** (on port 80)
2. **Serve frontend static files** from symlinked directory (on port 443)
3. **Proxy API requests** to the backend Node.js process

Example for production:

```nginx
server {
    listen 443 ssl;
    server_name swe.ajklein.io;
    
    # SSL certificates from Let's Encrypt
    ssl_certificate /etc/letsencrypt/live/swe.ajklein.io/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/swe.ajklein.io/privkey.pem;
    
    # Proxy /api/* requests to backend on port 3002
    location /api/ {
        proxy_pass http://127.0.0.1:3002;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
    
    # Serve all other requests from symlinked frontend build
    root /usr/share/nginx/SWE/prod_root/;
}
```

This pattern is the same for test and dev, with different ports and symlink paths.
