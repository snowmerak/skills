---
name: pino-nestjs
description: Pino integration with NestJS including custom logger factory, module configuration, request context propagation, and production-ready logging setup. Use when setting up Pino in NestJS applications, creating custom loggers, or configuring structured logging for microservices.
license: MIT
metadata:
  author: snowmerak
  version: "1.0"
  framework: pino
  category: nestjs
---

# Pino NestJS Integration Skills

This skill covers comprehensive Pino integration with NestJS applications using `nestjs-pino` and `pino-http`.

## Installation

### Basic Setup

```bash
npm install nestjs-pino pino-http
npm install -D pino-pretty  # For development
```

### With TypeScript Decorators (Optional)

```bash
npm install @nestjs/common @nestjs/core reflect-metadata
```

## Basic Configuration

### Simple Logger Factory

```typescript
import { Module } from '@nestjs/common';
import { PinoLogger, PinoModule } from 'nestjs-pino';

@Module({
  imports: [
    PinoModule.forRoot(),
  ],
})
export class AppModule {}
```

### With Custom Options

```typescript
import { LoggerService } from '@nestjs/common';
import { PinoLogger, PinoModule, RootPinoLogger } from 'nestjs-pino';
import pino from 'pino';

const loggerOptions: pino.LoggerOptions = {
  level: 'info',
  transport: process.env.NODE_ENV !== 'production'
    ? { target: 'pino-pretty' }
    : undefined,
};

@Module({
  imports: [
    PinoModule.forRoot({
      pinoHttp: {
        logger: pino(loggerOptions),
      },
    }),
  ],
})
export class AppModule {}
```

### Using Config Service (Recommended)

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

## Using PinoLogger in Services and Controllers

### Basic Usage with Dependency Injection

```typescript
import { Injectable, LoggerService } from '@nestjs/common';
import { PinoLogger } from 'nestjs-pino';

@Injectable()
export class UserService {
  constructor(private readonly pinoLogger: PinoLogger) {}

  async findAll() {
    this.pinoLogger.info('Fetching all users');
    
    const users = await this.userRepository.find();
    
    this.pinoLogger.info({ count: users.length }, 'Users fetched successfully');
    
    return users;
  }

  async findOne(id: number) {
    this.pinoLogger.info({ userId: id }, 'Fetching user by ID');
    
    const user = await this.userRepository.findOne(id);
    
    if (!user) {
      this.pinoLogger.warn({ userId: id }, 'User not found');
      throw new NotFoundException('User not found');
  }

  async create(data: CreateUserDto) {
    this.pinoLogger.info({ email: data.email }, 'Creating new user');
    
    const user = await this.userRepository.create(data);
    
    this.pinoLogger.info({ userId: user.id }, 'User created successfully');
    
    return user;
  }

  async update(id: number, data: UpdateUserDto) {
    this.pinoLogger.info({ userId: id }, 'Updating user');
    
    try {
      const user = await this.userRepository.update(id, data);
      this.pinoLogger.info({ userId: id }, 'User updated successfully');
      return user;
    } catch (error) {
      this.pinoLogger.error(error, `Failed to update user ${id}`);
      throw error;
    }
  }

  async remove(id: number) {
    this.pinoLogger.info({ userId: id }, 'Deleting user');
    
    await this.userRepository.remove(id);
    
    this.pinoLogger.info({ userId: id }, 'User deleted successfully');
  }
}
```

