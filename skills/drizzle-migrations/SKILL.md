---
name: drizzle-migrations
description: Drizzle ORM migrations using drizzle-kit including schema generation, migration creation, applying migrations, rollback strategies, and multi-environment deployment workflows. Use when managing database schema changes, creating or applying migrations, or handling production deployments in Drizzle ORM.
license: MIT
metadata:
  author: snowmerak
  version: "1.0"
  framework: drizzle-orm
  category: migrations
---

# Drizzle Migrations Skills

This skill covers comprehensive migration management using drizzle-kit for database schema changes and deployments.

## Installation and Setup

### Install drizzle-kit

```bash
npm install -D drizzle-kit
```

### Initialize Configuration

```bash
npx drizzle-kit init
```

This creates `drizzle.config.ts` or `drizzle.config.json`:

```typescript
// drizzle.config.ts - Basic configuration
import { defineConfig } from 'drizzle-kit';

export default defineConfig({
  dialect: 'postgresql', // 'mysql' | 'sqlite' | 'turso' | 'singlestore' | 'mssql' | 'cockroachdb'
  schema: './src/schema/index.ts',
  out: './drizzle',
});
```

### drizzle.config.ts - Full Configuration (v0.25+)

```typescript
import { defineConfig } from 'drizzle-kit';

export default defineConfig({
  // Database dialect: 'postgresql' | 'mysql' | 'sqlite' | 'turso' | 'singlestore' | 'mssql' | 'cockroachdb'
  dialect: 'postgresql',
  
  // Schema file(s) - supports glob patterns
  schema: './src/schema/index.ts',
  // Or multiple files:
  // schema: ['./src/schema/users.ts', './src/schema/posts.ts'],
  // Or glob pattern:
  // schema: './src/schema/**/*.ts',
  
  // Output directory for migration files (default: 'drizzle')
  out: './drizzle',
  
  // Driver for special database vendors (optional)
  // 'aws-data-api' | 'd1-http' | 'pglite'
  driver: 'pglite',
  
  // Database connection credentials
  dbCredentials: {
    url: process.env.DATABASE_URL!,
    // Or for AWS Data API:
    // database: 'database',
    // resourceArn: 'arn:aws:rds...',
    // secretArn: 'arn:aws:secretsmanager...',
  },
  
  // Migration-specific settings
  migrations: {
    prefix: '0000',           // Prefix for migration folders (default: timestamp)
    table: '__drizzle_migrations__',  // Migration tracking table name
    schema: 'public',         // Schema for migration table
  },
  
  // Introspection settings
  introspect: {
    casing: 'camel',          // 'camel' | 'pascal' - controls generated property names
  },
  
  // Filter settings
  tablesFilter: '*',          // Table name filter (glob pattern)
  schemaFilter: 'public',     // Schema name filter
  extensionsFilters: ['postgis'], // Extension filters
  
  // Entity settings for selective generation
  entities: {
    roles: {
      provider: '',           // Provider name
      exclude: [],            // Exclude tables
      include: [],            // Include only these tables
    },
  },
  
  // Debug options
  breakpoints: true,          // Generate breakpoint comments in SQL
  strict: true,               // Strict mode for schema validation
  verbose: true,              // Verbose output
});
```

### Package.json Scripts

```json
{
  "scripts": {
    "db:generate": "drizzle-kit generate",
    "db:migrate": "drizzle-kit migrate",
    "db:push": "drizzle-kit push",
    "db:pull": "drizzle-kit pull",
    "db:check": "drizzle-kit check",
    "db:up": "drizzle-kit up",
    "db:studio": "drizzle-kit studio",
    "db:export": "drizzle-kit export"
  }
}
```

## Migration Commands

### Generate Migrations

```bash
# Generate migration files (does not apply to database)
npx drizzle-kit generate

# With custom config
npx drizzle-kit generate --config=drizzle.config.ts

# Specify output directory
npx drizzle-kit generate --out=./migrations
```

### Apply Migrations

```bash
# Apply pending migrations to database
npx drizzle-kit migrate

# With verbose output
npx drizzle-kit migrate --verbose

# Dry run (preview without applying)
npx drizzle-kit migrate --dry-run
```

### Push Schema (No Migration Files)

