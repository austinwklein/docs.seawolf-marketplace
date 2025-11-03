# Frontend and Backend Build Pipeline

## Build overview

The CI/CD pipeline compiles both frontend (Angular) and backend (Express/Node.js) from source code to runnable artifacts.

```
┌───────────────────────────────────────────────────┐
│                GitHub Repository                  │
│  (TypeScript, Angular, Express source code)       │
└────────────────────────┬──────────────────────────┘
                         ↓                
         (GitHub Actions runner checks out)
                         ↓                  
          ┌──────────────────────────────┐
          │   Frontend Build             │
          │ (Angular CLI: ng build)      │
          │                              │
          │ Input: /frontend/src/        │
          │ Output: dist/browser/        │
          │ Format: Static files         │
          │ (HTML, CSS, JS, assets)      │
          └──────────────┬───────────────┘
                         ↓     
          ┌──────────────────────────────┐
          │   Backend Build              │
          │ (TypeScript: `tsc compile`)  │
          │                              │
          │ Input: /backend/src/         │
          │ Output: *.js files           │
          │ Format: JavaScript           │
          └──────────────┬───────────────┘
                         ↓  
          ┌──────────────────────────────┐
          │   Deploy to VPS              │
          │ - Symlink frontend to Nginx  │
          │ - Restart PM2 backend        │
          │ - Go live                    │
          └──────────────────────────────┘
```

## Frontend build pipeline

### What is compiled

**Input**: TypeScript, Angular templates, SCSS, images

```
/frontend/src/
├── app/
│   ├── components/
│   ├── services/
│   ├── pages/
│   └── *.ts files
├── assets/
│   ├── images/
│   ├── styles/
│   └── icons/
├── index.html
└── main.ts
```

**Output**: Static HTML, CSS, JavaScript files

```
/frontend/dist/frontend/browser/
├── index.html
├── main.*.js
├── polyfills.*.js
├── styles.*.css
├── assets/
└── 404.html
```

### Build command

```bash
cd /opt/seawolf-marketplace/prod/frontend
npm ci
ng build --configuration=production
```

**Step 1: Install dependencies**

```bash
npm ci
```

- `ci` = "clean install" (more reliable than `npm install` in CI)
- Reads `package.json` and `package-lock.json`
- Installs exact versions specified in lock file
- Faster than `npm install` due to caching

**Step 2: Compile with Angular CLI**

```bash
ng build --configuration=production
```

**For production**:
- `-configuration=production` flag enables optimizations:
  - Ahead-of-Time (AOT) compilation
  - Minification (removes whitespace, shortens variable names)
  - Tree-shaking (removes unused code)
  - Budgets enforcement (warns if bundle too large)
  - Source maps for debugging

**For test/dev**:
- `ng build` (default development build)
- Faster compilation
- Unminified for easier debugging
- Full source maps

### Build output files

Angular build produces multiple JavaScript bundles to optimize loading:

| File             | Purpose                                      |
|------------------|----------------------------------------------|
| `main.*.js`      | Application code                             |
| `polyfills.*.js` | Browser compatibility (IE11, older browsers) |
| `styles.*.css`   | Global styles                                |
| `runtime.*.js`   | Angular runtime                              |
| `vendor.*.js`    | Third-party libraries                        |

**Versioning**: Hash in filename (e.g., `main.abc123.js`) for cache busting. When code changes, hash changes, forcing browsers to reload.

### Frontend build time

- **Typical**: 30-60 seconds
- **Depends on**:
  - npm cache freshness
  - number of components
  - complexity of templates
  - network speed to npm registry

### Common frontend build errors

| Error                                          | Cause                        | Solution                                                 |
|------------------------------------------------|------------------------------|----------------------------------------------------------|
| `npm ERR! 404 Not Found`                       | Package not in npm registry  | Check package name spelling in `package.json`            |
| `ERROR in src/app/...ts`                       | TypeScript compilation error | Fix TypeScript errors; check variable names, types       |
| `ERROR: The Angular Compiler is not installed` | Angular CLI not installed    | Run `npm ci` to install dependencies                     |
| `Exceeds size budget`                          | Bundle too large             | Remove unused dependencies or optimize component imports |
| `Permission denied`                            | npm cache corrupted          | Run `npm cache verify` or `npm cache clean --force`      |

## Backend build pipeline

### What is compiled

**Input**: TypeScript backend code

```
/backend/src/
├── routes/
│   ├── listings.ts
│   ├── users.ts
│   └── *.ts
├── middleware/
│   ├── auth.ts
│   └── *.ts
├── db/
│   ├── connection.ts
│   └── *.ts
└── index.ts (main entry point)
```

**Output**: JavaScript files in `dist/` or alongside sources

```
/backend/
├── dist/ (or .js files alongside .ts)
│   ├── index.js
│   ├── routes/
│   ├── middleware/
│   ├── db/
│   └── *.js
├── src/ (original TypeScript)
└── tsconfig.json
```

### Build command

```bash
cd /opt/seawolf-marketplace/prod/backend
npm ci
npx tsc
```

**Step 1: Install dependencies**

```bash
npm ci
```

Same as frontend: clean install of dependencies.

**Step 2: Compile TypeScript**

```bash
npx tsc
```

