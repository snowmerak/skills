---
name: nestjs-standalone
description: NestJS standalone applications without modules. Use when building lightweight NestJS apps, microservices, or when you want to avoid the module system overhead.
license: MIT
metadata:
  author: snowmerak
  version: "1.0"
  framework: nestjs
  category: deployment
---

# NestJS Standalone Applications Skills

This skill covers creating NestJS applications without the traditional module system.

## Overview

Standalone applications allow you to bootstrap a NestJS app without defining a root `AppModule`. This is useful for:
- Lightweight applications
- Microservices
- Scripts and utilities
- When you want to avoid module system overhead

## 1. Basic Standalone App

### Without Module

```typescript
// main.ts
import { NestFactory } from '@nestjs/core';
import { Controller, Get, Module } from '@nestjs/common';

@Controller()
class AppController {
  @Get()
  getHello(): string {
    return 'Hello World!';
  }
}

@Module({
  controllers: [AppController],
})
class AppModule {}

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  await app.listen(3000);
}
bootstrap();
```

### Using createNestApplication

```typescript
import { NestFactory } from '@nestjs/core';
import { Controller, Get, INestApplication } from '@nestjs/common';

@Controller()
class AppController {
  @Get()
  getHello(): string {
    return 'Hello World!';
  }
}

async function bootstrap() {
  const app = await NestFactory.createApplicationContext();
  
  // Register controller manually
  app.registerController(AppController);
  
  await app.init();
  await app.listen(3000);
}
bootstrap();
```

## 2. Standalone with Services

```typescript
import { NestFactory } from '@nestjs/core';
import { Controller, Get, Injectable, Module } from '@nestjs/common';

@Injectable()
class CatsService {
  private readonly cats = ['Kitty', 'Mittens'];
  
  findAll() {
    return this.cats;
  }
}

@Controller('cats')
class CatsController {
  constructor(private readonly catsService: CatsService) {}
  
  @Get()
  findAll() {
    return this.catsService.findAll();
  }
}

@Module({
  controllers: [CatsController],
  providers: [CatsService],
})
class CatsModule {}

async function bootstrap() {
  const app = await NestFactory.create(CatsModule);
  await app.listen(3000);
}
bootstrap();
```

## 3. Standalone with Imports

```typescript
import { NestFactory } from '@nestjs/core';
import { Controller, Get, Injectable, Module, Global } from '@nestjs/common';

@Global()
@Module({
  providers: [ConfigService],
  exports: [ConfigService],
})
class ConfigModule {
  static forRoot() {
    return {
      module: ConfigModule,
      providers: [
        {
          provide: 'CONFIG',
          useValue: { port: 3000 },
        },
      ],
    };
  }
}

@Injectable()
class ConfigService {
  get(key: string) {
    return process.env[key];
  }
}

@Controller()
class AppController {
  constructor(private readonly configService: ConfigService) {}
  
  @Get()
  getHello(): string {
    return 'Hello World!';
  }
}

@Module({
  controllers: [AppController],
})
class AppModule {}

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  await app.listen(3000);
}
bootstrap();
```

## 4. Standalone Microservice

```typescript
import { NestFactory } from '@nestjs/core';
import { Controller, Get, Module } from '@nestjs/common';
import { MicroserviceOptions, Transport } from '@nestjs/microservices';

@Controller()
class MicroserviceController {
  @Get()
  getHello(): string {
    return 'Hello from Microservice!';
  }
}

@Module({
  controllers: [MicroserviceController],
})
class MicroserviceModule {}

async function bootstrap() {
  const app = await NestFactory.createMicroservice<MicroserviceOptions>(
    MicroserviceModule,
    {
      transport: Transport.TCP,
      options: {
        host: '0.0.0.0',
        port: 3001,
      },
    },
  );
  await app.listen();
}
bootstrap();
```

## 5. Standalone with Middleware

```typescript
import { NestFactory } from '@nestjs/core';
import { Controller, Get, Module, NestMiddleware } from '@nestjs/common';
import { Request, Response } from 'express';

class LoggerMiddleware implements NestMiddleware {
  use(req: Request, res: Response, next: () => void) {
    console.log(`${req.method} ${req.url}`);
    next();
  }
}

@Controller()
class AppController {
  @Get()
  getHello(): string {
    return 'Hello World!';
  }
}

@Module({
  controllers: [AppController],
})
class AppModule {}

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.use(new LoggerMiddleware());
  await app.listen(3000);
}
bootstrap();
```

## 6. Standalone with Validation

```typescript
import { NestFactory } from '@nestjs/core';
import { Controller, Get, Post, Body, Module, ValidationPipe } from '@nestjs/common';
import { IsString, IsInt } from 'class-validator';

class CreateCatDto {
  @IsString()
  name: string;

  @IsInt()
  age: number;
}

@Controller('cats')
class CatsController {
  @Post()
  create(@Body() createCatDto: CreateCatDto) {
    return { message: 'Cat created', cat: createCatDto };
  }
}

@Module({
  controllers: [CatsController],
})
class CatsModule {}

async function bootstrap() {
  const app = await NestFactory.create(CatsModule);
  app.useGlobalPipes(new ValidationPipe());
  await app.listen(3000);
}
bootstrap();
```

## 7. Standalone with Swagger

```typescript
import { NestFactory } from '@nestjs/core';
import { Controller, Get, Module } from '@nestjs/common';
import { SwaggerModule, DocumentBuilder } from '@nestjs/swagger';

@Controller()
class AppController {
  @Get()
  getHello(): string {
    return 'Hello World!';
  }
}

@Module({
  controllers: [AppController],
})
class AppModule {}

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  
  const config = new DocumentBuilder()
    .setTitle('Standalone API')
    .setDescription('API Documentation')
    .setVersion('1.0')
    .build();
  
  const document = SwaggerModule.createDocument(app, config);
  SwaggerModule.setup('api', app, document);
  
  await app.listen(3000);
}
bootstrap();
```

## 8. Standalone with Database

```typescript
import { NestFactory } from '@nestjs/core';
import { Controller, Get, Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';

@Controller('cats')
class CatsController {
  @Get()
  findAll() {
    return [{ id: 1, name: 'Kitty' }];
  }
}

@Module({
  imports: [
    TypeOrmModule.forRoot({
      type: 'sqlite',
      database: ':memory:',
      entities: [],
      synchronize: true,
    }),
  ],
  controllers: [CatsController],
})
class CatsModule {}

async function bootstrap() {
  const app = await NestFactory.create(CatsModule);
  await app.listen(3000);
}
bootstrap();
```

## Best Practices

1. **Use modules when needed** - Standalone is for simple apps
2. **Keep it simple** - Don't overcomplicate standalone apps
3. **Use global modules** - For shared services
4. **Test thoroughly** - Standalone apps still need testing
5. **Consider scalability** - Use modules for larger apps
6. **Document your setup** - Clear documentation for standalone configs

## When to Use Standalone

- ✅ Small utilities and scripts
- ✅ Microservices
- ✅ Prototypes and demos
- ✅ When you want minimal overhead
- ❌ Large applications with many modules
- ❌ When you need complex dependency injection
- ❌ When you need module lifecycle hooks

## References

- [NestJS Standalone Applications](https://docs.nestjs.com/standalone-applications)
- [NestJS Microservices](https://docs.nestjs.com/microservices/basics)