```bash
# Sync schema directly to database (no migration files created)
npx drizzle-kit push

# With force flag (applies changes even with data loss warnings)
npx drizzle-kit push --force
```

> **Warning**: `push` is for development only. Always use migrations in production.

### Pull Schema from Database

```bash
# Introspect existing database and generate schema files
npx drizzle-kit pull

# Generate TypeScript types from existing database (same as pull)
npx drizzle-kit pull
```

### Check Migration Status

```bash
# Check current migration status and pending changes
npx drizzle-kit check

# This compares your schema with the database and reports differences
```

### Upgrade to Latest Migrations

```bash
# Apply all pending migrations (same as migrate but more explicit)
npx drizzle-kit up

# Useful in CI/CD pipelines to ensure database is up-to-date
```

### Export Schema

```bash
# Export schema as SQL or TypeScript
npx drizzle-kit export

# Exports the current database schema to a file
```

## Migration File Structure (v0.25+)

### Directory Structure

Drizzle Kit v0.25+ uses a **folder-based** migration structure:

```
src/
├── drizzle/
│   ├── 202409125510_premium_mister_fear/    # Migration folder (timestamp + name)
│   │   └── migration.sql                      # SQL file inside the folder
│   ├── 202409130000_add_posts/               # Another migration folder
│   │   └── migration.sql
│   ├── user.ts                               # Schema snapshot file
│   └── post.ts                               # Schema snapshot file
├── schema/
│   ├── index.ts                              # Main schema exports
│   ├── users.ts
│   └── posts.ts
```

### Journal File (_journal.json)

The journal file tracks migration state:

```json
{
  "version": "7",
  "dialect": "postgresql",
  "entries": [
    {
      "idx": 0,
      "local": true,
      "forward": ["202409125510_premium_mister_fear"],
      "backward": []
    },
    {
      "idx": 1,
      "local": true,
      "forward": ["202409130000_add_posts"],
      "backward": ["202409125510_premium_mister_fear"]
    }
  ]
}
```

> **Note**: Migration entries now reference **folder names** instead of individual SQL files.

## Schema-First vs Migration-First

### Schema-First (Recommended)

Define schema in TypeScript, generate migrations:

```typescript
// schema/users.ts
import { pgTable, serial, varchar, timestamp } from 'drizzle-orm/pg-core';

export const users = pgTable('users', {
  id: serial('id').primaryKey(),
  name: varchar('name', { length: 255 }).notNull(),
  email: varchar('email', { length: 255 }).unique().notNull(),
  createdAt: timestamp('created_at').defaultNow().notNull(),
});

// Generate migration
npx drizzle-kit generate
```

### Migration-First (SQL)

Write SQL migrations directly, then introspect:

```sql
-- drizzle/202409125510_create_users/migration.sql
CREATE TABLE users (
  id SERIAL PRIMARY KEY,
  name VARCHAR(255) NOT NULL,
  email VARCHAR(255) UNIQUE NOT NULL,
  created_at TIMESTAMP DEFAULT NOW()
);
```

Then generate TypeScript schema:

```bash
npx drizzle-kit pull
```

## Multi-Database Support

### PostgreSQL

```typescript
export default defineConfig({
  dialect: 'postgresql',
  dbCredentials: { url: process.env.POSTGRES_URL! },
});
```

### MySQL

```typescript
export default defineConfig({
  dialect: 'mysql',
  dbCredentials: { url: process.env.MYSQL_URL! },
});
```

### SQLite

```typescript
export default defineConfig({
  dialect: 'sqlite',
  schema: './src/schema/index.ts',
  out: './drizzle',
  dbCredentials: { url: './dev.db' },
});
```

### AWS Data API (PostgreSQL)

```typescript
export default defineConfig({
  dialect: 'postgresql',
  driver: 'aws-data-api',
  schema: './src/schema.ts',
  dbCredentials: {
    database: 'mydb',
    resourceArn: 'arn:aws:rds:...',
    secretArn: 'arn:aws:secretsmanager:...',
  },
});
```

### Cloudflare D1 (SQLite)

```typescript
export default defineConfig({
  dialect: 'sqlite',
  driver: 'd1-http',
  schema: './src/schema.ts',
  dbCredentials: {
    databaseId: 'your-d1-database-id',
    token: 'your-cloudflare-token',
  },
});
```

