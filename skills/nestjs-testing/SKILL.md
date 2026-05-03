---
name: nestjs-testing
description: NestJS testing strategies including unit tests, integration tests, e2e tests, and mocking. Use when writing tests for NestJS applications, mocking dependencies, or testing controllers, services, and modules.
license: MIT
metadata:
  author: snowmerak
  version: '1.0'
  category: nestjs
  tags: [testing, unit-test, e2e, jest, mocking]
---

# NestJS Testing - Unit, Integration & E2E Strategies

## Overview

NestJS는 Jest를 기본 테스트 프레임워크로 제공합니다. 단위 테스트(Service/Controller 고립), 통합 테스트(여러 컴포넌트 연동), E2E 테스트(HTTP 서버 전체) 세 가지 레벨을 지원합니다.

---

## SOP: Step-by-Step Procedures

### SOP-1: Service 단위 테스트 작성

```typescript
// cats.service.spec.ts
import { Test, TestingModule } from '@nestjs/testing';
import { CatsService } from './cats.service';

describe('CatsService', () => {
  let service: CatsService;

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [CatsService],          // ← 테스트 대상만 등록
    }).compile();

    service = module.get<CatsService>(CatsService);
  });

  it('should return all cats', () => {
    service.create({ name: 'Kitty' });
    expect(service.findAll()).toEqual([{ name: 'Kitty' }]);
  });

  it('should throw when not found', async () => {
    await expect(service.findOne(999)).rejects.toThrow();
  });
});
```

### SOP-2: Controller 테스트 (Service Mock)

**핵심 원칙:** Controller만 테스트하고, Service는 **완전히 모킹**하세요.

```typescript
import { Test, TestingModule } from '@nestjs/testing';
import { CatsController } from './cats.controller';
import { CatsService } from './cats.service';

describe('CatsController', () => {
  let controller: CatsController;

  const mockCatsService = {
    create: jest.fn(),
    findAll: jest.fn().mockReturnValue([{ name: 'Kitty' }]),
  };

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      controllers: [CatsController],
      providers: [{ provide: CatsService, useValue: mockCatsService }], // ← Mock 주입
    }).compile();

    controller = module.get<CatsController>(CatsController);
  });

  it('should create a cat', () => {
    controller.create({ name: 'Kitty' });
    expect(mockCatsService.create).toHaveBeenCalledWith({ name: 'Kitty' });
  });

  it('should return all cats', () => {
    expect(controller.findAll()).toEqual([{ name: 'Kitty' }]);
  });
});
```

### SOP-3: Integration 테스트 (실제 의존성 연동)

```typescript
import { Test, TestingModule } from '@nestjs/testing';
import { CatsModule } from './cats.module';           // ← 전체 모듈 import
import { CatsService } from './cats.service';

describe('CatsModule Integration', () => {
  let service: CatsService;

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      imports: [CatsModule],                          // ← 실제 의존성 포함
    }).compile();

    service = module.get<CatsService>(CatsService);
  });

  it('should work with real dependencies', () => {
    service.create({ name: 'Kitty' });
    expect(service.findAll()).toContainEqual(expect.objectContaining({ name: 'Kitty' }));
  });
});
```

### SOP-4: E2E 테스트 (HTTP 서버 전체)

1. `test/jest-e2e.json` 생성 (별도 Jest 설정):
```json
{
  "moduleFileExtensions": ["js", "json", "ts"],
  "rootDir": ".",
  "testEnvironment": "node",
  "testRegex": ".e2e-spec.ts$",
  "transform": { "^.+\\.(t|j)s$": "ts-jest" }
}
```

2. E2E 테스트 작성:
```typescript
import { Test, TestingModule } from '@nestjs/testing';
import { INestApplication } from '@nestjs/common';
import * as request from 'supertest';
import { AppModule } from '../src/app.module';

describe('Cats E2E', () => {
  let app: INestApplication;

  beforeAll(async () => {
    const moduleRef: TestingModule = await Test.createTestingModule({
      imports: [AppModule],
    }).compile();

    app = moduleRef.createNestApplication();
    await app.init();
  });

  afterAll(async () => { await app.close(); });

  it('GET /cats → 200', () =>
    request(app.getHttpServer()).get('/cats').expect(200));

  it('POST /cats → 201', () =>
    request(app.getHttpServer())
      .post('/cats')
      .send({ name: 'Kitty' })
      .expect(201));
});
```

### SOP-5: Provider/Guard/Pipe Override 패턴

```typescript
// DB 모듈 Mock — In-memory SQLite 사용
TypeOrmModule.forRoot({
  type: 'sqlite', database: ':memory:', synchronize: true,
})

// Guard bypass — 테스트 시 인증 스킵
.overrideGuard(AuthGuard)
.useValue({ canActivate: () => true })
.compile()

// Service 전체 Mock
.overrideProvider(PaymentsService)
.useValue({ processPayment: jest.fn().mockResolvedValue(true) })
.compile()

// Module 전체 Mock (외부 모듈 대체)
.overrideModule(HttpModule)
.useModule({ providers: [{ provide: HttpService, useValue: mockHttp }] })
```

### SOP-6: 테스트 실행 명령어

```bash
npm run test          # 단위 테스트 (*.spec.ts)
npm run test:watch    # watch 모드 — 파일 변경 자동 재실행
npm run test:cov      # 코드 커버리지 리포트 생성
npm run test:e2e      # E2E 테스트 (test/*.e2e-spec.ts)
```

---

## Tool Integration

| 작업 | 도구 | 예시 |
|------|------|------|
| 테스트 파일 탐색 | `search_files` | `search_files("describe(", "*.spec.ts")` |
| 테스트 실행 | `run_command` | `npm run test -- --testPathPattern=cats` |
| 커버리지 확인 | `run_command` | `npm run test:cov && read_file coverage/lcov-report/index.html` |
| Jest 설정 읽기 | `read_file` | `jest.config.js`, `package.json` 테스트 스크립트 |

---

## Anti-Patterns & Guardrails

- ❌ **Service를 모킹하지 않고 Controller 테스트 금지** — Controller 테스트는 Service 의존성을 완전히 격리해야 함
- ❌ **"should be defined" 테스트 금지** — 의미 없는 확인 테스트. 실제 동작을 검증하세요
- ❌ **E2E 테스트에서 프로덕션 DB 사용 금지** — In-memory SQLite 또는 별도 테스트 DB 사용
- ❌ **`beforeAll`에 테스트 데이터 삽입 후 정리 안 함** — `afterEach`/`afterAll`에서 반드시 정리
- ⚠️ **Mock 함수의 반환값 타입 실제와 다르면 테스트는 통과하지만 런타임 에러 발생** — Mock 시gnature 일치 확인 필수

## Best Practices

1. 행동(Behavior)을 테스트하세요, 구현(Implementation)이 아닌
2. 외부 의존성은 항상 Mock (DB, HTTP, Queue 등)
3. 엣지 케이스 테스트: null, undefined, 빈 배열, 에러 상황
4. 테스트명 명시적: `"should return empty array when no cats exist"`
5. 테스트 독립성 — 각 `it` 블록이 단독으로 실행 가능해야 함

## References

- [NestJS Testing Docs](https://docs.nestjs.com/fundamentals/testing)
- [Jest Documentation](https://jestjs.io)
- [Supertest Documentation](https://github.com/ladjs/supertest)
