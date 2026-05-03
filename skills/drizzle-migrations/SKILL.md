---
name: drizzle-migrations
description: Drizzle ORM migrations using drizzle-kit including schema generation, migration creation, applying migrations, rollback strategies, and multi-environment deployment workflows. Use when managing database schema changes or deploying in production with Drizzle ORM.
license: MIT
metadata:
  author: snowmerak
  version: '1.0'
  framework: drizzle-orm
  category: migrations
---

# Drizzle Migrations - drizzle-kit Workflow & Deployment

## Overview

Drizzle ORM은 `drizzle-kit` CLI를 통해 마이그레이션 파일을 생성하고 적용합니다. 코드-first 접근 방식으로 TypeScript 스키마 정의가 SQL 마이그레이션으로 자동 변환됩니다.

---

## SOP: Step-by-Step Procedures

### SOP-1: drizzle-kit 설치 및 초기화

```bash
npm install -D drizzle-kit
npx drizzle-kit init
```

**생성된 파일:** `drizzle.config.ts` 또는 `drizzle.config.json`

### SOP-2: drizzle.config.ts 설정

**기본 구성 (v0.25+):**
```typescript
import { defineConfig } from 'drizzle-kit';

export default defineConfig({
  dialect: 'postgresql',              // 'mysql' | 'sqlite' | 'mssql' 등
  schema: './src/schema/**/*.ts',     // 스키마 파일 경로 (glob 패턴 지원)
  out: './drizzle',                   // 마이그레이션 파일 출력 디렉토리
});
```

**전체 옵션:**
```typescript
export default defineConfig({
  dialect: 'postgresql',
  schema: './src/schema/**/*.ts',
  out: './drizzle',

  // DB 연결 (환경 변수에서 읽기 권장)
  dbCredentials: { url: process.env.DATABASE_URL! },

  // 마이그레이션 테이블 설정
  migrations: {
    table: '__drizzle_migrations__',
    schema: 'public',
    prefix: '0000',                   // 마이그레이션 폴더 접두사
  },

  // 인트로스펙션 (DB → 코드) 옵션
  introspect: { casing: 'camel' },

  // 디버그
  strict: true,                       // 엄격한 스키마 검증
  verbose: true,                      // 상세 출력
});
```

### SOP-3: package.json 스크립트 등록 (권장)

```json
{
  "scripts": {
    "db:generate": "drizzle-kit generate",     // 마이그레이션 파일 생성
    "db:migrate":   "drizzle-kit migrate",      // DB에 적용
    "db:push":      "drizzle-kit push",         // 스키마 직접 동기화 (개발 전용)
    "db:pull":      "drizzle-kit pull",         // 기존 DB에서 스키마 추출
    "db:check":     "drizzle-kit check",        // 스키마-DB 일치 여부 확인
    "db:up":        "drizzle-kit up",           // 모든 미적용 마이그레이션 적용
    "db:down":      "drizzle-kit down",         // 마지막 마이그레이션 롤백
    "db:studio":    "drizzle-kit studio",       // GUI 데이터 브라우저 (http://localhost:5173)
  }
}
```

### SOP-4: 개발 워크플로우

```bash
# 1. 스키마 코드 수정 (src/schema/*.ts)
# 2. 마이그레이션 파일 생성 (DB에는 적용 안 됨!)
npm run db:generate

# 3. 생성된 파일 확인 — drizzle/YYYY-MM-DD_*.sql
ls drizzle/

# 4. DB에 적용
npm run db:migrate

# 5. 변경 사항 검증
npm run db:check          # ✅ "Your schema matches the database." 출력되면 OK

# 또는 GUI로 확인
npm run db:studio         # http://localhost:5173에서 테이블/데이터 조회
```

**개발 중 빠른 테스트 (push 모드):**
```bash
# 마이그레이션 파일 없이 DB에 직접 적용 — 개발 전용!
npm run db:push

# ⚠️ 프로덕션에서는 절대 사용하지 마세요. 데이터 손실 위험이 있습니다.
```

### SOP-5: 프로덕션 배포 워크플로우