- `tsc` = TypeScript compiler
- Reads `tsconfig.json` for compilation settings
- Transpiles `.ts` files to `.js`
- Type-checks code (catches type errors before runtime)
- Output location specified in `tsconfig.json` (`outDir`)

### TypeScript configuration (tsconfig.json)

Example settings:

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "commonjs",
    "lib": ["ES2020"],
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

**Key settings**:
- `target`: Compile to ES2020 JavaScript (modern Node.js)
- `module`: Use CommonJS modules (Node.js standard)
- `outDir`: Output JavaScript to `dist/` folder
- `strict`: Enable strict type checking
- `esModuleInterop`: Better compatibility with CommonJS modules

### Backend build time

- **Typical**: 10-20 seconds
- **Depends on**:
  - npm cache freshness
  - number of TypeScript files
  - complexity of code
  - network speed

### Running the compiled backend

After compilation, PM2 runs the output JavaScript:

```bash
pm2 start /opt/seawolf-marketplace/prod/backend/index.js \
  --name backend-prod \
  --cwd /opt/seawolf-marketplace/prod/backend
```

The Node.js runtime executes the compiled `.js` files. TypeScript is not needed at runtime (only development).

### Common backend build errors

| Error                                                                           | Cause                             | Solution                                             |
|---------------------------------------------------------------------------------|-----------------------------------|------------------------------------------------------|
| `error TS2307: Cannot find module`                                              | Missing import or wrong path      | Check import statement; verify file exists           |
| `error TS2345: Argument of type 'X' is not assignable to parameter of type 'Y'` | Type mismatch                     | Fix variable type or function signature              |
| `error TS2304: Cannot find name 'X'`                                            | Undefined variable or typo        | Check spelling; import if from another file          |
| `npm ERR! 404 Not Found`                                                        | Dependency not in npm registry    | Fix package name in `package.json`                   |
| `Permission denied`                                                             | Cannot write to `dist/` directory | Check file permissions; ensure deploy user can write |

## Database initialization

During deployment, SQL scripts initialize the database schema:

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

### Required SQL files

Place these in `/database/sql_scripts/` in your repository:

#### 1. create_database.sql

```sql
-- Create the database (idempotent: checks if exists first)
SELECT 'CREATE DATABASE seawolf_' || :'ENV'
WHERE NOT EXISTS (SELECT FROM pg_database WHERE datname = 'seawolf_' || :'ENV')
\gexec
```

Or for a specific environment:

```sql
CREATE DATABASE IF NOT EXISTS seawolf_prod;
```

#### 2. create_tables.sql

```sql
CREATE TABLE IF NOT EXISTS users (
  userID BIGSERIAL PRIMARY KEY,
  name VARCHAR(100) NOT NULL,
  email VARCHAR(100) NOT NULL,
  -- ... other columns
);

CREATE TABLE IF NOT EXISTS listings (
  listingID BIGSERIAL PRIMARY KEY,
  userID BIGINT NOT NULL REFERENCES users(userID),
  title VARCHAR(150) NOT NULL,
  -- ... other columns
);

-- ... more tables for history, watchlist, etc.
```

#### 3. create_functions.sql

```sql
-- Stored procedures or functions (if any)
CREATE OR REPLACE FUNCTION update_timestamp()
RETURNS TRIGGER AS $$
BEGIN
  NEW.updatedAt = NOW();
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

#### 4. create_indexes.sql

```sql
-- Indexes for query performance
CREATE INDEX IF NOT EXISTS idx_listings_user ON listings(userID);
CREATE INDEX IF NOT EXISTS idx_history_user ON history(userID);
CREATE INDEX IF NOT EXISTS idx_watchlist_user ON watchlist(userID);
```

#### 5. create_mock_data.sql

```sql
-- Test data for development/testing
INSERT INTO users (name, email) VALUES ('Test User', 'test@alaska.edu');
INSERT INTO listings (userID, title, price) VALUES (1, 'Test Book', 25.00);
```

### Idempotency requirement

All SQL scripts must be **idempotent** (safe to run multiple times):

```sql
-- Bad (fails on second run):
CREATE TABLE users (...);

-- Good (safe on multiple runs):
CREATE TABLE IF NOT EXISTS users (...);
```

This allows redeployments without a manual database cleanup.

## Build artifacts

After builds are complete, artifacts are ready for deployment:

### Frontend artifacts

```
/opt/seawolf-marketplace/prod/frontend/dist/frontend/browser/
├── index.html            (entry point)
├── main.abc123.js        (application code)
├── polyfills.def456.js   (browser compatibility)
├── styles.ghi789.css     (styles)
└── assets/               (images, fonts, etc.)
```

These are served by Nginx.

### Backend artifacts

```
/opt/seawolf-marketplace/prod/backend/dist/
├── index.js              (entry point)
├── routes/
│   ├── listings.js
│   ├── users.js
│   └── ...
├── middleware/
│   ├── auth.js
│   └── ...
└── db/
    ├── connection.js
    └── ...
```

These are executed by Node.js via PM2.

## Deployment integration

Build artifacts are used immediately after creation:

```
Build Frontend ──→ Output to dist/
                        ↓
Build Backend  ──→ Output to dist/
                        ↓
Update Nginx symlink ──→ Serve new frontend
       ↓
Restart PM2 ──→ Run new backend code
```

No manual step needed; deployment script handles it all automatically.
