---
name: drizzle-core
description: Drizzle ORM core concepts including architecture, supported databases, TypeScript integration, and basic setup. Use when learning Drizzle fundamentals, choosing database drivers, or setting up initial configuration.
license: MIT
metadata:
  author: snowmerak
  version: '1.0'
  framework: drizzle-orm
  category: core
---

# Drizzle ORM Core - Architecture & Setup

## Overview

Drizzle ORM은 TypeScript-first, 경량급의 타입 안전한 SQL 빌더입니다. 런타임 오버헤드가 최소화되어 다른 ORM 대비 빠르고 가벼우며, 코드-first 방식으로 스키마를 정의합니다.

---

## SOP: Step-by-Step Procedures

### SOP-1: 설치 및 드라이버 선택

```bash
# 핵심 라이브러리 + DB 드라이버 (하나만 선택)
npm install drizzle-orm pg              # PostgreSQL (node-postgres)
npm install drizzle-orm mysql2          # MySQL
npm install drizzle-orm better-sqlite3  # SQLite (동기식)

# 개발 도구 — 마이그레이션 CLI
npm install -D drizzle-kit
```

**드라이버 선택 가이드:**

| DB | 드라이버 | 패키지 | 특징 |
|----|---------|--------|------|
| PostgreSQL | node-postgres | `drizzle-orm/node-postgres` | 가장 일반적, 비동기 |
| PostgreSQL | postgres.js | `drizzle-orm/postgres-js` | 경량, 서버리스에 적합 |
| MySQL | mysql2 | `drizzle-orm/mysql2` | MySQL 표준 드라이버 |
| SQLite (Node) | better-sqlite3 | `drizzle-orm/better-sqlite3` | 동기식, 빠름 |
| SQLite (Turso) | libsql | `drizzle-orm/libsql` | Edge/브라우저 지원 |
| Cloudflare D1 | d1-http | `drizzle-orm/d1` | 서버리스 WASM 환경 |

### SOP-2: PostgreSQL 연결 설정

```typescript
import { drizzle } from 'drizzle-orm/node-postgres';
import { Pool } from 'pg';

const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
  max: 20,                          // 최대 연결 수
  idleTimeoutMillis: 30000,         // 유휴 연결 종료 시간
  connectionTimeoutMillis: 2000,    // 연결 대기 시간
});

const db = drizzle(pool);           // Drizzle 인스턴스 생성

// 쿼리 실행 예시
const users = await db.select().from(usersTable);
```

### SOP-3: TypeScript 설정 (권장)

```json
{
  "compilerOptions": {
    "strict": true,                 // 필수 — 타입 안전성의 핵심
    "noUncheckedIndexedAccess": true,  // undefined 체크 강화
    "esModuleInterop": true
  }
}
```

**`strict: true`가 없으면 Drizzle의 타입 추론이 무력화됩니다.** 반드시 활성화하세요.

### SOP-4: 기본 아키텍처 구조

```
src/
├── schema/
│   ├── index.ts          # 모든 테이블 export 통합
│   ├── users.ts          # pgTable('users', {...})
│   └── posts.ts          # pgTable('posts', {...})
├── db/
│   └── index.ts          # drizzle(pool) 인스턴스 생성 + export
├── services/
│   └── users.service.ts  # Drizzle 쿼리 사용
└── main.ts               # 진입점
```

**db/index.ts:**
```typescript
import { drizzle } from 'drizzle-orm/node-postgres';
import { Pool } from 'pg';

const pool = new Pool({ connectionString: process.env.DATABASE_URL! });
export const db = drizzle(pool);
```

**schema/index.ts:**
```typescript
export * from './users';
export * from './posts';
```

### SOP-5: 타입 추론 활용

```typescript
import type { InferModel } from 'drizzle-orm';
import { users } from './schema';

// 테이블에서 자동 추론된 타입
type User = InferModel<typeof users>;           // SELECT 결과 전체
type UserInsert = InferModel<typeof users, 'insert'>;  // INSERT 시 허용 필드
type UserSelect = InferModel<typeof users, 'select'>;  // SELECT 반환 필드

// 실제 사용 — DTO와 통합
function createUser(dto: UserInsert): Promise<User> { /* ... */ }
```

---

## Tool Integration

| 작업 | 도구 | 예시 |
|------|------|------|
| 스키마 파일 탐색 | `search_files` | `search_files("pgTable", "*.ts")` |
| DB 연결 설정 확인 | `read_file` | `.env`, `db/index.ts` 읽기 |
| 마이그레이션 실행 | `run_command` | `npx drizzle-kit generate && npx drizzle-kit migrate` |

---

## Anti-Patterns & Guardrails

- ❌ **TypeScript `strict: false`에서 Drizzle 사용 금지** — 타입 추론이 무력화되어 장점 반감
- ❌ **매 쿼리마다 새로운 Pool 생성 금지** — 싱글톤 Pool을 재사용하세요. 연결 오버헤드가 치명적
- ❌ **프로덕션에서 `synchronize: true` 같은 자동 동기화 패턴 사용 금지** — Drizzle은 마이그레이션(`drizzle-kit migrate`)만 권장
- ⚠️ **Drizzle은 ORM이 아닙니다.** SQL 빌더입니다. 복잡한 비즈니스 로직(연관 관계 자동 로드 등)은 직접 구현해야 함

## Best Practices

1. `strict: true` TypeScript 설정 필수
2. Pool을 싱글톤으로 관리 (DB 연결 재사용)
3. 스키마 파일을 도메인별로 분리 (`schema/users.ts`, `schema/posts.ts`)
4. `InferModel`로 타입 자동 추론 활용 — 수동 타입 정의 금지
5. 마이그레이션은 항상 `drizzle-kit generate → migrate` 워크플로우 사용

## References

- [Drizzle ORM Documentation](https://orm.drizzle.team/docs/overview)
- [GitHub Repository](https://github.com/drizzle-team/drizzle-orm)
- [Drizzle Kit Docs](https://orm.drizzle.team/docs/kit-overview)
