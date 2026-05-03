---
name: drizzle-queries
description: Drizzle ORM query builder including SELECT, INSERT, UPDATE, DELETE, joins, subqueries, and aggregation patterns. Use when writing database queries or performing CRUD operations in Drizzle ORM.
license: MIT
metadata:
  author: snowmerak
  version: '1.0'
  framework: drizzle-orm
  category: queries
---

# Drizzle Query Builder - SELECT, INSERT, UPDATE, DELETE & Joins

## Overview

Drizzle는 타입 안전한 SQL 빌더를 제공합니다. `.select()`, `.insert()`, `.update()`, `.delete()` 메서드로 모든 CRUD 작업을 수행할 수 있으며, 조인/서브쿼리/집계 함수도 지원합니다.

> 💡 **참고:** 스키마 정의(`pgTable` 등)는 `drizzle-schema` 스킬에서 다룹니다. 여기서는 쿼리 빌더 패턴만 집중합니다.

---

## SOP: Step-by-Step Procedures

### SOP-1: SELECT 기본 — 전체 조회 & 컬럼 선택

```typescript
import { eq, and, or, gt, gte, lt, lte, like, ilike } from 'drizzle-orm';

// 전체 조회 (모든 컬럼)
const allUsers = await db.select().from(users);

// 특정 컬럼만 선택
const userNames = await db.select({ name: users.name, email: users.email }).from(users);

// 조건 필터링 — 단일 조건
const activeUser = await db.select()
  .from(users)
  .where(eq(users.isActive, true))
  .limit(1);

// 다중 조건 — AND / OR 조합
const results = await db.select().from(users).where(and(
  eq(users.role, 'admin'),
  gt(users.age, 18),
  like(users.name, '%John%')   // LIKE '%John%'
));

// ILIKE (대소문자 무시) — PostgreSQL 전용
const caseInsensitive = await db.select().from(users).where(ilike(users.email, '%@gmail.com'));
```

### SOP-2: WHERE 고급 연산자

| 연산자 | 사용법 | SQL 대응 |
|--------|--------|----------|
| `eq` / `neq` | `eq(col, val)` / `neq(col, val)` | `=`, `!=` |
| `gt` / `gte` / `lt` / `lte` | `gt(col, 10)` | `>`, `>=`, `<`, `<=` |
| `inArray` / `notInArray` | `inArray(col, [1,2,3])` | `IN (...)`, `NOT IN (...)` |
| `isNull` / `isNotNull` | `isNull(col)` | `IS NULL`, `IS NOT NULL` |
| `like` / `ilike` | `like(col, '%term%')` | `LIKE`, `ILIKE` |
| `between` | `between(col, min, max)` | `BETWEEN` |

```typescript
// IN 절
const ids = [1, 2, 3];
const usersByIds = await db.select().from(users).where(inArray(users.id, ids));

// BETWEEN
const recentUsers = await db.select().from(users)
  .where(between(users.createdAt, new Date('2024-01-01'), new Date()));

// NOT IN + IS NULL 조합
await db.select().from(users).where(and(
  notInArray(users.id, [99]),
  isNull(users.deletedAt)
));
```

### SOP-3: ORDER BY & LIMIT (정렬 & 페이지네이션)

```typescript
import { asc, desc } from 'drizzle-orm';

// 단일 컬럼 정렬
const newest = await db.select().from(posts).orderBy(desc(posts.createdAt));

// 다중 컬럼 정렬
const sorted = await db.select().from(users).orderBy(asc(users.name), desc(users.createdAt));

// 페이지네이션 (offset 기반)
const page = 2;
const pageSize = 10;
const paginated = await db.select()
  .from(posts)
  .orderBy(desc(posts.createdAt))
  .limit(pageSize)
  .offset((page - 1) * pageSize);

// cursor 기반 페이지네이션 (대용량 권장)
const lastId = getLastSeenId(); // 이전 요청의 마지막 ID
const nextBatch = await db.select()
  .from(posts)
  .where(gt(posts.id, lastId))
  .orderBy(asc(posts.id))
  .limit(pageSize);
```

### SOP-4: INSERT — 단일/다중 삽입 & Upsert

```typescript
// 단일 삽입 + 반환값 받기
const newUser = await db.insert(users).values({
  name: 'John', email: 'john@example.com'
}).returning();

// 다중 삽입 (배치)
await db.insert(users).values([
  { name: 'User1', email: 'u1@test.com' },
  { name: 'User2', email: 'u2@test.com' },
]);

// PostgreSQL Upsert — ON CONFLICT DO UPDATE
await db.insert(users).values({ id: 1, name: 'Updated' }).onConflictDoUpdate({
  target: users.email,
  set: { name: 'Updated' },
});

// MySQL Upsert — ON DUPLICATE KEY UPDATE
await db.insert(users).values({ id: 1, name: 'Updated' }).onDuplicateKeyUpdate({
  set: { name: 'Updated' },
});

// SQLite Upsert — ON CONFLICT DO NOTHING
await db.insert(users).values({ id: 1, name: 'New' }).onConflictDoNothing();
```

