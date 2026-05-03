---
name: nestjs-core
description: Core NestJS concepts including controllers, modules, routing, and basic application structure. Use when building NestJS applications, creating controllers, defining routes, or understanding NestJS architecture.
license: MIT
metadata:
  author: snowmerak
  version: '1.0'
  category: nestjs
  tags: [core, fundamentals, controllers, modules, routing]
---

# NestJS Core - Fundamentals & Architecture

## Overview

NestJS is a progressive Node.js framework for building efficient, scalable server-side applications. It uses TypeScript and combines OOP, FP, and FRP paradigms. Under the hood, NestJS uses Express (default) or Fastify as the HTTP platform.

Every NestJS app consists of **Modules** → **Controllers** → **Providers**. This is the foundation everything else builds on.

---

## SOP: Step-by-Step Procedures

### SOP-1: 새 NestJS 프로젝트 생성

1. `nest new project-name` 실행 (또는 `nest new project-name --strict`으로 엄격한 TS 설정)
2. 패키지 매니저 지정 필요시 `--package-manager pnpm` 추가
3. 생성된 프로젝트 구조 확인:
   - `src/main.ts` — 부트스트랩 진입점
   - `src/app.module.ts` — 루트 모듈
4. `npm run start:dev`로 개발 서버 실행

```bash
nest new my-app --strict
cd my-app && npm run start:dev
```

### SOP-2: Feature Module + Controller 생성

1. `nest g module modules/users` → UsersModule 생성
2. `nest g controller modules/users --flat` → UsersController 생성 (`--flat`: 서브디렉토리 안 만듦)
3. `nest g service modules/users --flat` → UserService 생성
4. `UsersModule`에 controller/service 등록

```bash
# Agent 도구 활용
run_command: "nest g module modules/users"
run_command: "nest g controller modules/users --flat"
run_command: "nest g service modules/users --flat"
```

### SOP-3: Controller에서 라우트 정의

1. 클래스에 `@Controller('prefix')` 데코레이터 적용
2. 각 메서드에 HTTP 메서드 데코레이터(`@Get`, `@Post`, 등) 추가
3. 파라미터 데코레이터로 요청 데이터 접근 (`@Body`, `@Param`, `@Query`)

```typescript
@Controller('cats')
export class CatsController {
  constructor(private readonly catsService: CatsService) {}

  @Post()                           // POST /cats
  create(@Body() dto: CreateCatDto) { return this.catsService.create(dto); }

  @Get()                            // GET /cats
  findAll() { return this.catsService.findAll(); }

  @Get(':id')                       // GET /cats/:id
  findOne(@Param('id') id: string) { return this.catsService.findOne(id); }
}
```

### SOP-4: Provider (Service) 작성

1. `@Injectable()` 데코레이터로 클래스 마크
2. 생성자 인젝션으로 의존성 받기
3. 비즈니스 로직 구현 (Controller는 얇게 유지)

```typescript
@Injectable()
export class CatsService {
  create(dto: CreateCatDto) { /* ... */ }
  findAll() { /* ... */ }
}
```

### SOP-5: Module 구성

1. `@Module()` 데코레이터에 메타데이터 정의:
   - `controllers`: 이 모듈의 컨트롤러 목록
   - `providers`: 서비스/리포지토리 등
   - `imports`: 의존하는 다른 모듈
   - `exports`: 외부 모듈에서 사용할 수 있도록 내보낼 프로바이더

```typescript
@Module({
  imports: [DatabaseModule],
  controllers: [CatsController],
  providers: [CatsService],
  exports: [CatsService],           // ← importing 모듈에서 사용 가능
})
export class CatsModule {}
```

### SOP-6: 애플리케이션 부트스트랩 (`main.ts`)

```typescript
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  // 또는 Fastify 사용: new FastifyAdapter()
  await app.listen(process.env.PORT ?? 3000);
}
bootstrap();
```

---

## Tool Integration

| 작업 | 도구 | 예시 |
|------|------|------|
| 프로젝트 구조 파악 | `list_dir` / `read_file` | `src/` 내용 확인 |
| 기존 패턴 탐색 | `search_files` | `search_files("@Controller", "*.controller.ts")` |
| 코드 생성 | `run_command` | `nest g controller users --flat` |
| 서버 실행 | `run_command` | `npm run start:dev` |

---

## Anti-Patterns & Guardrails

- ❌ **Controller에 비즈니스 로직 넣지 마세요** — Controller는 요청/응답만 처리하고 Service로 위임하세요
- ❌ **Module에서 providers를 export하지 않으면 다른 모듈에서 접근 불가** — 꼭 `exports` 명시
- ❌ **`@Res()` 데코레이터 남용 금지** — NestJS 표준 응답 방식(리턴 값)을 우선 사용하세요. `passthrough: true`가 필요한 경우만 예외적으로 사용
- ❌ **모든 컨트롤러를 `AppModule`에 등록하지 마세요** — Feature Module로 분리하여 유지보수성 확보

## Best Practices

1. Controller는 얇게, Service에는 비즈니스 로직을
2. DTO 사용: 요청/응답 데이터 검증 및 직렬화
3. Feature Module로 관련 기능 그룹화
4. 생성자 인젝션 사용 (프로퍼티 인젝션 금지)
5. `ConfigModule`으로 환경 변수 관리

## References

- [NestJS Docs — Fundamentals](https://docs.nestjs.com/fundamentals/getting-started)
- [NestJS Docs — Controllers](https://docs.nestjs.com/controllers)
- [NestJS Docs — Providers](https://docs.nestjs.com/providers)
- [NestJS GitHub](https://github.com/nestjs/nest)