### Using LoggerService Interface (NestJS Standard)

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
    this.logger.debug(`Fetching user with ID: ${id}`);
    
    const user = await this.userRepository.findOne(id);
    
    if (!user) {
      this.logger.warn(`User with ID ${id} not found`);
      throw new NotFoundException('User not found');
    }
    
    return user;
  }

  async create(data: CreateUserDto) {
    this.logger.log(`Creating new user: ${data.email}`);
    
    const user = await this.userRepository.create(data);
    
    this.logger.log(`User created with ID: ${user.id}`);
    
    return user;
  }

  async update(id: number, data: UpdateUserDto) {
    this.logger.log(`Updating user with ID: ${id}`);
    
    try {
      const user = await this.userRepository.update(id, data);
      this.logger.log(`User updated successfully`);
      return user;
    } catch (error) {
      this.logger.error(error, `Failed to update user ${id}`);
      throw error;
    }
  }

  async remove(id: number) {
    this.logger.debug(`Removing user with ID: ${id}`);
    
    await this.userRepository.remove(id);
    
    this.logger.log(`User removed successfully`);
  }
}
```

### Controller Usage

```typescript
import { Controller, Get, Post, Body, Param, Delete, UseGuards } from '@nestjs/common';
import { PinoLogger } from 'nestjs-pino';
import { UserService } from './user.service';
import { JwtAuthGuard } from '../auth/guards/jwt-auth.guard';

@Controller('users')
@UseGuards(JwtAuthGuard)
export class UserController {
  constructor(
    private readonly userService: UserService,
    private readonly pinoLogger: PinoLogger,
  ) {}

  @Get()
  async findAll() {
    this.pinoLogger.info({ action: 'GET /users' }, 'Fetching all users');
    
    const users = await this.userService.findAll();
    
    this.pinoLogger.info({ count: users.length }, 'Users fetched successfully');
    
    return users;
  }

  @Get(':id')
  async findOne(@Param('id') id: string) {
    const userId = parseInt(id, 10);
    
    this.pinoLogger.info({ userId, action: 'GET /users/:id' }, 'Fetching user by ID');
    
    try {
      const user = await this.userService.findOne(userId);
      
      this.pinoLogger.info({ userId }, 'User fetched successfully');
      
      return user;
    } catch (error) {
      this.pinoLogger.error(error, `Failed to fetch user ${id}`);
      throw error;
    }
  }

  @Post()
  async create(@Body() createUserDto: CreateUserDto) {
    this.pinoLogger.info({ email: createUserDto.email, action: 'POST /users' }, 'Creating new user');
    
    try {
      const user = await this.userService.create(createUserDto);
      
      this.pinoLogger.info({ userId: user.id }, 'User created successfully');
      
      return user;
    } catch (error) {
      this.pinoLogger.error(error, `Failed to create user ${createUserDto.email}`);
      throw error;
    }
  }

  @Delete(':id')
  async remove(@Param('id') id: string) {
    const userId = parseInt(id, 10);
    
    this.pinoLogger.info({ userId, action: 'DELETE /users/:id' }, 'Deleting user');
    
    try {
      await this.userService.remove(userId);
      
      this.pinoLogger.info({ userId }, 'User deleted successfully');
      
      return { message: 'User deleted successfully' };
    } catch (error) {
      this.pinoLogger.error(error, `Failed to delete user ${id}`);
      throw error;
    }
  }
}
```

## Request Context Propagation

### Using HttpAdapterHost for Request Logger

```typescript
import { Injectable, Inject, Optional } from '@nestjs/common';
import { REQUEST } from '@nestjs/core';
import { PinoLogger } from 'nestjs-pino';
import type { Request } from 'express';

@Injectable()
export class RequestContextService {
  constructor(
    @Inject(REQUEST) private readonly request: Request,
    private readonly pinoLogger: PinoLogger,
  ) {}

  getRequestLogger() {
    // Get the child logger with request context
    return this.pinoLogger.child({ 
      requestId: this.request.id,
      userId: (this.request as any).user?.id,
    });
  }

  getRequestId(): string | undefined {
    return this.request.id;
  }

  getCurrentUserId(): number | undefined {
    return (this.request as any).user?.id;
  }
}
```

### Custom Interceptor for Request Logging

```typescript
import { Injectable, NestInterceptor, ExecutionContext, CallHandler } from '@nestjs/common';
import { Observable } from 'rxjs';
import { tap } from 'rxjs/operators';
import { PinoLogger } from 'nestjs-pino';

@Injectable()
export class LoggingInterceptor implements NestInterceptor {
  constructor(private readonly pinoLogger: PinoLogger) {}

  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const request = context.switchToHttp().getRequest();
    const method = request.method;
    const url = request.url;
    const now = Date.now();
    
