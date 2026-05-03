---
name: drizzle-schema
description: Drizzle ORM schema definition including table creation, column types, constraints, indexes, and relationships. Use when defining database schemas, creating tables, or setting up data models in Drizzle ORM.
license: MIT
metadata:
  author: snowmerak
  version: '1.0'
  framework: drizzle-orm
  category: schema
---

# Drizzle Schema Definition - Tables, Columns & Constraints

## Overview

Drizzle는 코드-first 방식으로 TypeScript로 데이터베이스 스키마를 정의합니다. PostgreSQL(`pg-core`), MySQL(`mysql2`), SQLite(`libsql`) 등 DB마다 전용 모듈이 제공되며, 컬럼 타입과 제약 조건을 체이닝으로 설정합니다.

> 💡 **참고:** 상세한 컬럼 타입 예시와 복잡한 관계 패턴은 `references/schema-examples.md`를 참조하세요. 여기서는 핵심 패턴만 다룹니다.

---

## SOP: Step-by-Step Procedures

### SOP-1: PostgreSQL 테이블 정의

```typescript
import { pgTable, serial, varchar, text, timestamp, boolean, integer, uuid, jsonb } from 'drizzle-orm/pg-core';

export const users = pgTable('users', {
  // Primary Key — auto-increment (4-byte)
  id: serial('id').primaryKey(),

  // UUID PK (random 생성)
  // uuidId: uuid('uuid_id').defaultRandom().primaryKey(),

  // String types
  name: varchar('name', { length: 255 }).notNull(),
  email: varchar('email', { length: 255 }).unique().notNull(),

  // Enum-like (text + enum 옵션 — 런타임 체크 없음!)
  role: text('role', { enum: ['admin', 'user', 'moderator'] }),

  // Boolean with default
  isActive: boolean('is_active').default(true).notNull(),

  // Timestamps
  createdAt: timestamp('created_at').defaultNow().notNull(),
  updatedAt: timestamp('updated_at')
    .$onUpdate(() => new Date())
    .notNull(),

  // JSONB (반드시 $type 단언 필요)
  metadata: jsonb('metadata').$type<Record<string, unknown>>(),

  // Array type
  tags: text('tags').array(),
});
```

### SOP-2: MySQL 테이블 정의

```typescript
import { mysqlTable, int, varchar, text, timestamp, boolean, decimal } from 'drizzle-orm/mysql2';

export const products = mysqlTable('products', {
  id: int('id').autoincrement().primaryKey(),   // auto-increment
  name: varchar('name', { length: 255 }).notNull(),
  description: text('description'),
  price: decimal('price', { precision: 10, scale: 2 }),
  stock: int('stock').default(0).notNull(),
  isActive: boolean('is_active').default(true),
  createdAt: timestamp('created_at').defaultNow().notNull(),
});
```

### SOP-3: SQLite 테이블 정의

```typescript
import { sqliteTable, integer, text } from 'drizzle-orm/libsql';

export const tasks = sqliteTable('tasks', {
  id: integer('id').primaryKey({ autoIncrement: true }),
  title: text('title').notNull(),
  description: text('description'),
  status: text('status', { enum: ['pending', 'in_progress', 'completed'] }),
  priority: integer('priority').default(1),
  dueDate: text('due_date', { mode: 'date' }),   // ISO 문자열 → Date 자동 변환
});
```

### SOP-4: 제약 조건 & 인덱스

```typescript
import { pgTable, serial, varchar, timestamp } from 'drizzle-orm/pg-core';
import { index, uniqueIndex, primaryKey } from 'drizzle-orm/pg-core';

export const posts = pgTable('posts', {
  id: serial('id').primaryKey(),
  title: varchar('title', { length: 200 }).notNull(),
  slug: varchar('slug', { length: 255 }).unique().notNull(),   // ← unique 제약
  authorId: integer('author_id').references(() => users.id).notNull(),  // ← 외래 키
  createdAt: timestamp('created_at').defaultNow().notNull(),
}, (table) => ({
  // 복합 인덱스
  idxAuthorCreated: index('posts_author_created_idx')
    .on(table.authorId, table.createdAt),

  // 복합 기본키
  // pk: primaryKey({ columns: [table.userId, table.postId] }),
}));
```

