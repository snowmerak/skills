---
name: drizzle-schema
description: Drizzle ORM schema definition including table creation, column types, constraints, indexes, and relationships. Use when defining database schemas, creating tables, or setting up data models in Drizzle ORM.
license: MIT
metadata:
  author: snowmerak
  version: "1.0"
  framework: drizzle-orm
  category: schema
---

# Drizzle Schema Definition Skills

This skill covers comprehensive schema definition patterns in Drizzle ORM including tables, columns, constraints, and indexes.

## Table Definitions

### PostgreSQL Tables

```typescript
import { 
  pgTable, 
  serial, 
  varchar, 
  text, 
  timestamp, 
  boolean, 
  integer,
  uuid,
  jsonb,
  date,
  time,
  interval,
  numeric,
  real,
  doublePrecision,
  smallint,
  bigint,
  char,
  cidr,
  inet,
  macaddr,
  money,
  oid,
  tsvector,
  timestamptz,
  interval as pgInterval,
  bytea,                          // Binary data (files, images)
  bigserial,                      // 8-byte auto-increment
  smallserial,                    // 2-byte auto-increment
} from 'drizzle-orm/pg-core';

export const users = pgTable('users', {
  // Primary keys
  id: serial('id').primaryKey(),           // AUTO_INCREMENT equivalent (4-byte)
  uuidId: uuid('uuid_id').defaultRandom().primaryKey(),
  
  // String types
  name: varchar('name', { length: 255 }).notNull(),
  email: varchar('email', { length: 255 }).unique().notNull(),
  slug: char('slug', { length: 100 }),     // Fixed-length string
  
  // Text types
  bio: text('bio'),                         // Unlimited length
  content: text('content').default(''),    // With default value
  
  // Text with enum for type inference (runtime check NOT performed)
  role: text('role', { enum: ['admin', 'user', 'moderator'] }),
  
  // Numeric types
  age: integer('age'),                      // 4 bytes (-2B to +2B)
  smallAge: smallint('small_age'),          // 2 bytes (-32K to +32K)
  bigAge: bigint('big_age'),                // 8 bytes (huge range)
  price: numeric('price', { precision: 10, scale: 2 }), // Decimal
  
  // BigInt with mode option for large auto-increment IDs
  bigId: bigserial('big_id', { mode: 'number' }), // Returns number instead of bigint
  
  // Small serial (2-byte auto-increment)
  smallId: smallserial('small_id'),
  
  // Boolean types
  isActive: boolean('is_active').default(true).notNull(),
  
  // Timestamps
  createdAt: timestamp('created_at')
    .defaultNow()
    .notNull(),
  updatedAt: timestamp('updated_at')
    .$onUpdate(() => new Date())
    .notNull(),
  publishedAt: timestamptz('published_at'), // With timezone
  
  // Binary data (e.g., files, images)
  avatar: bytea('avatar'),
  
  // Other types
  metadata: jsonb('metadata').$type<Record<string, unknown>>(),
  tags: text('tags').array(),               // Array type
});
```

### MySQL Tables

```typescript
import { 
  mysqlTable, 
  int, 
  varchar, 
  text, 
  timestamp, 
  boolean, 
  date,
  time,
  decimal,
  json,
} from 'drizzle-orm/mysql2';

export const products = mysqlTable('products', {
  id: int('id').autoincrement().primaryKey(),
  name: varchar('name', { length: 255 }).notNull(),
  description: text('description'),
  price: decimal('price', { precision: 10, scale: 2 }),
  stock: int('stock').default(0).notNull(),
  isActive: boolean('is_active').default(true),
  createdAt: timestamp('created_at').defaultNow().notNull(),
  metadata: json('metadata'),
});
```

### SQLite Tables

```typescript
import { 
  sqliteTable, 
  integer, 
  text, 
  real,
} from 'drizzle-orm/libsql';

export const tasks = sqliteTable('tasks', {
  id: integer('id').primaryKey({ autoIncrement: true }),
  title: text('title').notNull(),
  description: text('description'),
  status: text('status', { enum: ['pending', 'in_progress', 'completed'] }),
  priority: integer('priority').default(1),
  dueDate: text('due_date', { mode: 'date' }), // ISO format date string
});
```

## Column Types and Options

### Primary Keys

```typescript
// Auto-incrementing primary key (4-byte)
id: serial('id').primaryKey();

// Auto-incrementing primary key (2-byte)
smallId: smallserial('small_id').primaryKey();

// Auto-incrementing primary key (8-byte)
bigId: bigserial('big_id').primaryKey();

// BigInt with mode option for large auto-increment IDs
// mode: 'number' returns JavaScript number (up to 2^53)
// mode: 'bigint' returns native bigint
bigIdNumber: bigserial('big_id', { mode: 'number' }).primaryKey();

// UUID primary key with default random value
uuidId: uuid('uuid_id').defaultRandom().primaryKey();

// Composite primary key (defined in table options)
export const postTags = sqliteTable('post_tags', {
  postId: integer('post_id'),
  tagId: integer('tag_id'),
}, (table) => ({
  pk: primaryKey({ columns: [table.postId, table.tagId] }),
}));
```