    // Create child logger with request context
    const logger = this.pinoLogger.child({ 
      requestId: request.id,
      method,
      url,
    });

    logger.info({ action: 'request-start' }, `Incoming ${method} request`);

    return next.handle().pipe(
      tap((response) => {
        const responseTime = Date.now() - now;
        
        logger.info({ 
          statusCode: response?.statusCode || 200,
          responseTimeMs: responseTime,
          action: 'request-end',
        }, `Request completed`);
      }),
    );
  }
}
```

### Register Interceptor Globally

```typescript
import { Module } from '@nestjs/common';
import { APP_INTERCEPTOR } from '@nestjs/core';
import { LoggingInterceptor } from './logging.interceptor';
import { PinoLogger, PinoModule } from 'nestjs-pino';

@Module({
  providers: [
    {
      provide: APP_INTERCEPTOR,
      useFactory: (pinoLogger: PinoLogger) => new LoggingInterceptor(pinoLogger),
      inject: [PinoLogger],
    },
  ],
})
export class AppModule {}
```

## Exception Handling with Pino

### Custom Exception Filter

```typescript
import { 
  ExceptionFilter, 
  Catch, 
  ArgumentsHost, 
  HttpException, 
  HttpStatus 
} from '@nestjs/common';
import { Request, Response } from 'express';
import { PinoLogger } from 'nestjs-pino';

@Catch()
export class AllExceptionsFilter implements ExceptionFilter {
  constructor(private readonly pinoLogger: PinoLogger) {}

  catch(exception: unknown, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();
    const request = ctx.getRequest<Request>();
    
    const status = exception instanceof HttpException 
      ? exception.getStatus() 
      : HttpStatus.INTERNAL_SERVER_ERROR;

    const message = exception instanceof HttpException
      ? exception.getResponse()
      : 'Internal server error';

    // Log the exception with full context
    this.pinoLogger.error(
      {
        requestId: request.id,
        method: request.method,
        url: request.url,
        statusCode: status,
        message: message instanceof Object ? JSON.stringify(message) : message,
        stack: exception instanceof Error ? exception.stack : undefined,
      },
      `Exception occurred: ${status}`,
    );

    response.status(status).json({
      statusCode: status,
      timestamp: new Date().toISOString(),
      path: request.url,
      message: message instanceof Object 
        ? (message as any).message || 'Internal server error' 
        : message,
    });
  }
}
```

### Register Exception Filter Globally

```typescript
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';
import { AllExceptionsFilter } from './filters/all-exceptions.filter';
import { PinoLogger } from 'nestjs-pino';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  
  // Register exception filter with PinoLogger injection
  const pinoLogger = app.get(PinoLogger);
  app.useGlobalFilters(new AllExceptionsFilter(pinoLogger));
  
  await app.listen(3000);
}
bootstrap();
```

## Production Configuration

### Environment-Based Setup

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
      useFactory: (configService: ConfigService) => {
        const isProduction = configService.get<string>('NODE_ENV') === 'production';
        
        return {
          pinoHttp: {
            logger: pino({
              level: configService.get<string>('LOG_LEVEL', isProduction ? 'info' : 'debug'),
              name: configService.get<string>('APP_NAME', 'nestjs-app'),
              
              // Production: file transport, Development: pretty print
              transport: isProduction
                ? {
                    target: 'pino/file',
                    options: { 
                      destination: './logs/app.log',
                      mkdir: true,  // Auto-create log directory
                    },
                  }
                : {
                    target: 'pino-pretty',
                    options: {
                      colorize: true,
                      translateTime: 'SYS:yyyy-mm-dd HH:MM:ss.l',
                      ignore: 'pid,hostname',
                    },
                  },
              
              // Redact sensitive data in production
              redact: isProduction 
                ? ['password', 'token', 'apiSecret'] 
                : [],
            }),
          },
        };
      },
    }),
  ],
  providers: [PinoLogger],
  exports: [PinoLogger],
})
export class LoggerModule {}
```