### SOP-5: UPDATE & DELETE

```typescript
// 조건부 업데이트
await db.update(users)
  .set({ isActive: false })
  .where(eq(users.id, 42));

// 여러 컬럼 동시 업데이트
await db.update(posts)
  .set({ title: 'New Title', updatedAt: new Date() })
  .where(and(
    eq(posts.authorId, userId),
    eq(posts.status, 'draft')
  ));

// 조건부 삭제
await db.delete(tasks).where(eq(tasks.userId, userId));

// 논리적 삭제 (soft delete) — 권장 패턴
await db.update(users)
  .set({ deletedAt: new Date() })
  .where(eq(users.id, id));
```

### SOP-6: JOIN & 서브쿼리

**INNER JOIN:**
```typescript
import { sql } from 'drizzle-orm';

const userPosts = await db.select({
  userName: users.name,
  postTitle: posts.title,
}).from(users)
  .innerJoin(posts, eq(users.id, posts.authorId));
```

**LEFT JOIN:**
```typescript
import { leftJoin } from 'drizzle-orm';

const allUsersWithPosts = await db.select({
  user: users,
  postCount: count(posts.id),
}).from(users)
  .leftJoin(posts, eq(users.id, posts.authorId))
  .groupBy(users.id);
```

**서브쿼리:**
```typescript
// EXISTS 서브쿼리
const authorsWithPosts = await db.select().from(users).where(exists(
  db.select().from(posts).where(eq(posts.authorId, users.id))
));

// FROM 절 서브쿼리 (CTE 스타일)
const topAuthors = await db.select({
  name: subQuery.name,
  totalPosts: subQuery.totalPosts,
}).from(
  db.select({
    name: users.name,
    totalPosts: count(posts.id).as('totalPosts'),
  }).from(users)
   .innerJoin(posts, eq(users.id, posts.authorId))
   .groupBy(users.id)
   .as('subQuery')
).where(gt(sql`${subQuery.totalPosts}`, 10));
```

### SOP-7: 집계 함수 (COUNT, SUM, AVG, MIN, MAX)

```typescript
import { count, sum, avg, min, max } from 'drizzle-orm';

// COUNT
const [{ total }] = await db.select({ total: count() }).from(users);

// GROUP BY + HAVING
const postsByAuthor = await db.select({
  authorId: posts.authorId,
  postCount: count(posts.id),
  avgViews: avg(posts.views),
}).from(posts)
  .groupBy(posts.authorId)
  .having(gte(count(posts.id), 5));

// SUM / MIN / MAX
const stats = await db.select({
  totalRevenue: sum(products.price),
  minPrice: min(products.price),
  maxPrice: max(products.price),
}).from(products);
```

---

## Tool Integration

| 작업 | 도구 | 예시 |
|------|------|------|
| 쿼리 패턴 탐색 | `search_files` | `search_files("db.select", "*.service.ts")` |
| 스키마 참조 | `read_file` | `schema.ts`에서 테이블/컬럼 정의 확인 |
| SQL 검증 | `run_command` | `npx drizzle-kit check`로 스키마-DB 일치 확인 |

---

## Anti-Patterns & Guardrails

- ❌ **`.select()`에 `.from()` 누락 금지** — 반드시 `.from(table)` 체이닝 필요
- ❌ **대용량 테이블에서 `limit` 없이 전체 조회 금지** — 메모리 부족으로 크래시 발생. 항상 페이지네이션 적용
- ❌ **N+1 쿼리 방치 금지** — 루프 안에서 DB 호출하지 말고 `inArray()`로 한 번에 조회
- ❌ **서브쿼리 남용 금지** — 복잡한 서브쿼리는 CTE나 별도 테이블로 분리 고려
- ⚠️ **`sql`` 템플릿 리터럴은 타입 안전성이 떨어집니다.** 가능하면 빌더 메서드 우선 사용

## Best Practices

1. `eq`, `and`, `or` 등 함수형 쿼리 빌더 패턴 일관되게 사용
2. 페이지네이션은 offset(소규모) 또는 cursor(대용량) 선택
3. 논리적 삭제는 `deletedAt` 컬럼에 타임스탬프 기록 (실제 DELETE 아님)
4. 조인은 `.innerJoin()`, `.leftJoin()` 메서드 사용 (`.from().join()`보다 타입 안전)
5. 집계 쿼리는 반드시 GROUP BY + HAVING 조합으로 필터링

## References

- [Drizzle Query Builder](https://orm.drizzle.team/docs/query-builders/select)
- [Drizzle Joins](https://orm.drizzle.team/docs/queries/joins)
- [Drizzle Aggregations](https://orm.drizzle.team/docs/queries/aggregation)
