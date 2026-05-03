---
name: nestjs-database
description: NestJS database integration patterns with TypeORM, Prisma, Drizzle ORM. Use when setting up data access layers, creating repositories, or configuring database connections in NestJS.
license: MIT
metadata:
  author: snowmerak
  version: '1.0'
  category: nestjs
  tags: [database, typeorm, prisma, drizzle, repository]
---

# NestJS Database Integration - TypeORM, Prisma & Drizzle Patterns

## Overview

NestJS는 여러 ORM을 지원합니다. 이 스킬은 프로젝트에서 가장 많이 사용되는 **TypeORM**, **Drizzle ORM**(타입 안전 SQL 빌더), 그리고 **Prisma**의 통합 패턴을 다룹니다.

> 💡 **참고:** Drizzle ORM 상세 스키마/쿼리는 `drizzle-*` 계열 스킬(`drizzle-schema`, `drizzle-queries`)에서 별도 설명합니다. 여기서는 NestJS 통합 패턴만 다룹니다.

---

## SOP: Step-by-Step Procedures

### SOP-1: TypeORM 통합 설정

```bash
npm install @nestjs/typeorm typeorm pg      # 또는 mysql2, sqlite3
```

**AppModule에 등록 (Async 권장):**
```typescript
@Module({
  imports: [
    ConfigModule.forRoot(),
    TypeOrmModule.forRootAsync({
      imports: [ConfigModule],
      inject: [ConfigService],
      useFactory: (config: ConfigService) => ({
        type: 'postgres',
        host: config.get('DB_HOST'),
        port: +config.get('DB_PORT'),
        username: config.get('DB_USER'),
        password: config.get('DB_PASS'),
        database: config.get('DB_NAME'),
        entities: [__dirname + '/**/*.entity{.ts,.js}'],
        synchronize: false,            // ⚠️ 프로덕션에서는 항상 false!
      }),
    }),
  ],
})
export class AppModule {}
```

### SOP-2: TypeORM Entity & Repository 패턴

```typescript
// 1. Entity 정의
import { Entity, PrimaryGeneratedColumn, Column } from 'typeorm';

@Entity()
export class Cat {
  @PrimaryGeneratedColumn()
  id: number;

  @Column() name: string;

  @Column({ type: 'int' }) age: number;
}

// 2. Feature Module에 Repository 등록 (권장: forFeature)
@Module({
  imports: [TypeOrmModule.forFeature([Cat])], // ← Cat을 DI로 사용 가능
  providers: [CatsService],
})
export class CatsModule {}

// 3. Service에서 InjectRepository 사용
@Injectable()
export class CatsService {
  constructor(
    @InjectRepository(Cat)
    private catsRepo: Repository<Cat>,
  ) {}

  async create(dto: CreateCatDto): Promise<Cat> {
    const cat = this.catsRepo.create(dto);
    return this.catsRepo.save(cat);
  }
}
```

### SOP-3: Drizzle ORM 통합 (`drizzle-nestjs` 스킬과 연동)

Drizzle은 TypeORM처럼 별도의 Module을 제공하지 않으므로, `DrizzleService`를 직접 만듭니다.

```bash
npm install drizzle-orm pg                # 또는 better-sqlite3, mysql2
npm install -D drizzle-kit
```

**DrizzleService 생성:**
```typescript
import { Injectable, OnModuleInit } from '@nestjs/common';
import { Pool } from 'pg';
import { drizzle } from 'drizzle-orm/pg-core';
import type { PgDatabase } from 'drizzle-orm/pg-core';

@Injectable()
export class DatabaseService implements OnModuleInit {
  private pool!: Pool;
  db!: PgDatabase;

  async onModuleInit() {
    this.pool = new Pool({ /* env config */ });
    this.db = drizzle(this.pool);
  }

  getDb() { return this.db; }

  async destroy() { await this.pool.end(); }
}
```

**Feature Module에 등록:**
```typescript
@Module({
  providers: [DatabaseService, CatsService],
})
export class CatsModule {}
```

### SOP-4: Prisma 통합 패턴

```bash
npm install @prisma/client prisma
npx prisma init
```

**PrismaService (싱글톤):**
```typescript
import { Injectable, OnModuleInit, OnModuleDestroy } from '@nestjs/common';
import { PrismaClient } from '@prisma/client';

@Injectable()
export class PrismaService extends PrismaClient implements OnModuleInit, OnModuleDestroy {
  async onModuleInit() { await this.$connect(); }
  async onModuleDestroy() { await this.$disconnect(); }
}
```

**AppModule 또는 Global Module에 등록:**
```typescript
@Global()
@Module({
  providers: [PrismaService],
  exports: [PrismaService],
})
export class DatabaseModule {}
```

### SOP-5: 트랜잭션 처리

**TypeORM:**
```typescript
async createWithTransaction(dto: CreateCatDto) {
  const queryRunner = this.catsRepo.manager.connection.createQueryRunner();
  await queryRunner.connect();
  await queryRunner.startTransaction();

  try {
    // 여러 테이블 업데이트...
    await queryRunner.commitTransaction();
  } catch {
    await queryRunner.rollbackTransaction();
    throw new BadRequestException('Transaction failed');
  } finally {
    await queryRunner.release();
  }
}
```

**Drizzle:**
```typescript
// DrizzleService에서 transaction 메서드 노출
await this.db.transaction(async (tx) => {
  await tx.insert(users).values(user);
  await tx.insert(profiles).values({ userId: user.id, bio });
});
```

---

## Tool Integration

| 작업 | 도구 | 예시 |
|------|------|------|
| Entity 파일 탐색 | `search_files` | `search_files("@Entity", "*.entity.ts")` |
| DB 설정 확인 | `read_file` | `app.module.ts`, `.env` DB 변수 |
| 마이그레이션 실행 | `run_command` | `npx drizzle-kit migrate` 또는 `npm run typeorm:migration:run` |
| 스키마 검증 | `run_command` | `npx drizzle-kit check` |

---

## Anti-Patterns & Guardrails

- ❌ **프로덕션에서 `synchronize: true` 절대 금지** — DB 스키마 자동 동기화는 데이터 손실/컬럼 삭제 위험이 큽니다. 반드시 마이그레이션 사용
- ❌ **ORM 2개 이상 혼용 금지** — TypeORM + Prisma 같이 쓰지 마세요. 유지보수 지옥입니다
- ❌ **Controller에서 직접 DB 호출 금지** — 항상 Service 레이어를 거쳐야 합니다
- ❌ **N+1 쿼리 방치 금지** — `leftJoinAndSelect`(TypeORM) 또는 `with`(Drizzle)로 미리 Join 처리
- ⚠️ **`findAll()`에 Pagination 필수** — 테이블이 커지면 메모리 부족으로 크래시 발생. `limit` + `offset`/`cursor` 사용

## Best Practices

1. 환경 변수(ConfigService)로 DB 설정 관리 (`forRootAsync`)
2. Feature Module에서 `TypeOrmModule.forFeature([Entity])` 로컬 등록
3. 마이그레이션 도구는 필수 (Drizzle: `drizzle-kit`, TypeORM: CLI)
4. 트랜잭션은 반드시 try-catch-rollback 패턴 사용
5. Prisma/DrizzleService는 `@Global()`로 전역 제공

## References

- [NestJS Database Docs](https://docs.nestjs.com/techniques/database)
- [TypeORM Documentation](https://typeorm.io)
- [Prisma Documentation](https://www.prisma.io/docs)
- [Drizzle ORM](https://orm.drizzle.team)
