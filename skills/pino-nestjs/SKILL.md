---
name: pino-nestjs
description: Pino integration with NestJS including custom logger factory, module configuration, request context propagation, and production-ready logging setup. Use when setting up Pino in NestJS applications, creating custom loggers, or configuring structured logging for microservices.
license: MIT
metadata:
  author: snowmerak
  version: '1.0'
  category: nestjs
  tags: [pino, logging, nestjs, structured-logging]
---

# Pino NestJS Integration

This skill covers Pino integration with NestJS using `nestjs-pino` and `pino-http`.

## SOP: Step-by-Step Procedures

### SOP 1: Installation & Basic Setup

```bash
npm install nestjs-pino pino-http
npm install -D pino-pretty  # For development only
```

**Basic Module Setup:**

```typescript
import { Module } from '@nestjs/common';
import { PinoLogger, PinoModule } from 'nestjs-pino';

@Module({
  imports: [PinoModule.forRoot()],
})
export class AppModule {}
```

### SOP 2: Configuration with ConfigService (Recommended)

```typescript
import { Module, Global } from '@nestjs/common';
import { ConfigService, ConfigModule } from '@nestjs/config';
import { PinoLogger, PinoModule } from 'nestjs-pino';
import pino from 'pino';

@Global()
@Module({
  imports: [
    ConfigModule.forRoot(),
    PinoModule.forRootAsync({
      inject: [ConfigService],
      useFactory: (configService: ConfigService) => ({
        pinoHttp: {
          logger: pino({
            level: configService.get<string>('LOG_LEVEL', 'info'),
            name: configService.get<string>('APP_NAME', 'nestjs-app'),
            transport: process.env.NODE_ENV !== 'production'
              ? { target: 'pino-pretty' }
              : undefined,
          }),
        },
      }),
    }),
  ],
  providers: [PinoLogger],
  exports: [PinoLogger],
})
export class LoggerModule {}
```

### SOP 3: Using PinoLogger in Services & Controllers

**In Services (via `@InjectPinoLogger`):**

```typescript
import { Injectable, LoggerService } from '@nestjs/common';
import { InjectPinoLogger } from 'nestjs-pino';

@Injectable()
export class UserService {
  constructor(
    @InjectPinoLogger(UserService.name)
    private readonly logger: LoggerService,
  ) {}

  async findAll() {
    this.logger.log('Fetching all users');
    return await this.userRepository.find();
  }

  async findOne(id: number) {
    const user = await this.userRepository.findOne(id);
    if (!user) {
      this.logger.warn({ userId: id }, 'User not found');
      throw new NotFoundException('User not found');
    }
    return user;
  }

  async create(data: CreateUserDto) {
    try {
      const user = await this.userRepository.create(data);
      this.logger.info({ userId: user.id }, 'User created successfully');
      return user;
    } catch (error) {
      this.logger.error(error, `Failed to create user ${data.email}`);
      throw error;
    }
  }
}
```

**In Controllers:**

```typescript
import { Controller, Get, Post, Body, Param, Delete } from '@nestjs/common';
import { InjectPinoLogger, LoggerService } from 'nestjs-pino';

@Controller('users')
export class UserController {
  constructor(
    private readonly userService: UserService,
    @InjectPinoLogger(UserController.name)
    private readonly logger: LoggerService,
  ) {}

  @Get()
  async findAll() {
    this.logger.info({ action: 'GET /users' }, 'Fetching all users');
    return await this.userService.findAll();
  }

  @Post()
  async create(@Body() data: CreateUserDto) {
    this.logger.info({ email: data.email, action: 'POST /users' }, 'Creating user');
    return await this.userService.create(data);
  }
}
```

### SOP 4: Request Context & Logging Interceptor

**Create Interceptor:**

```typescript
import { Injectable, NestInterceptor, ExecutionContext, CallHandler } from '@nestjs/common';
import { Observable } from 'rxjs';
import { tap } from 'rxjs/operators';
import { InjectPinoLogger, LoggerService } from 'nestjs-pino';

@Injectable()
export class LoggingInterceptor implements NestInterceptor {
  constructor(
    @InjectPinoLogger(LoggingInterceptor.name)
    private readonly logger: LoggerService,
  ) {}

  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const request = context.switchToHttp().getRequest();
    const method = request.method;
    const url = request.url;
    const now = Date.now();

    const logger = this.logger.child({ requestId: request.id, method, url });
    logger.info({ action: 'request-start' }, `Incoming ${method} request`);

    return next.handle().pipe(
      tap(() => {
        const responseTime = Date.now() - now;
        logger.info({ responseTimeMs: responseTime, action: 'request-end' }, 'Request completed');
      }),
    );
  }
}
```

**Register Globally:**

```typescript
import { APP_INTERCEPTOR } from '@nestjs/core';

@Module({
  providers: [
    { provide: APP_INTERCEPTOR, useClass: LoggingInterceptor },
  ],
})
export class AppModule {}
```

