---
name: drizzle-core
description: Drizzle ORM core concepts including architecture, supported databases, TypeScript integration, and basic setup. Use when learning Drizzle ORM fundamentals, choosing database drivers, or setting up initial configuration.
license: MIT
metadata:
  author: snowmerak
  version: "1.0"
  framework: drizzle-orm
  category: core
---

# Drizzle ORM Core Skills

This skill covers the fundamental concepts of Drizzle ORM for building type-safe SQL applications.

## Overview

Drizzle ORM is a TypeScript-first, lightweight, and fast Object Relational Mapper (ORM) that provides complete type safety with minimal runtime overhead. It uses a code-first approach where database schemas are defined in TypeScript code.

### Key Features

- **Type-safe**: Full TypeScript support with automatic type inference
- **Lightweight**: Minimal runtime footprint compared to other ORMs
- **Fast**: Direct SQL generation without heavy abstraction layers
- **Driver agnostic**: Works with multiple database drivers
- **Code-first**: Define schemas in TypeScript, not SQL files

## Architecture

```
drizzle-orm (Core ORM Library)
├── drizzle-orm/pg-core      # PostgreSQL column types
├── drizzle-orm/mysql2       # MySQL column types  
├── drizzle-orm/libsql       # SQLite column types
└── drizzle-orm/node-postgres # Driver adapter

drizzle-kit (Migration CLI)
└── drizzle-kit              # Command-line tool for migrations
```

## Supported Databases

| Database | Driver | Package |
|----------|--------|---------|
| PostgreSQL | node-postgres | `drizzle-orm/node-postgres` |
| PostgreSQL | postgres.js | `drizzle-orm/postgres-js` |
| MySQL | mysql2 | `drizzle-orm/mysql2` |
| SQLite | libsql | `drizzle-orm/libsql` |
| SQLite | better-sqlite3 | `drizzle-orm/better-sqlite3` |
| Turso | @libsql/client | `drizzle-orm/libsql` |
| Cloudflare D1 | d1-http | `drizzle-orm/d1` |
| CockroachDB | node-postgres | `drizzle-orm/node-postgres` |
| MSSQL | mssql | `drizzle-orm/mssql` |

## Installation

### Basic Setup

```bash
# Install core library and database driver
npm install drizzle-orm pg  # PostgreSQL example
npm install -D drizzle-kit   # Migration tool (dev dependency)
```

### TypeScript Configuration

```json
{
  "compilerOptions": {
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "esModuleInterop": true
  }
}
```

## Basic Setup Example

### PostgreSQL with node-postgres

```typescript
import { drizzle } from 'drizzle-orm/node-postgres';
import { Pool } from 'pg';

// Create connection pool
const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
});

// Initialize Drizzle ORM
const db = drizzle(pool);

// Use the database instance
const users = await db.select().from(usersTable);
```

### MySQL with mysql2

```typescript
import { drizzle } from 'drizzle-orm/mysql2';
import { createPool } from 'mysql2/promise';

const pool = createPool({
  host: process.env.DB_HOST,
  user: process.env.DB_USER,
  password: process.env.DB_PASSWORD,
  database: process.env.DB_NAME,
});

const db = drizzle(pool);
```

### SQLite with libsql

```typescript
import { drizzle } from 'drizzle-orm/libsql';
import { createClient } from '@libsql/client';

const client = createClient({
  url: "file:local.db",
});

const db = drizzle(client);
```

## Core Concepts

### Schema Definition

Drizzle uses a code-first approach where you define your database schema in TypeScript:

```typescript
import { pgTable, serial, varchar, text, timestamp } from 'drizzle-orm/pg-core';

export const users = pgTable('users', {
  id: serial('id').primaryKey(),
  name: varchar('name', { length: 255 }).notNull(),
  email: varchar('email', { length: 255 }).unique().notNull(),
  createdAt: timestamp('created_at').defaultNow().notNull(),
});
```

### Query Builder

Drizzle provides a type-safe query builder that generates SQL queries:

```typescript
import { eq, and, or } from 'drizzle-orm';

// Select all users
const allUsers = await db.select().from(users);

// Filter by condition
const activeUsers = await db.select()
  .from(users)
  .where(and(
    eq(users.isActive, true),
    gt(users.createdAt, new Date('2024-01-01'))
  ));
```

### Type Inference

Drizzle automatically infers TypeScript types from your schema:

```typescript
import type { InferModel } from 'drizzle-orm';

// Automatically inferred types
type User = InferModel<typeof users>; // Full user object type
type UserInsert = InferModel<typeof users, 'insert'>; // Insert-only type
type UserSelect = InferModel<typeof users, 'select'>; // Select-only type
```

## Best Practices

1. **Use strict TypeScript**: Enable `strict` and `noUncheckedIndexedAccess` in tsconfig.json
2. **Define schemas separately**: Keep schema definitions in dedicated files
3. **Use migrations**: Always use drizzle-kit for database changes
4. **Type safety**: Leverage TypeScript types throughout your application
5. **Connection pooling**: Use connection pools for production databases

## References

- [Drizzle ORM Documentation](https://orm.drizzle.team)
- [GitHub Repository](https://github.com/drizzle-team/drizzle-orm)
- [Drizzle Kit Documentation](https://orm.drizzle.team/docs/kit-overview)
