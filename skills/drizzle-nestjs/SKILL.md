---
name: drizzle-nestjs
description: NestJS integration with Drizzle ORM including service patterns, repository implementations, and dependency injection. Use when integrating Drizzle into NestJS applications, creating database services, or implementing data access layers.
license: MIT
metadata:
  author: snowmerak
  version: '1.0'
  framework: drizzle-orm
  category: nestjs-integration
---

# Drizzle + NestJS Integration - Service Patterns & DI

## Overview

Drizzle ORM은 별도의 NestJS Module 패키지를 제공하지 않습니다. 대신 `Pool` 기반의 `DrizzleService`를 직접 구현하여 NestJS의 DI 시스템에 통합합니다. 이 스킬은 그 패턴을 다룹니다.

> 💡 **참고:** Drizzle 스키마 정의는 `drizzle-schema`, 쿼리는 `drizzle-queries`, 마이그레이션은 `drizzle-migrations` 스킬에서 별도 설명합니다. 여기서는 NestJS 통합 패턴만 다룹니다.

---

## SOP: Step-by-Step Procedures

### SOP-1: 설치 및 기본 설정

```bash
npm install drizzle-orm pg              # PostgreSQL 예시
npm install -D drizzle-kit @types/pg    # 개발 도구
```

**의존성:**
| 용도 | 패키지 |
|------|--------|
| ORM 라이브러리 | `drizzle-orm` |
| DB 드라이버 | `pg`(PostgreSQL), `mysql2`, `better-sqlite3` 등 |
| 마이그레이션 CLI | `drizzle-kit` (dev) |

### SOP-2: DrizzleService 구현 (싱글톤)

```typescript
import { Injectable, OnModuleInit, OnModuleDestroy } from '@nestjs/common';
import { drizzle } from 'drizzle-orm/node-postgres';
import type { PgDatabase } from 'drizzle-orm/pg-core';
import { Pool } from 'pg';

@Injectable()
export class DatabaseService implements OnModuleInit, OnModuleDestroy {
  private pool!: Pool;
  db!: PgDatabase;

  async onModuleInit() {
    this.pool = new Pool({
      connectionString: process.env.DATABASE_URL,
      max: 20,
      idleTimeoutMillis: 30000,
    });
    this.db = drizzle(this.pool);
  }

  async onModuleDestroy() {
    await this.pool.end();
  }

  getDb() { return this.db; }
}
```

**핵심 포인트:**
- `OnModuleInit`에서 Pool 연결 + Drizzle 인스턴스 생성
- `OnModuleDestroy`에서 Pool 정리 (리소스 누수 방지)
- `getDb()` 메서드로 DB 인스턴스 노출

### SOP-3: Global Module로 등록

```typescript
import { Module, Global } from '@nestjs/common';
import { DatabaseService } from './database.service';

@Global()                          // ← 전역 제공 (모든 모듈에서 사용 가능)
@Module({
  providers: [DatabaseService],
  exports: [DatabaseService],
})
export class DatabaseModule {}
```

**AppModule에 등록:**
```typescript
@Module({
  imports: [DatabaseModule, UsersModule],
})
export class AppModule {}
```

### SOP-4: Service에서 Drizzle 사용 (Repository 패턴)

```typescript
import { Injectable } from '@nestjs/common';
import { eq, and, desc, count } from 'drizzle-orm';
import { DatabaseService } from '../database/database.service';
import { users, type User } from '../schema';

@Injectable()
export class UsersService {
  constructor(private readonly db: DatabaseService) {}

  async create(dto: CreateUserDto): Promise<User> {
    const result = await this.db.getDb().insert(users).values({
      name: dto.name,
      email: dto.email,
    }).returning();
    return result[0];
  }

  async findAll(page = 1, limit = 10): Promise<{ users: User[]; total: number }> {
    const db = this.db.getDb();
    const [userList, [{ count: total }]] = await Promise.all([
      db.select().from(users).orderBy(desc(users.createdAt))
        .limit(limit).offset((page - 1) * limit),
      db.select({ count: count() }).from(users),
    ]);
    return { users: userList, total };
  }

  async findById(id: number): Promise<User | null> {
    const [user] = await this.db.getDb().select()
      .from(users).where(eq(users.id, id)).limit(1);
    return user ?? null;
  }
}
```

