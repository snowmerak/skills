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
// drizzle.config.ts
import { defineConfig } from 'drizzle-kit';

export default defineConfig({
  schema: './src/schema/index.ts',
  out: './src/drizzle',
  dialect: 'postgresql', // 'mysql' | 'sqlite'
  dbCredentials: {
    url: process.env.DATABASE_URL!,
  },
  verbose: true,
  strict: true,
});
```

```json
// drizzle.config.json
{
  "schema": "./src/schema/index.ts",
  "out": "./src/drizzle",
  "dialect": "postgresql",
  "dbCredentials": {
    "url": "postgresql://user:password@localhost:5432/dbname"
  },
  "verbose": true,
  "strict": true
}
```

### Package.json Scripts

```json
{
  "scripts": {
    "db:generate": "drizzle-kit generate",
    "db:migrate": "drizzle-kit migrate",
    "db:push": "drizzle-kit push",
    "db:pull": "drizzle-kit pull",
    "db:drop": "drizzle-kit drop",
    "db:studio": "drizzle-kit studio",
    "db:validate": "drizzle-kit validate",
    "db:introspect": "drizzle-kit introspect"
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

Generated migration files example:

```typescript
// 0001_initial_migration.sql
CREATE TABLE IF NOT EXISTS "users" (
  "id" serial PRIMARY KEY,
  "name" varchar(255) NOT NULL,
  "email" varchar(255) UNIQUE NOT NULL,
  "created_at" timestamp DEFAULT now() NOT NULL
);

CREATE INDEX IF NOT EXISTS "users_email_idx" ON "users"("email");
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

# Generate TypeScript types from existing database
npx drizzle-kit introspect
```

## Migration Workflow

### Development Workflow

```bash
# 1. Make changes to your schema.ts file
# 2. Generate migration files
npm run db:generate

# 3. Review the generated SQL in ./drizzle/migrations/

# 4. Apply migrations to local database
npm run db:migrate

# 5. Verify changes
npm run db:studio  # Opens Drizzle Studio GUI
```

### Production Deployment Workflow

```bash
# 1. Generate migration files locally
npm run db:generate

# 2. Commit migration files to git
git add ./src/drizzle/migrations/
git commit -m "feat: add user roles table"

# 3. Deploy code (CI/CD pipeline)

# 4. Apply migrations in production
npx drizzle-kit migrate --config=drizzle.config.production.ts
```

### Production Config Example

```typescript
// drizzle.config.production.ts
import { defineConfig } from 'drizzle-kit';

export default defineConfig({
  schema: './src/schema/index.ts',
  out: './src/drizzle',
  dialect: 'postgresql',
  dbCredentials: {
    url: process.env.PRODUCTION_DATABASE_URL!,
  },
  strict: true,
  verbose: true,
});
```

## Migration File Structure

### Directory Structure

```
src/
├── drizzle/
│   ├── migrations/
│   │   ├── _journal.json      # Migration tracking file
│   │   ├── 0001_initial.sql   # First migration
│   │   ├── 0002_add_posts.sql # Second migration
│   │   └── 0003_add_comments.sql
│   └── schema.ts              # Generated schema (optional)
├── schema/
│   ├── index.ts               # Main schema exports
│   ├── users.ts
│   └── posts.ts
```

### Journal File (_journal.json)

```json
{
  "version": "5",
  "dialect": "postgresql",
  "entries": [
    {
      "idx": 0,
      "local": true,
      "forward": ["0001_initial.sql"],
      "backward": []
    },
    {
      "idx": 1,
      "local": true,
      "forward": ["0002_add_posts.sql"],
      "backward": ["0001_initial.sql"]
    }
  ]
}
```

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
-- migrations/0001_create_users.sql
CREATE TABLE users (
  id SERIAL PRIMARY KEY,
  name VARCHAR(255) NOT NULL,
  email VARCHAR(255) UNIQUE NOT NULL,
  created_at TIMESTAMP DEFAULT NOW()
);
```

Then generate TypeScript schema:

```bash
npx drizzle-kit introspect
```

## Multi-Database Support

### PostgreSQL

```typescript
// drizzle.config.ts
export default defineConfig({
  dialect: 'postgresql',
  dbCredentials: {
    url: process.env.POSTGRES_URL!,
  },
});
```

### MySQL

```typescript
export default defineConfig({
  dialect: 'mysql',
  dbCredentials: {
    url: process.env.MYSQL_URL!,
  },
});
```

### SQLite

```typescript
export default defineConfig({
  dialect: 'sqlite',
  schema: './src/schema/index.ts',
  out: './drizzle',
  dbCredentials: {
    url: './dev.db',
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

### Production Studio Config

```typescript
export default defineConfig({
  // ... other config
  studio: {
    port: 5270,
    host: 'localhost',
  },
});
```

## Rollback Strategies

### Manual Rollback

1. Create a new migration file that reverses changes:

```sql
-- migrations/0004_rollback_add_roles.sql
DROP TABLE IF EXISTS user_roles;
ALTER TABLE users DROP COLUMN role;
```

2. Apply the rollback:

```bash
npx drizzle-kit migrate
```

### Using drizzle-kit Drop (Development Only)

```bash
# Drop all tables (DANGEROUS - development only!)
npx drizzle-kit drop

# With confirmation
npx drizzle-kit drop --force
```

## Environment-Specific Configurations

### Development

```typescript
// drizzle.config.dev.ts
export default defineConfig({
  dialect: 'postgresql',
  dbCredentials: { url: process.env.DEV_DATABASE_URL! },
  out: './drizzle/dev',
});
```

### Staging

```typescript
// drizzle.config.staging.ts
export default defineConfig({
  dialect: 'postgresql',
  dbCredentials: { url: process.env.STAGING_DATABASE_URL! },
  out: './drizzle/staging',
});
```

### Production

```typescript
// drizzle.config.prod.ts
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
3. **Version control migrations** - Commit all migration files to git
4. **Test migrations locally first** - Apply migrations in development/staging before production
5. **Use strict mode** - Enable `strict: true` for safer schema changes
6. **Backup before migrations** - Always backup production database before applying migrations
7. **Name migrations descriptively** - Use meaningful names that explain the change
8. **Keep migrations idempotent** - Design migrations to be safe to run multiple times

## Troubleshooting

### Migration Conflicts

```bash
# Check current migration status
npx drizzle-kit validate

# Reset migration tracking (DANGEROUS)
rm -rf ./drizzle/migrations/_journal.json
npx drizzle-kit generate
```

### Schema Drift Detection

```bash
# Detect differences between schema and database
npx drizzle-kit validate

# Generate diff report
npx drizzle-kit diff
```

## References

- [Drizzle Kit Documentation](https://orm.drizzle.team/docs/kit-overview)
- [Migrations Guide](https://orm.drizzle.team/docs/migrate)
- [Schema First Approach](https://orm.drizzle.team/docs/sql-schema-declaration)
