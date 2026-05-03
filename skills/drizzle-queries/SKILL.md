---
name: drizzle-queries
description: Drizzle ORM query builder including select, insert, update, delete operations, joins, subqueries, aggregations, and advanced querying patterns. Use when writing database queries, performing CRUD operations, or building complex data retrieval logic in Drizzle ORM.
license: MIT
metadata:
  author: snowmerak
  version: "1.0"
  framework: drizzle-orm
  category: queries
---

# Drizzle Query Builder Skills

This skill covers comprehensive query patterns in Drizzle ORM including CRUD operations, joins, subqueries, and aggregations.

## Basic Setup

```typescript
import { drizzle } from 'drizzle-orm/node-postgres';
import { Pool } from 'pg';
import { users, posts } from './schema';

const pool = new Pool({ connectionString: process.env.DATABASE_URL });
const db = drizzle(pool);
```

## SELECT Queries

### Basic Select

```typescript
// Get all records
const allUsers = await db.select().from(users);

// Get specific columns
const userNames = await db.select({ name: users.name, email: users.email }).from(users);

// With alias
const result = await db.select({
  fullName: sql`${users.firstName} ${users.lastName}`,
}).from(users);
```

### WHERE Clauses

```typescript
import { eq, and, or, gt, gte, lt, lte, like, ilike, inArray, notInArray, isNull, isNotNull } from 'drizzle-orm';

// Single condition
const user = await db.select().from(users).where(eq(users.id, 1)).limit(1);

// Multiple conditions with AND
const activeUsers = await db.select()
  .from(users)
  .where(and(
    eq(users.isActive, true),
    gt(users.age, 18),
    like(users.name, '%John%')
  ));

// Multiple conditions with OR
const usersByNameOrEmail = await db.select()
  .from(users)
  .where(or(
    like(users.name, '%John%'),
    eq(users.email, 'john@example.com')
  ));

// IN clause
const userInIds = [1, 2, 3];
const usersByIds = await db.select().from(users).where(inArray(users.id, userInIds));

// NOT IN clause
const excludedUsers = await db.select().from(users).where(notInArray(users.id, userInIds));

// NULL checks
const usersWithNoEmail = await db.select().from(users).where(isNull(users.email));
const usersWithEmail = await db.select().from(users).where(isNotNull(users.email));

// ILIKE (case-insensitive like) - PostgreSQL only
const caseInsensitiveUsers = await db.select()
  .from(users)
  .where(ilike(users.name, '%john%'));
```

### ORDER BY and LIMIT

```typescript
import { asc, desc } from 'drizzle-orm';

// Order by single column
const newestPosts = await db.select().from(posts).orderBy(desc(posts.createdAt));

// Order by multiple columns
const sortedUsers = await db.select()
  .from(users)
  .orderBy(asc(users.name), desc(users.createdAt));

// Pagination
const page = 1;
const pageSize = 10;
const paginatedPosts = await db.select()
  .from(posts)
  .orderBy(desc(posts.createdAt))
  .limit(pageSize)
  .offset((page - 1) * pageSize);
```

### DISTINCT and GROUP BY

```typescript
// DISTINCT values
const uniqueRoles = await db.select({ role: users.role }).from(users).distinct();

// GROUP BY with aggregation
import { count, avg, sum, min, max } from 'drizzle-orm';

const postsByAuthor = await db.select({
  authorId: posts.authorId,
  postCount: count(posts.id),
  totalViews: sum(posts.views),
})
.from(posts)
.groupBy(posts.authorId);

// HAVING clause (PostgreSQL)
const authorsWithManyPosts = await db.select({
  authorId: posts.authorId,
  postCount: count(posts.id),
})
.from(posts)
.groupBy(posts.authorId)
.having(gte(count(posts.id), 10));
```

## INSERT Operations

### Single Insert

```typescript
// Return inserted row
const newUser = await db.insert(users).values({
  name: 'John Doe',
  email: 'john@example.com',
}).returning();

// Get ID only
const result = await db.insert(users).values({
  name: 'Jane Doe',
  email: 'jane@example.com',
}).returning({ id: users.id });

// Insert with default values
const userWithDefaults = await db.insert(users).values({
  name: 'Default User',
  // Other fields use defaults
}).returning();
```

### Multiple Inserts

```typescript
// Insert multiple rows
await db.insert(users).values([
  { name: 'User 1', email: 'user1@example.com' },
  { name: 'User 2', email: 'user2@example.com' },
  { name: 'User 3', email: 'user3@example.com' },
]);

// Insert with returning
const newUsers = await db.insert(users).values([
  { name: 'Bulk User 1', email: 'bulk1@example.com' },
  { name: 'Bulk User 2', email: 'bulk2@example.com' },
]).returning();
```