### SOP-5: 다중 데이터베이스 (Named Connection)

여러 DB를 사용해야 하면 토큰 기반 주입을 사용합니다.

```typescript
// constants.ts
export const SECONDARY_DB = 'SECONDARY_DB';

// database.service.ts — 여러 인스턴스 관리
@Injectable()
export class DatabaseService {
  private primaryDb!: PgDatabase;
  private secondaryDb!: PgDatabase;

  async onModuleInit() {
    this.primaryDb = drizzle(new Pool({ connectionString: process.env.DB_PRIMARY }));
    this.secondaryDb = drizzle(new Pool({ connectionString: process.env.DB_SECONDARY }));
  }

  getPrimaryDb() { return this.primaryDb; }
  getSecondaryDb() { return this.secondaryDb; }
}

// Service에서 선택적 주입
@Injectable()
export class AnalyticsService {
  constructor(
    @Inject('SECONDARY_DB') private readonly db: DatabaseService,
  ) {}

  async getStats() {
    return this.db.getSecondaryDb().select().from(analytics);
  }
}
```

### SOP-6: 트랜잭션 처리

```typescript
async createWithProfile(dto: CreateUserDto): Promise<User> {
  const db = this.db.getDb();

  // Drizzle transaction — 모든 쿼리가 성공하거나 모두 롤백
  return await db.transaction(async (tx) => {
    const [user] = await tx.insert(users).values({
      name: dto.name, email: dto.email,
    }).returning();

    await tx.insert(profiles).values({
      userId: user.id, bio: dto.bio,
    });

    return user;
  });
}
```

---

## Tool Integration

| 작업 | 도구 | 예시 |
|------|------|------|
| 스키마 파일 탐색 | `search_files` | `search_files("pgTable", "*.ts")` |
| DB 설정 확인 | `read_file` | `.env`, `database.service.ts`의 Pool 설정 |
| 마이그레이션 실행 | `run_command` | `npx drizzle-kit migrate` |
| 스키마 검증 | `run_command` | `npx drizzle-kit check` |

---

## Anti-Patterns & Guardrails

- ❌ **Controller에서 직접 DB 호출 금지** — 항상 Service 레이어를 거쳐야 합니다 (NestJS 원칙)
- ❌ **Pool을 매 요청마다 생성하지 마세요** — 싱글톤 `DatabaseService`에 Pool을 보관하고 재사용하세요. 성능 치명적
- ❌ **프로덕션에서 `drizzle-kit push` 금지** — 반드시 마이그레이션(`migrate`) 사용. `push`는 개발 전용
- ❌ **`getDb()`를 매번 호출하는 패턴 남용 금지** — Service 생성자에서 `db: DatabaseService`로 주입받아 재사용
- ⚠️ **트랜잭션 내에서 await 누락 시 롤백 안 됨** — 반드시 `return await db.transaction(...)` 패턴 사용

## Best Practices

1. `DatabaseService`를 `@Global()` 모듈로 전역 제공
2. Pool은 싱글톤으로 관리 (연결 재사용)
3. 트랜잭션은 `db.transaction(async tx => {...})` 패턴 필수
4. 다중 DB는 Named Token 주입으로 분리
5. 마이그레이션은 CI/CD 파이프라인에서 자동 실행

## References

- [Drizzle ORM](https://orm.drizzle.team)
- [NestJS Database Techniques](https://docs.nestjs.com/techniques/database)
- [pg Pool Documentation](https://node-postgres.com/apis/pool)