### SOP 5: Exception Handling with Pino

**Custom Exception Filter:**

```typescript
import { ExceptionFilter, Catch, ArgumentsHost, HttpException, HttpStatus } from '@nestjs/common';
import { Request, Response } from 'express';
import { InjectPinoLogger, LoggerService } from 'nestjs-pino';

@Catch()
export class AllExceptionsFilter implements ExceptionFilter {
  constructor(
    @InjectPinoLogger(AllExceptionsFilter.name)
    private readonly logger: LoggerService,
  ) {}

  catch(exception: unknown, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();
    const request = ctx.getRequest<Request>();

    const status = exception instanceof HttpException
      ? exception.getStatus() : HttpStatus.INTERNAL_SERVER_ERROR;

    this.logger.error(
      {
        requestId: request.id,
        method: request.method,
        url: request.url,
        statusCode: status,
        stack: exception instanceof Error ? exception.stack : undefined,
      },
      `Exception occurred: ${status}`,
    );

    response.status(status).json({
      statusCode: status,
      timestamp: new Date().toISOString(),
      path: request.url,
    });
  }
}
```

**Register in bootstrap:**

```typescript
import { NestFactory } from '@nestjs/core';
import { AllExceptionsFilter } from './filters/all-exceptions.filter';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.useGlobalFilters(new AllExceptionsFilter());
  await app.listen(3000);
}
bootstrap();
```

### SOP 6: Production Configuration

```typescript
useFactory: (configService: ConfigService) => {
  const isProduction = configService.get<string>('NODE_ENV') === 'production';

  return {
    pinoHttp: {
      logger: pino({
        level: configService.get<string>('LOG_LEVEL', isProduction ? 'info' : 'debug'),
        name: configService.get<string>('APP_NAME', 'nestjs-app'),

        // Production: file transport, Development: pretty print
        transport: isProduction
          ? { target: 'pino/file', options: { destination: './logs/app.log', mkdir: true } }
          : { target: 'pino-pretty', options: { colorize: true, translateTime: 'SYS:yyyy-mm-dd HH:MM:ss.l' } },

        // Redact sensitive data in production
        redact: isProduction ? ['password', 'token', 'apiSecret'] : [],
      }),
    },
  };
},
```

**Docker (log to stdout, no base bindings):**

```typescript
const isDocker = process.env.DOCKER_CONTAINER === 'true';
return {
  pinoHttp: {
    logger: pino({
      level: configService.get<string>('LOG_LEVEL', 'info'),
      base: isDocker ? null : undefined,  // Orchestrator provides pid/hostname
      transport: !isDocker && process.env.NODE_ENV !== 'production'
        ? { target: 'pino-pretty' }
        : undefined,
    }),
  },
};
```

## Tool Integration

| Task | Tool | Usage |
|------|------|-------|
| Verify Pino installation | `run_command` | `npm list nestjs-pino pino-http` |
| Inspect logger config | `read_file` + `search_files` | Find `PinoModule.forRoot` usage in AppModule |
| Add logging to service | `edit_file` | Inject `@InjectPinoLogger(ServiceName.name)` into constructor |
| Create interceptor | `write_file` → `edit_file` | Create `logging.interceptor.ts`, register via `APP_INTERCEPTOR` |
| Verify exception filter | `search_files` | Find `useGlobalFilters` in bootstrap file |

## Anti-Patterns & Guardrails

❌ **Never** use string interpolation for structured data:
```typescript
// BAD
logger.log(`User ${userId} with IP ${ip} logged in`);
// GOOD
logger.info({ userId, ip }, 'User logged in');
```

❌ **Never** pass Error as a property — always pass as first argument:
```typescript
// BAD - loses stack trace
logger.error({ error: err.message });
// GOOD - Pino serializes properly
logger.error(err, 'Operation failed');
```

❌ **Never** log full objects in production (sensitive/large data):
```typescript
// BAD
logger.info({ user: fullUserObject }, 'User action');
// GOOD
logger.info({ userId: 123 }, 'User action');
```

⚠️ **Always** use `@InjectPinoLogger(ClassName.name)` — the class name becomes a binding in every log line for easy filtering.

⚠️ **Never** use `PinoModule.forRoot()` with async config — always use `forRootAsync` with `useFactory`.

## Best Practices

1. Use child loggers (`logger.child({ requestId })`) for request-scoped context
2. Set appropriate level per environment: `production → info`, `development → debug`
3. Always pass Error objects directly as first argument to `.error()` / `.fatal()`
4. Redact sensitive fields (`password`, `token`) in production via `redact` option
5. Use `APP_INTERCEPTOR` for global request/response timing logs

## References

- [nestjs-pino Documentation](https://github.com/iamolegga/nestjs-pino)
- [Pino Web Logging Guide](https://getpino.io/#/docs/web)

---

**Skill successfully created:** `skills/pino-nestjs/SKILL.md`

This skill is now ready. Please renew the Skill Index before using it.