## Drizzle Studio (GUI)

### Start Drizzle Studio

```bash
npx drizzle-kit studio
# Opens at http://localhost:5270
```

### Features

- Browse tables and data
- Run SQL queries
- Visualize schema relationships
- Export/import data

## Rollback Strategies

### Manual Rollback (Recommended)

1. Create a new migration file that reverses changes:

```sql
-- drizzle/20240915_rollback_add_roles/migration.sql
DROP TABLE IF EXISTS user_roles;
ALTER TABLE users DROP COLUMN role;
```

2. Apply the rollback:

```bash
npx drizzle-kit migrate
```

### Drop All Tables (Development Only)

```bash
# Drop all tables tracked by drizzle-kit (DANGEROUS!)
npx drizzle-kit drop

# With confirmation prompt
npx drizzle-kit drop --force
```

## Multiple Configuration Files

You can have multiple config files for different environments:

```bash
# Development
npx drizzle-kit generate --config=drizzle.dev.config.ts

# Production
npx drizzle-kit generate --config=drizzle.prod.config.ts
```

Project structure:

```
📦 <project root>
├── 📂 drizzle/
├── 📂 src/
├── 📜 .env
├── 📜 drizzle.dev.config.ts
├── 📜 drizzle.prod.config.ts
└── 📜 package.json
```

## Environment-Specific Configurations

### Development

```typescript
// drizzle.dev.config.ts
export default defineConfig({
  dialect: 'postgresql',
  dbCredentials: { url: process.env.DEV_DATABASE_URL! },
  out: './drizzle/dev',
});
```

### Staging

```typescript
// drizzle.staging.config.ts
export default defineConfig({
  dialect: 'postgresql',
  dbCredentials: { url: process.env.STAGING_DATABASE_URL! },
  out: './drizzle/staging',
});
```

### Production

```typescript
// drizzle.prod.config.ts
export default defineConfig({
  dialect: 'postgresql',
  dbCredentials: { url: process.env.PROD_DATABASE_URL! },
  out: './drizzle/prod',
  strict: true,
  verbose: true,
});
```

## CI/CD Integration

### GitHub Actions Example

```yaml
name: Database Migrations

on:
  push:
    branches: [main]

jobs:
  migrate:
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_DB: testdb
          POSTGRES_USER: testuser
          POSTGRES_PASSWORD: testpass
        ports:
          - 5432:5432
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
      
      - run: npm ci
      
      - name: Run migrations
        env:
          DATABASE_URL: postgresql://testuser:testpass@localhost:5432/testdb
        run: npx drizzle-kit migrate
```

### Docker Integration

```dockerfile
# Dockerfile
FROM node:20-alpine

WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .

# Run migrations on startup
CMD ["sh", "-c", "npx drizzle-kit migrate && node dist/main.js"]
```

## Best Practices

1. **Always use migrations in production** - Never use `push` for production databases
2. **Review generated SQL** - Always check migration files before applying
3. **Version control migrations** - Commit all migration folders to git
4. **Test migrations locally first** - Apply migrations in development/staging before production
5. **Use strict mode** - Enable `strict: true` for safer schema changes
6. **Backup before migrations** - Always backup production database before applying migrations
7. **Use descriptive folder names** - Migration folders should explain the change (e.g., `20240915_add_user_roles`)
8. **Use multiple config files** - Separate dev/staging/prod configurations for safety

## Troubleshooting

### Migration Conflicts

```bash
# Check current migration status and pending changes
npx drizzle-kit check

# Compare schema with database
npx drizzle-kit generate --dry-run
```

### Schema Drift Detection

```bash
# Detect differences between schema and database
npx drizzle-kit check

# Generate migration to see what would change
npx drizzle-kit generate
```

### Reset Migration Tracking (DANGEROUS)

```bash
# Remove journal and regenerate (loses migration history!)
rm -rf ./drizzle/_journal.json
npx drizzle-kit generate
```

## References

- [Drizzle Kit Documentation](https://orm.drizzle.team/docs/kit-overview)
- [drizzle.config.ts Reference](https://orm.drizzle.team/docs/drizzle-config-file)
- [Migrations Guide](https://orm.drizzle.team/docs/migrate)
- [Schema First Approach](https://orm.drizzle.team/docs/sql-schema-declaration)
