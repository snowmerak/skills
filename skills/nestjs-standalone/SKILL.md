---
name: nestjs-standalone
description: NestJS standalone applications without modules. Use when building lightweight NestJS apps, microservices, scripts, or when you want to avoid the module system overhead.
license: MIT
metadata:
  author: snowmerak
  version: '1.0'
  category: nestjs
  tags: [standalone, microservices, lightweight, bootstrap]
---

# NestJS Standalone - Lightweight App Patterns

## Overview

NestJS는 모듈 시스템 없이도 애플리케이션을 부트스트랩할 수 있습니다. `NestFactory.createApplicationContext()`를 사용하면 모듈 오버헤드 없이 경량 앱을 구축할 수 있으며, 스크립트/유틸리티/마이크로서비스에 적합합니다.

> ⚠️ **참고:** 대부분의 NestJS 앱은 Module 시스템을 사용해야 합니다. Standalone 패턴은 특수한 용도만 위한 것입니다.

---

## SOP: Step-by-Step Procedures

### SOP-1: 기본 Standalone 앱 (모듈 없이)

```typescript
import { NestFactory } from '@nestjs/core';
import { Controller, Get } from '@nestjs/common';

@Controller()
class AppController {
  @Get() getHello(): string { return 'Hello World!'; }
}

async function bootstrap() {
  const app = await NestFactory.createApplicationContext();
  
  // 컨트롤러 직접 등록 (모듈 없이)
  app.registerController(AppController);
  
  await app.init();
  await app.listen(3000);
}
bootstrap();
```

### SOP-2: Standalone + Service (간단한 의존성)

```typescript
import { Injectable } from '@nestjs/common';

@Injectable()
class CatsService {
  private readonly cats = ['Kitty', 'Mittens'];
  findAll() { return this.cats; }
}

// 서비스 직접 인스턴스화 후 컨트롤러에 주입
const service = new CatsService();

@Controller('cats')
class CatsController {
  constructor(private svc: CatsService) {}
  
  @Get() findAll() { return this.svc.findAll(); }
}

async function bootstrap() {
  const app = await NestFactory.createApplicationContext();
  app.registerController(new CatsController(service));
  await app.init();
  await app.listen(3000);
}
```

### SOP-3: Standalone 마이크로서비스 (TCP/RMQ/Kafka)

```typescript
import { Transport } from '@nestjs/microservices';

async function bootstrap() {
  const app = await NestFactory.createMicroservice<MicroserviceOptions>(AppModule, {
    transport: Transport.TCP,
    options: { host: '0.0.0.0', port: 3001 },
  });

  // 또는 RabbitMQ 사용 시
  // transport: Transport.RMQ,
  // options: { urls: [`amqp://${process.env.RABBITMQ_HOST}:${process.env.RABBITMQ_PORT}`] }

  await app.listen();
}
```

### SOP-4: 스크립트/유틸리티 (HTTP 없이 실행)

```typescript
import { NestFactory } from '@nestjs/core';
import { Injectable, Module } from '@nestjs/common';

@Injectable()
class DataProcessor {
  async process() { /* 배치 처리 로직 */ }
}

@Module({ providers: [DataProcessor] })
class AppModule {}

async function run() {
  const app = await NestFactory.createApplicationContext(AppModule);
  const processor = app.get(DataProcessor);
  
  await processor.process();
  await app.close();                      // ← 반드시 종료! 프로세스가 멈춤 방지
}
run();
```

### SOP-5: Standalone에 ValidationPipe 추가

```typescript
import { ValidationPipe } from '@nestjs/common';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  
  // 전역 파이프 적용 가능
  app.useGlobalPipes(new ValidationPipe({
    whitelist: true, forbidNonWhitelisted: true, transform: true,
  }));
  
  await app.listen(3000);
}
```

---

## Tool Integration

| 작업 | 도구 | 예시 |
|------|------|------|
| 부트스트랩 파일 확인 | `read_file` | `main.ts`, `src/bootstrap.ts` 읽기 |
| 마이크로서비스 설정 | `search_files` | `search_files("createMicroservice", "*.ts")` |
| 앱 실행 | `run_command` | `npm run start:dev` 또는 `node dist/main.js` |

---

## Anti-Patterns & Guardrails

- ❌ **대규모 앱에서 Standalone 사용 금지** — 모듈 시스템의 DI, 생명주기 관리, 코드 분리가 훨씬 효과적입니다
- ❌ **마이크로서비스에서 HTTP 포트도 동시에 열면 충돌 발생** — `createMicroservice` 또는 `createHttpAdapter` 중 하나로만 설정
- ❌ **`app.close()` 호출하지 않으면 프로세스가 종료되지 않음** — 스크립트/배치 작업 후 반드시 호출
- ⚠️ **Standlone 패턴은 공식 문서에서 점차 권장하지 않는 추세입니다.** 가능하면 `@Module()` 기반 구조 사용

## Best Practices

1. 단순 유틸리티/스크립트에만 Standalone 고려
2. 마이크로서비스는 `createMicroservice` 사용 (HTTP 아답터와 분리)
3. 복잡한 의존성이 필요하면 결국 Module 시스템으로 전환
4. 배치 처리 후 `app.close()` 필수 호출

## References

- [NestJS Standalone Apps](https://docs.nestjs.com/standalone-applications)
- [NestJS Microservices](https://docs.nestjs.com/microservices/basics)
- [NestFactory API](https://docs.nestjs.com/fundamentals/application-bootstrap)