### Upsert (ON CONFLICT)

```typescript
// PostgreSQL upsert
await db.insert(users).values({
  id: 1,
  name: 'Updated Name',
  email: 'updated@example.com',
}).onConflictDoUpdate({
  target: users.email,
  set: { name: 'Updated Name' },
});

// MySQL upsert
await db.insert(users).values({
  id: 1,
  name: 'Updated Name',
  email: 'updated@example.com',
}).onDuplicateKeyUpdate({
  set: { name: 'Updated Name' },
});

// SQLite upsert
await db.insert(users).values({
  id: 1,
  name: 'Updated Name',
  email: 'updated@example.com',
}).onConflictDoNothing();
```

## UPDATE Operations

### Basic Update

```typescript
// Single column update
await db.update(users)
  .set({ name: 'New Name' })
  .where(eq(users.id, 1));

// Multiple columns update
await db.update(users)
  .set({
    name: 'Updated Name',
    email: 'updated@example.com',
    updatedAt: new Date(),
  })
  .where(eq(users.id, 1))
  .returning();

// Update with condition
const updatedCount = await db.update(posts)
  .set({ published: true })
  .where(and(
    eq(posts.authorId, 1),
    eq(posts.published, false)
  ));
```

### Conditional Updates

```typescript
import { sql } from 'drizzle-orm';

// Increment/decrement
await db.update(users)
  .set({ views: sql`${users.views} + 1` })
  .where(eq(users.id, 1));

// Update based on another column
await db.update(posts)
  .set({ updatedAt: new Date() })
  .where(sql`${posts.updatedAt} < ${new Date(Date.now() - 24 * 60 * 60 * 1000)}`);
```

## DELETE Operations

### Basic Delete

```typescript
// Single delete
await db.delete(users).where(eq(users.id, 1));

// Multiple deletes
await db.delete(posts).where(eq(posts.authorId, 1));

// Delete with returning (PostgreSQL only)
const deletedUsers = await db.delete(users)
  .where(eq(users.isActive, false))
  .returning();
```

### Conditional Deletes

```typescript
import { lt } from 'drizzle-orm';

// Delete old records
await db.delete(posts).where(lt(posts.createdAt, new Date('2023-01-01')));

// Delete with multiple conditions
await db.delete(comments)
  .where(and(
    eq(comments.postId, 1),
    isNull(comments.parentId)
  ));
```

## JOIN Operations

### INNER JOIN

```typescript
import { eq } from 'drizzle-orm';

// Simple join
const postsWithAuthors = await db.select({
  postTitle: posts.title,
  authorName: users.name,
})
.from(posts)
.innerJoin(users, eq(posts.authorId, users.id));

// Multiple joins
const commentsWithDetails = await db.select({
  commentContent: comments.content,
  postTitle: posts.title,
  authorName: users.name,
})
.from(comments)
.innerJoin(posts, eq(comments.postId, posts.id))
.innerJoin(users, eq(comments.authorId, users.id));
```

### LEFT JOIN / RIGHT JOIN

```typescript
import { eq } from 'drizzle-orm';

// Left join (include all posts even without comments)
const postsWithComments = await db.select({
  postTitle: posts.title,
  commentCount: count(comments.id),
})
.from(posts)
.leftJoin(comments, eq(posts.id, comments.postId))
.groupBy(posts.id);

// Right join (include all users even without posts)
const usersWithPosts = await db.select({
  userName: users.name,
  postTitle: posts.title,
})
.from(users)
.rightJoin(posts, eq(users.id, posts.authorId));
```

### FULL OUTER JOIN

```typescript
import { eq } from 'drizzle-orm';

// Full outer join (PostgreSQL only)
const allUsersAndPosts = await db.select({
  userName: users.name,
  postTitle: posts.title,
})
.from(users)
.fullOuterJoin(posts, eq(users.id, posts.authorId));
```

### Cross Join

```typescript
// Cross join (Cartesian product) - use with caution!
const combinations = await db.select()
  .from(users)
  .crossJoin(tags);
```

## Subqueries

### Correlated Subquery

```typescript
import { eq, count } from 'drizzle-orm';

// Get users with their post counts using subquery
const usersWithPostCounts = await db.select({
  name: users.name,
  email: users.email,
  postCount: (subQuery) => subQuery
    .select({ count: count() })
    .from(posts)
    .where(eq(posts.authorId, users.id)),
}).from(users);

// Alternative using selectFrom
const result = await db.selectFrom(users).selectAll().execute();
```