### Identity Columns (PostgreSQL)

Identity columns are the SQL standard replacement for SERIAL types:

```typescript
import { pgTable, integer } from 'drizzle-orm/pg-core';

// Traditional SERIAL (still supported)
export const users1 = pgTable('users1', {
  id: serial('id').primaryKey(),
});

// Identity column with GENERATED ALWAYS (cannot be overridden)
export const users2 = pgTable('users2', {
  id: integer('id').generatedAlwaysAsIdentity().primaryKey(),
});

// Identity column with GENERATED BY DEFAULT (can be overridden)
export const users3 = pgTable('users3', {
  id: integer('id').generatedByDefaultAsIdentity().primaryKey(),
});

// Identity column with custom start and increment
export const users4 = pgTable('users4', {
  id: integer('id').generatedAlwaysAsIdentity({ 
    startWith: 1000,  // Start from 1000
    increment: 5      // Increment by 5
  }).primaryKey(),
});
```

> **Note**: Identity columns are the recommended approach for new PostgreSQL tables. They follow SQL standards and provide more control than SERIAL types.

### Constraints

```typescript
// NOT NULL constraint
name: varchar('name', { length: 255 }).notNull();

// UNIQUE constraint
email: varchar('email', { length: 255 }).unique().notNull();

// DEFAULT value
status: varchar('status').default('active');
createdAt: timestamp('created_at').defaultNow().notNull();

// CHECK constraints (PostgreSQL)
age: integer('age').check((age) => age >= 0);

// REFERENCES / FOREIGN KEY
authorId: integer('author_id')
  .references(() => users.id, { onDelete: 'cascade' });

// ON UPDATE actions (PostgreSQL)
authorId2: integer('author_id')
  .references(() => users.id, { 
    onDelete: 'cascade',
    onUpdate: 'cascade' 
  });
```

### Array Types

```typescript
// PostgreSQL arrays
tags: text('tags').array();                    // TEXT[]
categories: varchar('categories', { length: 50 }).array(); // VARCHAR(50)[]

// MySQL JSON array (stored as JSON)
tags: json('tags');                            // JSON type

// SQLite (store as JSON string or use separate table)
tags: text('tags').$type<string[]>();         // Type hint for arrays
```

### Enum-like Types

```typescript
// PostgreSQL enum type (creates actual ENUM in database)
export const userStatus = pgEnum('user_status', [
  'active',
  'inactive',
  'suspended',
]);

export const users = pgTable('users', {
  id: serial('id').primaryKey(),
  status: userStatus().default('active'),
});

// MySQL enum type (varchar with enum for type inference)
export const products = mysqlTable('products', {
  status: varchar('status', { 
    enum: ['draft', 'published', 'archived'] 
  }),
});

// SQLite enum type (text with enum for type inference)
export const tasks = sqliteTable('tasks', {
  status: text('status', { enum: ['pending', 'in_progress', 'completed'] }),
});

// Text with enum for type inference (runtime check NOT performed)
// Use this when you want TypeScript type safety without creating a database ENUM
export const roles = pgTable('roles', {
  id: serial('id').primaryKey(),
  name: text('name', { enum: ['admin', 'user', 'moderator'] }).notNull(),
});
```

> **Important**: The `enum` option in varchar/text columns only provides TypeScript type inference. It does NOT enforce the enum values at the database level or runtime. Use `pgEnum` for actual database ENUM types.

## Indexes

### Automatic Indexing

- Primary keys are automatically indexed
- UNIQUE columns are automatically indexed

### Manual Index Creation

```typescript
import { index, sql } from 'drizzle-orm';

export const users = pgTable('users', {
  id: serial('id').primaryKey(),
  name: varchar('name', { length: 255 }).notNull(),
  email: varchar('email', { length: 255 }).unique().notNull(),
  createdAt: timestamp('created_at').defaultNow().notNull(),
}, (table) => ({
  // Composite index
  nameCreatedIdx: index('name_created_idx').on(table.name, table.createdAt),
  
  // Expression index (PostgreSQL)
  emailLowerIdx: index('email_lower_idx').on(sql`lower(${table.email})`),
}));

// Or using separate index definition
export const usersEmailIndex = index('users_email_idx').on(users.email);
```

### Index Types

```typescript
// B-tree (default)
index('idx_name').on(table.name);

// Hash index
index('idx_name_hash', { type: 'hash' }).on(table.name);

// GIN index for JSON/arrays (PostgreSQL)
index('idx_metadata_gin', { type: 'gin' }).on(table.metadata);

// GiST index for geospatial data
index('idx_location_gist', { type: 'gist' }).on(table.location);

// SP-GiST index
index('idx_name_spgist', { type: 'spgist' }).on(table.name);

// BRIN index (for large tables with naturally ordered data)
index('idx_created_brin', { type: 'brin' }).on(table.createdAt);
```