```bash
# 1. CI/CD에서 마이그레이션 파일 생성 + 커밋
npx drizzle-kit generate --config=drizzle.config.ts
git add drizzle/ && git commit -m "chore: migrate <date>"

# 2. 프로덕션 서버에서 마이그레이션 적용
npm run db:migrate        # 또는 npx drizzle-kit migrate

# 3. 상태 확인
npx drizzle-kit check     # ✅ 일치하는지 검증
```

**CI/CD 파이프라인 예시 (GitHub Actions):**
```yaml
- name: Apply Drizzle Migrations
  run: |
    npx drizzle-kit migrate --config=drizzle.config.ts
    npx drizzle-kit check --config=drizzle.config.ts
  env:
    DATABASE_URL: ${{ secrets.PROD_DATABASE_URL }}
```

### SOP-6: 롤백 & 마이그레이션 관리

```bash
# 마지막 마이그레이션 롤백 (드러울 수 있음)
npm run db:down           # 또는 npx drizzle-kit down

# 특정 버전으로 롤백
npx drizzle-kit down --name 0001_initial

# 마이그레이션 상태 확인
npx drizzle-kit check     # 스키마-DB 일치 여부

# dry-run (적용 전 미리보기)
npx drizzle-kit migrate --dry-run
```

**마이그레이션 파일 구조:**
```
drizzle/
├── 0001_create_users.sql      # 생성된 SQL 마이그레이션
├── 0002_add_posts.sql
└── _drizzle_migrations.json   # 적용 이력 추적 (자동 관리)
```

### SOP-7: 기존 DB에서 스키마 추출 (Pull)

기존 데이터베이스가 있는데 Drizzle로 마이그레이션하고 싶을 때:

```bash
# 1. 기존 DB 연결 설정
export DATABASE_URL="postgresql://user:pass@localhost/dbname"

# 2. 스키마 추출 + TypeScript 코드 생성
npx drizzle-kit pull

# 3. 생성된 파일 확인 — src/schema/에 자동 생성됨
```

---

## Tool Integration

| 작업 | 도구 | 예시 |
|------|------|------|
| 마이그레이션 상태 확인 | `run_command` | `npx drizzle-kit check` |
| 마이그레이션 적용 | `run_command` | `npm run db:migrate` |
| 스키마 파일 탐색 | `search_files` | `search_files("pgTable", "*.ts")` |
| 생성된 SQL 확인 | `read_file` | `drizzle/0001_*.sql` 읽기 |

---

## Anti-Patterns & Guardrails

- ❌ **프로덕션에서 `push` 절대 금지** — 마이그레이션 파일 없이 DB를 직접 수정하면 버전 관리가 불가능해집니다. 항상 `migrate` 사용
- ❌ **마이그레이션 파일 수동 수정 금지** — `drizzle/` 폴더의 SQL 파일을 직접 편집하지 마세요. 스키마 코드를 수정하고 `generate`로 재생성하세요
- ❌ **`.gitignore`에 `drizzle/` 추가 금지** — 마이그레이션 파일은 반드시 커밋하여 팀 협업과 배포에 필수입니다
- ❌ **동일한 마이그레이션을 여러 환경에서 중복 적용하지 마세요** — `__drizzle_migrations__` 테이블이 자동 추적하지만, 수동으로 지우지 마세요
- ⚠️ **컬럼 삭제/타입 변경은 데이터 손실 위험** — 마이그레이션 전 반드시 백업 또는 데이터 이전 전략 수립

## Best Practices

1. `db:generate` → `db:migrate` 순서로 항상 마이그레이션 파일 먼저 생성 후 적용
2. 프로덕션 배포 시 CI/CD에서 자동 `migrate` + `check` 실행
3. 개발 환경은 `push`로 빠르게 테스트, 프로덕션은 반드시 `migrate`
4. 마이그레이션 파일은 Git에 커밋 (버전 관리 필수)
5. `db:studio`로 GUI에서 데이터 확인 및 디버깅

## References

- [Drizzle Kit Overview](https://orm.drizzle.team/docs/kit-overview)
- [Drizzle Migrations](https://orm.drizzle.team/docs/migrate)
- [Drizzle Studio](https://orm.drizzle.team/studio/overview)