### Subquery in WHERE

```typescript
import { eq, inArray } from 'drizzle-orm';

// Get posts by authors with more than 10 posts
const popularAuthorsPosts = await db.select()
  .from(posts)
  .where(inArray(
    posts.authorId,
    db.select({ authorId: posts.authorId })
      .from(posts)
      .groupBy(posts.authorId)
      .having(gte(count(posts.id), 10))
  ));

// Get users who have no posts
const inactiveUsers = await db.select()
  .from(users)
  .where(notInArray(
    users.id,
    db.select({ authorId: posts.authorId }).from(posts).distinct()
  ));
```

## Aggregations

### Count, Sum, Average, Min, Max

```typescript
import { count, avg, sum, min, max } from 'drizzle-orm';

// Total user count
const totalUsers = await db.select({ count: count() }).from(users);

// Average age of users
const averageAge = await db.select({ avg: avg(users.age) }).from(users);

// Sum of views across all posts
const totalViews = await db.select({ sum: sum(posts.views) }).from(posts);

// Min and max dates
const dateRange = await db.select({
  earliest: min(posts.createdAt),
  latest: max(posts.createdAt),
}).from(posts);
```

### Grouped Aggregations

```typescript
import { count, avg } from 'drizzle-orm';

// Posts per author with average views
const authorStats = await db.select({
  authorId: posts.authorId,
  userName: users.name,
  postCount: count(posts.id),
  avgViews: avg(posts.views),
})
.from(posts)
.innerJoin(users, eq(posts.authorId, users.id))
.groupBy(posts.authorId, users.name);

// Monthly post statistics
import { sql } from 'drizzle-orm';

const monthlyStats = await db.select({
  month: sql<string>`to_char(${posts.createdAt}, 'YYYY-MM')`,
  postCount: count(posts.id),
  totalViews: sum(posts.views),
})
.from(posts)
.groupBy(sql`to_char(${posts.createdAt}, 'YYYY-MM')`)
.orderBy(sql`min(${posts.createdAt})`);
```

## Advanced Query Patterns

### Raw SQL Queries

```typescript
import { sql } from 'drizzle-orm';

// Execute raw SQL
const result = await db.execute(
  sql`SELECT * FROM users WHERE created_at > ${new Date('2024-01-01')}`
);

// Parameterized query with type safety
const user = await db.$queryRaw`
  SELECT * FROM users WHERE id = ${userId}
`;

// Insert with raw SQL
await db.execute(
  sql`INSERT INTO users (name, email) VALUES (${name}, ${email}) RETURNING *`
);
```

### Transaction Support

```typescript
// Single transaction
const result = await db.transaction(async (tx) => {
  const newUser = await tx.insert(users).values({
    name: 'John Doe',
    email: 'john@example.com',
  }).returning();

  await tx.insert(posts).values({
    title: 'First Post',
    content: 'Hello World!',
    authorId: newUser[0].id,
  });

  return { success: true };
});

// Nested transactions (savepoints)
await db.transaction(async (tx) => {
  await tx.insert(users).values({ name: 'User 1', email: 'user1@example.com' });
  
  // Savepoint for partial rollback
  try {
    await tx.$executeTransaction(async (nestedTx) => {
      await nestedTx.insert(posts).values({ title: 'Post 1', authorId: 1 });
    });
  } catch (error) {
    // Handle error, main transaction continues
  }
});
```

### Batch Operations

```typescript
// Insert in batches
const batchSize = 100;
for (let i = 0; i < largeDataset.length; i += batchSize) {
  const batch = largeDataset.slice(i, i + batchSize);
  await db.insert(users).values(batch);
}

// Update in batches
await Promise.all(
  userIds.map(id => 
    db.update(users)
      .set({ updatedAt: new Date() })
      .where(eq(users.id, id))
  )
);
```

## Best Practices

1. **Use parameterized queries**: Always use parameter binding to prevent SQL injection
2. **Limit result sets**: Use `.limit()` for large datasets
3. **Index frequently queried columns**: Ensure proper indexes exist
4. **Use transactions for data integrity**: Wrap related operations in transactions
5. **Prefer joins over subqueries when possible**: Joins are often more efficient
6. **Use returning() wisely**: Only return needed fields to reduce memory usage
7. **Handle null values explicitly**: Use `isNotNull()` and `isNull()` checks

## References

- [Drizzle Query Builder Documentation](https://orm.drizzle.team/docs/select)
- [Drizzle Joins Documentation](https://orm.drizzle.team/docs/joins)
- [Drizzle Transactions Documentation](https://orm.drizzle.team/docs/transactions)
