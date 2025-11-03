# Nginx Web Server Configuration

## Overview

Nginx serves two critical roles:

1. **Frontend server**: Serves static Angular files (HTML, CSS, JS)
2. **Reverse proxy**: Forwards `/api/*` requests to Node.js backend

Three virtual hosts run simultaneously, one per environment.

## Nginx directory structure

```
/etc/nginx/
├── nginx.conf                          (main configuration file)
└── sites-enabled.d/                    (Where Nginx loads configs from: Symlinks to available configs)
    ├── 05-tls.conf                     (SSL/TLS settings)
    ├── 10-docs_swe_ajklein_io.conf
    ├── 11-test_swe_ajklein_io.conf
    ├── 12-dev_swe_ajklein_io.conf
    └── 20-swe_ajklein_io.conf
```

The numeric prefixes control load order (lower numbers load first).

## Main nginx.conf

```nginx
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log notice;
pid /run/nginx.pid;

events {
    worker_connections 1024;
}

http {
    # Logging format
    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for"';
    
    access_log /var/log/nginx/access.log main;
    
    # Performance settings
    sendfile on;
    tcp_nopush on;
    keepalive_timeout 65;
    types_hash_max_size 4096;
    
    # MIME types
    include /etc/nginx/mime.types;
    default_type application/octet-stream;
    
    # Load environment-specific configs
    include /etc/nginx/sites-enabled.d/*.conf;
}
```

**Key settings**:
- `worker_processes auto`: Use as many worker processes as CPU cores
- `worker_connections 1024`: Each worker can handle 1024 concurrent connections
- `types_hash_max_size 4096`: Increase if adding many MIME types
- `include /etc/nginx/sites-enabled.d/*.conf`: Load all virtual host configs

## SSL/TLS configuration (05-tls.conf)

Shared SSL settings applied to all virtual hosts:

```nginx
ssl_certificate /etc/letsencrypt/live/swe.ajklein.io/fullchain.pem;
ssl_certificate_key /etc/letsencrypt/live/swe.ajklein.io/privkey.pem;

ssl_session_cache shared:SSL:1m;
ssl_session_timeout 10m;
ssl_ciphers PROFILE=SYSTEM;
ssl_prefer_server_ciphers on;
```

**Certificate details**:
- **Provider**: Let's Encrypt
- **Domain**: swe.ajklein.io (wildcard for *.swe.ajklein.io)
- **Renewal**: Automated via certbot (renewed before expiration)
- **Location**: `/etc/letsencrypt/live/swe.ajklein.io/`

## Production virtual host (20-swe_ajklein_io.conf)

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name swe.ajklein.io;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    listen [::]:443 ssl;
    server_name swe.ajklein.io;
    
    ssl_certificate /etc/letsencrypt/live/swe.ajklein.io/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/swe.ajklein.io/privkey.pem;
    
    # Proxy API requests to Node.js backend on port 3002
    location /api/ {
        proxy_pass http://127.0.0.1:3002;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
    
    # Serve frontend static files
    root /usr/share/nginx/SWE/prod_root/;
}
```

**Configuration breakdown**:

### HTTP to HTTPS redirect (lines 1-5)

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name swe.ajklein.io;
    return 301 https://$host$request_uri;
}
```

- Listens on HTTP port 80
- Returns HTTP 301 (permanent redirect)
- Redirects to HTTPS with same path

### HTTPS server (lines 7-27)

```nginx
server {
    listen 443 ssl;
    listen [::]:443 ssl;
    server_name swe.ajklein.io;
    ssl_certificate ...;
    ssl_certificate_key ...;
}
```

- Listens on HTTPS port 443
- Specifies SSL certificate and key
- Handles all HTTPS requests to swe.ajklein.io

### API proxy (lines 16-24)

```nginx
location /api/ {
    proxy_pass http://127.0.0.1:3002;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection 'upgrade';
    proxy_set_header Host $host;
    proxy_cache_bypass $http_upgrade;
}
```

**Purpose**: Forward API requests to Node.js backend

**Key headers**:
- `Upgrade`: Supports WebSocket upgrades (if needed in future)
- `Connection: upgrade`: Required for WebSocket
- `Host`: Passes original hostname to backend (important for virtual hosts)
- `cache_bypass`: Bypass Nginx cache for dynamic API responses

**Result**: Request to `https://swe.ajklein.io/api/listings` forwards to `http://127.0.0.1:3002/api/listings`

### Frontend files (line 26)

```nginx
root /usr/share/nginx/SWE/prod_root/;
```

**Purpose**: Serve all non-API requests from the symlinked directory