## Complete Schema Example

### Blog Application Schema

```typescript
import { 
  pgTable, 
  serial, 
  varchar, 
  text, 
  timestamp, 
  boolean, 
  integer,
  jsonb,
} from 'drizzle-orm/pg-core';

// Users table
export const users = pgTable('users', {
  id: serial('id').primaryKey(),
  name: varchar('name', { length: 255 }).notNull(),
  email: varchar('email', { length: 255 }).unique().notNull(),
  passwordHash: varchar('password_hash', { length: 255 }).notNull(),
  role: varchar('role', { length: 50 }).default('user').notNull(),
  isActive: boolean('is_active').default(true).notNull(),
  createdAt: timestamp('created_at').defaultNow().notNull(),
  updatedAt: timestamp('updated_at').$onUpdate(() => new Date()).notNull(),
});

// Posts table
export const posts = pgTable('posts', {
  id: serial('id').primaryKey(),
  title: varchar('title', { length: 255 }).notNull(),
  slug: varchar('slug', { length: 255 }).unique().notNull(),
  content: text('content'),
  excerpt: text('excerpt'),
  published: boolean('published').default(false).notNull(),
  authorId: integer('author_id')
    .references(() => users.id, { onDelete: 'cascade' })
    .notNull(),
  views: integer('views').default(0).notNull(),
  createdAt: timestamp('created_at').defaultNow().notNull(),
  updatedAt: timestamp('updated_at').$onUpdate(() => new Date()).notNull(),
});

// Comments table
export const comments = pgTable('comments', {
  id: serial('id').primaryKey(),
  content: text('content').notNull(),
  postId: integer('post_id')
    .references(() => posts.id, { onDelete: 'cascade' })
    .notNull(),
  authorId: integer('author_id')
    .references(() => users.id, { onDelete: 'cascade' })
    .notNull(),
  parentId: integer('parent_id').references(() => comments.id, { onDelete: 'cascade' }),
  createdAt: timestamp('created_at').defaultNow().notNull(),
});

// Tags table
export const tags = pgTable('tags', {
  id: serial('id').primaryKey(),
  name: varchar('name', { length: 100 }).unique().notNull(),
  slug: varchar('slug', { length: 100 }).unique().notNull(),
});

// Post-Tag junction table (Many-to-Many)
export const postTags = pgTable('post_tags', {
  postId: integer('post_id')
    .references(() => posts.id, { onDelete: 'cascade' }),
  tagId: integer('tag_id')
    .references(() => tags.id, { onDelete: 'cascade' }),
}, (table) => ({
  pk: primaryKey({ columns: [table.postId, table.tagId] }),
}));

// Indexes
export const users = pgTable('users', {
  id: serial('id').primaryKey(),
  name: varchar('name', { length: 255 }).notNull(),
  email: varchar('email', { length: 255 }).unique().notNull(),
}, (table) => ({
  emailIdx: index('users_email_idx').on(table.email),
}));

export const posts = pgTable('posts', {
  id: serial('id').primaryKey(),
  title: varchar('title', { length: 255 }).notNull(),
  slug: varchar('slug', { length: 255 }).unique().notNull(),
  authorId: integer('author_id')
    .references(() => users.id, { onDelete: 'cascade' })
    .notNull(),
  published: boolean('published').default(false).notNull(),
  createdAt: timestamp('created_at').defaultNow().notNull(),
}, (table) => ({
  authorIdx: index('posts_author_idx').on(table.authorId),
  publishedIdx: index('posts_published_idx').on(table.published),
  slugIdx: index('posts_slug_idx').on(table.slug),
}));

// Export schema type
export type Schema = {
  users: typeof users;
  posts: typeof posts;
  comments: typeof comments;
  tags: typeof tags;
  postTags: typeof postTags;
};
```

## Best Practices

1. **Use meaningful names**: Table and column names should be descriptive
2. **Consistent naming**: Use snake_case for database columns, camelCase in TypeScript
3. **Add indexes strategically**: Index frequently queried columns
4. **Use constraints**: Leverage foreign keys, unique constraints, and check constraints
5. **Version control schemas**: Keep schema files under version control
6. **Document relationships**: Add comments explaining complex relationships
7. **Use Identity columns in PostgreSQL**: Prefer `generatedAlwaysAsIdentity()` over SERIAL for new tables
8. **Use pgEnum for strict enums**: When you need database-level enum enforcement, use `pgEnum`
9. **Use text({ enum }) for type inference**: When you only need TypeScript safety without DB-level enforcement

## References

- [Drizzle Schema Declaration](https://orm.drizzle.team/docs/sql-schema-declaration)
- [PostgreSQL Column Types](https://www.postgresql.org/docs/current/datatype.html)
- [MySQL Column Types](https://dev.mysql.com/doc/refman/8.0/en/data-types.html)
