---
name: nestjs-cli
description: NestJS CLI commands and generators for scaffolding projects, creating modules, controllers, services, and managing the development workflow. Use when setting up NestJS projects, generating code, or using CLI commands.
license: MIT
metadata:
  author: snowmerak
  version: '1.0'
  category: nestjs
  tags: [cli, generators, scaffolding, project-setup]
---

# NestJS CLI - Code Generation & Project Management

## Overview

NestJS CLI (`@nestjs/cli`)는 프로젝트 생성 및 코드 스캐폴딩을 위한 핵심 도구입니다. `nest g`(generate) 명령어로 Controller, Service, Module 등을 일관된 패턴으로 자동 생성합니다.

---

## SOP: Step-by-Step Procedures

### SOP-1: 새 프로젝트 생성

```bash
# 기본 생성 (npm 패키지 매니저, 표준 TS 설정)
nest new project-name

# 엄격한 TypeScript 설정 권장
nest new project-name --strict

# pnpm 사용
nest new project-name --package-manager pnpm

# yarn 사용
nest new project-name --package-manager yarn
```

**생성된 주요 파일:**
- `src/main.ts` — 진입점
- `src/app.module.ts` — 루트 모듈
- `tsconfig.json` / `nest-cli.json` — 빌드/CLI 설정

### SOP-2: Resource (Controller + Service + Module) 한 번에 생성

```bash
# 전체 CRUD 리소스 생성 (권장)
nest g resource users

# 프롬프트 응답 순서:
# 1. Transport layer → REST API (Enter)
# 2. CRUD entry points? → Yes (y)
# 3. DTO import source → class-transformer, class-validator 선택

# 모듈 안에 flat 구조로 생성
nest g resource modules/orders --flat
```

**생성되는 파일들:**
- `users.controller.ts` — CRUD 엔드포인트
- `users.service.ts` — CRUD 메서드
- `users.module.ts` — 모듈 정의
- `dto/create-user.dto.ts`, `dto/update-user.dto.ts` — 검증 DTO
- `entities/user.entity.ts` — 엔티티 (TypeORM 사용 시)
- `*.spec.ts` — 테스트 파일

### SOP-3: 개별 코드 생성

| 명령어 | 설명 | 옵션 |
|--------|------|------|
| `nest g controller name` | Controller | `--flat`, `--no-spec` |
| `nest g service name` | Service (Provider) | `--flat`, `--no-spec` |
| `nest g module name` | Module | `--flat` |
| `nest g provider name` | 일반 Provider | `--flat`, `--no-spec` |
| `nest g class name` | 클래스 | `--flat` |
| `nest g interface name` | 인터페이스 | `--flat` |
| `nest g guard name` | Guard (인증/인가) | `--flat` |
| `nest g interceptor name` | Interceptor | `--flat` |
| `nest g pipe name` | Pipe (변환/검증) | `--flat` |
| `nest g filter name` | Exception Filter | `--no-spec` |
| `nest g middleware name` | Middleware | `--flat` |
| `nest g gateway name` | WebSocket Gateway | `--flat`, `--no-spec` |

**옵션 설명:**
- `--dry-run`: 파일 생성 없이 출력만 확인
- `--spec`: 테스트 파일 생성 (기본값: true)
- `--no-spec`: 테스트 파일 안 만듦
- `--flat`: 서브디렉토리 없이 현재 폴더에 직접 생성

### SOP-4: 개발/빌드/테스트 명령어

```bash
# 개발 서버 (watch 모드)
npm run start          # 또는 nest start --watch

# 핫 리로드 (nodemon)
npm run start:dev

# 프로덕션 빌드
npm run build          # 또는 nest build

# SWC 컴파일러 사용 (20배 빠름)
nest build -b swc
nest start --watch -b swc

# 린팅 & 포맷팅
npm run lint           # ESLint 자동 수정
npm run format         # Prettier 적용

# 테스트
npm run test           # 단위 테스트
npm run test:watch     # watch 모드
npm run test:cov       # 코드 커버리지
npm run test:e2e       # E2E 테스트
```

### SOP-5: nest-cli.json 설정 커스터마이징

```json
{
  "$schema": "https://json.schemastore.org/nest-cli",
  "collection": "@nestjs/schematics",
  "sourceRoot": "src",
  "compilerOptions": {
    "deleteOutDir": true,
    "webpack": true,
    "tsConfigPath": "tsconfig.build.json",
    "plugins": [
      {
        "name": "@nestjs/swagger",
        "options": {
          "classValidatorShim": true,
          "introspectComments": true
        }
      }
    ]
  }
}
```

**주요 설정:**
- `webpack: true` → Webpack 컴파일러 사용 (디버깅 용이)
- `deleteOutDir: true` → 빌드 전 dist 폴더 삭제
- Swagger 플러그인 → 자동 OpenAPI 문서 생성

---

## Tool Integration

| 작업 | 도구 | 예시 |
|------|------|------|
| 프로젝트 구조 확인 | `list_dir` / `read_file` | `src/`, `nest-cli.json` 읽기 |
| CLI 명령 실행 | `run_command` | `nest g resource orders --flat` |
| 기존 파일 패턴 탐색 | `search_files` | `search_files("nest g", "*.md")` |
| 빌드 확인 | `run_command` | `npm run build && list_dir dist/` |

---

## Anti-Patterns & Guardrails

- ❌ **CLI 생성 코드 맹신 금지** — `nest g resource --crud`가 생성한 코드는 템플릿일 뿐, 프로젝트 요구사항에 맞게 수정해야 함
- ❌ **프로덕션에서 `synchronize: true` 사용 금지** — DB 스키마 자동 동기화는 데이터 손실 위험이 있음. 마이그레이션 도구 사용
- ❌ **전역 CLI 설치(`npm i -g`)는 팀 협업 시 버전 충돌 가능** — `npx nest` 또는 프로젝트 로컬 의존성으로 실행 권장

## Best Practices

1. `--strict` 옵션으로 TypeScript 엄격 모드 활성화
2. Feature Module 패턴: 각 도메인별로 별도 모듈 생성
3. `nest g resource`로 초기 스캐폴딩 후 커스터마이징
4. SWC 컴파일러로 개발 시 빌드 속도 향상 (`-b swc`)
5. 린팅/포맷팅을 pre-commit hook에 포함

## References

- [NestJS CLI Documentation](https://docs.nestjs.com/cli/overview)
- [NestJS CRUD Generator](https://docs.nestjs.com/recipes/crud-generator)
- [NestJS Schematics GitHub](https://github.com/nestjs/schematics)