### Docker Configuration

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
      useFactory: (configService: ConfigService) => {
        // In Docker, log to stdout for container orchestration
        const isDocker = process.env.DOCKER_CONTAINER === 'true';
        
        return {
          pinoHttp: {
            logger: pino({
              level: configService.get<string>('LOG_LEVEL', 'info'),
              
              // No base bindings in containers (orchestrator provides this)
              base: isDocker ? null : undefined,
              
              // Pretty print only if not in Docker/production
              transport: !isDocker && process.env.NODE_ENV !== 'production'
                ? { target: 'pino-pretty' }
                : undefined,
            }),
          },
        };
      },
    }),
  ],
  providers: [PinoLogger],
  exports: [PinoLogger],
})
export class LoggerModule {}
```

## Microservices Logging

### Separate Loggers for Different Modules

```typescript
import { Module } from '@nestjs/common';
import { PinoLogger, PinoModule } from 'nestjs-pino';
import pino from 'pino';

@Module({
  imports: [
    PinoModule.forRootAsync({
      useFactory: () => ({
        pinoHttp: {
          logger: pino({ name: 'auth-service' }),
        },
      }),
    }),
  ],
})
export class AuthModule {}

// In UserService module
@Module({
  imports: [
    PinoModule.forRootAsync({
      useFactory: () => ({
        pinoHttp: {
          logger: pino({ name: 'user-service' }),
        },
      }),
    }),
  ],
})
export class UserService {}
```

### Request Tracing Across Services

```typescript
import { Injectable } from '@nestjs/common';
import { PinoLogger } from 'nestjs-pino';
import type { ClientProxy } from '@nestjs/microservices';

@Injectable()
export class OrderService {
  constructor(
    private readonly pinoLogger: PinoLogger,
    private readonly client: ClientProxy,
  ) {}

  async createOrder(orderData: CreateOrderDto) {
    const logger = this.pinoLogger.child({ 
      action: 'create-order',
      orderId: orderData.id,
    });

    logger.info('Starting order creation');

    try {
      // Send to payment service with correlation ID
      await this.client.send('payment.process', {
        ...orderData,
        correlationId: logger.bindings().requestId,
      }).toPromise();

      logger.info({ orderId: orderData.id }, 'Order created successfully');

      return { success: true };
    } catch (error) {
      logger.error(error, `Failed to create order ${orderData.id}`);
      throw error;
    }
  }
}
```

## Best Practices

### 1. Use Child Loggers for Context

```typescript
// GOOD: Use child loggers with context
const requestLogger = this.pinoLogger.child({ 
  requestId: req.id,
  userId: user.id,
});
requestLogger.info('Processing payment');

// BAD: Manually add context to every log
this.pinoLogger.info({ requestId: id, userId: uid }, 'message 1');
this.pinoLogger.info({ requestId: id, userId: uid }, 'message 2');
```

### 2. Use Structured Logging

```typescript
// GOOD: Log data as structured properties
logger.info({ userId: 123, action: 'login', ip: '192.168.1.1' }, 'User logged in');

// BAD: String interpolation for data (harder to query/analyze)
logger.log(`User ${userId} with IP ${ip} logged in`);
```

### 3. Set Appropriate Level per Environment

```typescript
const level = 
  process.env.NODE_ENV === 'production' ? 'info' : 
  process.env.NODE_ENV === 'staging' ? 'debug' : 'trace';

const logger = pino({ level });
```

### 4. Use Child Loggers for Request Context

```typescript
// In HTTP middleware
const requestLogger = logger.child({ 
  requestId: req.id, 
  method: req.method, 
  path: req.path 
});
```

### 5. Avoid Logging Large Objects in Production

```typescript
// GOOD
logger.info({ userId: 123 }, 'User action');

// BAD - may contain sensitive/large data
logger.info({ user: fullUserObjectWithAllFields }, 'User action');
```

## References

- [nestjs-pino Documentation](https://github.com/iamolegga/nestjs-pino)
- [Pino NestJS Integration Guide](https://getpino.io/#/docs/web)