### SOP-5: 외래 키 관계 정의

**1:N (User → Posts):**
```typescript
// posts 테이블에서 authorId를 users.id에 연결
export const posts = pgTable('posts', {
  id: serial('id').primaryKey(),
  title: varchar('title', { length: 200 }).notNull(),
  authorId: integer('author_id')
    .references(() => users.id, { onDelete: 'cascade' })  // ← 삭제 시 연동
    .notNull(),
});

// 쿼리에서 조인 (drizzle-queries 스킬 참조)
const userPosts = await db.select()
  .from(users)
  .innerJoin(posts, eq(users.id, posts.authorId));
```

**N:M (Post ↔ Tags):**
```typescript
export const postTags = pgTable('post_tags', {
  postId: integer('post_id').references(() => posts.id, { onDelete: 'cascade' }),
  tagId: integer('tag_id').references(() => tags.id, { onDelete: 'cascade' }),
}, (t) => ({
  pk: primaryKey({ columns: [t.postId, t.tagId] }),  // 복합 PK
}));
```

### SOP-6: 타입 추론

```typescript
import type { InferModel } from 'drizzle-orm';

// 전체 User 객체 타입 (SELECT 결과)
type User = InferModel<typeof users>;

// INSERT 시 사용할 수 있는 필드 타입 (id, createdAt 제외)
type UserInsert = InferModel<typeof users, 'insert'>;

// SELECT 시 반환되는 필드 타입
type UserSelect = InferModel<typeof users, 'select'>;
```

---

## Tool Integration

| 작업 | 도구 | 예시 |
|------|------|------|
| 스키마 파일 탐색 | `search_files` | `search_files("pgTable", "*.ts")` |
| 컬럼 타입 확인 | `read_file` | `schema.ts`에서 컬럼 정의 읽기 |
| 마이그레이션 생성 | `run_command` | `npx drizzle-kit generate` |

---

## Anti-Patterns & Guardrails

- ❌ **`text({enum})`은 런타임 체크가 아닙니다.** class-validator나 애플리케이션 레벨 검증과 함께 사용하세요. Drizzle은 타입 추론만 제공
- ❌ **`.unique()`는 컬럼 전체에 하나만 허용합니다.** 중복을 원하면 인덱스(`index()`)를 사용하세요
- ❌ **`$onUpdate()`는 DB trigger가 아닙니다.** Drizzle이 SELECT 시 현재 시간을 반환할 뿐, 실제 DB 업데이트는 애플리케이션에서 처리해야 합니다. 자동 갱신하려면 DB trigger 또는 `updatedAt` 쿼리에서 명시적 설정 필요
- ❌ **`jsonb.$type<>()`는 타입 단언일 뿐 런타임 검증이 아닙니다.** 잘못된 데이터가 들어오면 에러 없이 무시될 수 있음
- ⚠️ **외래 키 제약 조건(`references()`)은 마이그레이션 시 실제 DB FK 생성** — 삭제/업데이트 정책(`onDelete`, `onUpdate`)을 반드시 명시하세요

## Best Practices

1. 컬럼명은 snake_case(DB), 프로퍼티명은 camelCase(TypeScript)로 분리
2. 모든 NOT NULL 컬럼에 `.notNull()` 명시 (타입 안전성 확보)
3. timestamp는 `defaultNow()` + `$onUpdate()` 조합으로 생성/수정 시간 자동화
4. 외래 키는 `references(() => OtherTable.col)` 패턴 사용
5. 인덱스는 자주 조회/필터링하는 컬럼에 추가 (특히 WHERE, JOIN 대상)

## References

- [Drizzle Schema Docs](https://orm.drizzle.team/docs/table-def)
- [Drizzle Column Types](https://orm.drizzle.team/docs/column-types/postgresql)
- [Schema Examples Reference](./references/schema-examples.md)