When a request comes in (e.g., `/index.html`), Nginx:
1. Checks if `/api/` prefix (if yes, proxy to backend)
2. Otherwise, serves from `/usr/share/nginx/SWE/prod_root/index.html`

## Test/Dev virtual hosts

Test, Dev, and Prod follow the same pattern, with different ports and root directories:

| Environment | Port | Root Directory |
|---|---|---|
| Test | 3000 | `/usr/share/nginx/SWE/test_root/` |
| Dev | 3001 | `/usr/share/nginx/SWE/dev_root/` |
| Prod | 3002 | `/usr/share/nginx/SWE/prod_root/` |

Example for test:

```nginx
server {
    listen 443 ssl;
    server_name test.swe.ajklein.io;
    
    ssl_certificate /etc/letsencrypt/live/swe.ajklein.io/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/swe.ajklein.io/privkey.pem;
    
    location /api/ {
        proxy_pass http://127.0.0.1:3000;  # {3000|3001|3002}
        # ... proxy headers ...
    }
    
    root /usr/share/nginx/SWE/test_root/;
}
```

## How requests flow through Nginx

### Frontend request example

**User navigates to** `https://swe.ajklein.io/listings`

1. Browser sends HTTPS request
2. Nginx receives on port 443
3. Matches server block for `swe.ajklein.io`
4. Request path is `/listings`
5. Nginx serves from root directory: `/usr/share/nginx/SWE/prod_root/`
6. Browser receives `index.html` (Angular's router handles the path client-side)

### API request example

**Frontend JavaScript calls** `fetch('/api/listings')`

1. Browser sends HTTPS request to `https://swe.ajklein.io/api/listings`
2. Nginx receives on port 443
3. Matches server block for `swe.ajklein.io`
4. Request path is `/api/listings` (matches `location /api/` block)
5. Nginx proxies request to `http://127.0.0.1:3002/api/listings`
6. Node.js backend responds
7. Nginx forwards response back to browser

## Reloading Nginx after config changes

```bash
# Check syntax
sudo nginx -t

# Reload config without restarting (only if nginx -t passes)
sudo systemctl reload nginx

# Or restart if needed
sudo systemctl restart nginx
```

Always run `nginx -t` before reloading to catch syntax errors.

## Viewing Nginx logs

```bash
# Access log
tail -f /var/log/nginx/access.log

# Error log
tail -f /var/log/nginx/error.log

# Last 100 lines of error log
tail -n 100 /var/log/nginx/error.log
```

## Common Nginx issues

| Issue | Symptoms | Solution |
|-------|----------|----------|
| API requests fail | Frontend loads but "Cannot GET /api/..." errors | Check PM2 backend process is running on correct port; verify proxy_pass port |
| Frontend returns 404 | Accessing specific routes returns 404 | Angular SPA issue; ensure `index.html` is served for all non-API routes |
| SSL certificate error | Browser shows untrusted certificate warning | Run `sudo certbot renew` to renew Let's Encrypt certificate |
| Nginx won't start | `nginx: command not found` or permission denied | Check if nginx is installed: `nginx -v`; ensure running as root or sudo |
| Port 443 already in use | "Address already in use" error | Check: `lsof -i :443`; kill conflicting process if needed |
| Symlink points to wrong directory | Frontend files not updating after deployment | Verify symlink target: `ls -la /usr/share/nginx/SWE/`; check build output directory |

## Symlink management

During deployment, Nginx symlinks are updated to point to the latest build:

```bash
ln -sfn "$FRONTEND_DIR/dist/frontend/browser" "$WEB_ROOT"
```

**Verify symlink is correct**:

```bash
# Check what the symlink points to
ls -la /usr/share/nginx/SWE/

# Output example:
# test_root -> /opt/seawolf-marketplace/test/frontend/dist/frontend/browser
# dev_root -> /opt/seawolf-marketplace/dev/frontend/dist/frontend/browser
# prod_root -> /opt/seawolf-marketplace/prod/frontend/dist/frontend/browser
```

If a symlink points to the wrong directory, manually update it for testing and then add to scripts for deployment:

```bash
ln -sfn /opt/seawolf-marketplace/prod/frontend/dist/frontend/browser /usr/share/nginx/SWE/prod_root
```

Then verify with `ls -la` again.

## Performance tuning

For the scope of this project, the current Nginx configuration is sufficient. Future optimizations:

- **Gzip compression**: Reduce file sizes for faster downloads
- **Browser caching**: Set cache headers for static files
- **Load balancing**: Distribute traffic across multiple backend servers
- **Rate limiting**: Prevent abuse and malicious requests

These can be added to Nginx config as needed.
