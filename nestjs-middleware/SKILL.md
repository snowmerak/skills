---
name: nestjs-middleware
description: NestJS middleware, guards, interceptors, exception filters, pipes, and custom decorators. Use when implementing request processing pipeline, validation, error handling, or cross-cutting concerns in NestJS.
license: MIT
metadata:
  author: snowmerak
  version: "1.0"
  framework: nestjs
  category: overview
---

# NestJS Overview Skills

This skill covers the core building blocks of NestJS applications: middleware, guards, interceptors, exception filters, pipes, and custom decorators.

## Request Processing Pipeline

NestJS processes requests through a well-defined pipeline:

```
Request → Middleware → Guards → Interceptors → Pipes → Controller → Interceptors → Exception Filters → Response
```

## 1. Middleware

Middleware functions execute before the route handler.

### Creating Middleware

```typescript
import { Injectable, NestMiddleware } from '@nestjs/common';
import { Request, Response, NextFunction } from 'express';

@Injectable()
export class LoggerMiddleware implements NestMiddleware {
  use(req: Request, res: Response, next: NextFunction) {
    console.log(`Request... ${req.method} ${req.url}`);
    next();
  }
}
```

### Applying Middleware

```typescript
import { Module, NestModule, MiddlewareConsumer } from '@nestjs/common';

@Module({})
export class AppModule implements NestModule {
  configure(consumer: MiddlewareConsumer) {
    // Apply to all routes
    consumer.apply(LoggerMiddleware).forRoutes('*');
    
    // Apply to specific routes
    consumer.apply(LoggerMiddleware).forRoutes('cats');
    
    // Apply with conditions
    consumer.apply(LoggerMiddleware).forRoutes({ path: 'cats', method: RequestMethod.GET });
  }
}
```

### Middleware Patterns

**CORS Middleware**
```typescript
app.enableCors({
  origin: 'http://example.com',
  methods: 'GET,HEAD,PUT,PATCH,POST,DELETE',
});
```

**Body Parser**
```typescript
app.use(express.json());
app.use(express.urlencoded({ extended: true }));
```

## 2. Guards

Guards determine whether a request will be handled by the route handler.

### Creating a Guard

```typescript
import { Injectable, CanActivate, ExecutionContext } from '@nestjs/common';
import { Observable } from 'rxjs';

@Injectable()
export class AuthGuard implements CanActivate {
  canActivate(
    context: ExecutionContext,
  ): boolean | Promise<boolean> | Observable<boolean> {
    const request = context.switchToHttp().getRequest();
    return validateRequest(request);
  }
}
```

### Applying Guards

```typescript
// At controller level
@Guards(AuthGuard)
@Controller('cats')
export class CatsController {}

// At route level
@Get()
@Guards(AuthGuard)
findAll() {}

// Globally
app.useGlobalGuards(new AuthGuard());
```

### Guard with Dependencies

```typescript
@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private rolesService: RolesService) {}

  canActivate(context: ExecutionContext): boolean {
    const request = context.switchToHttp().getRequest();
    const user = request.user;
    const roles = this.rolesService.findAll();
    return matchRoles(user.role, roles);
  }
}
```

## 3. Interceptors

Interceptors execute before and after the route handler.

### Creating an Interceptor

```typescript
import { Injectable, NestInterceptor, ExecutionContext, CallHandler } from '@nestjs/common';
import { Observable } from 'rxjs';
import { tap } from 'rxjs/operators';

@Injectable()
export class LoggingInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    console.log('Before...');
    
    const now = Date.now();
    return next
      .handle()
      .pipe(
        tap(() => {
          console.log(`After... ${Date.now() - now}ms`);
        }),
      );
  }
}
```

### Response Transformation

```typescript
@Injectable()
export class TransformInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    return next.handle().pipe(
      map(data => ({
        statusCode: 200,
        timestamp: new Date().toISOString(),
        data,
      })),
    );
  }
}
```

### Applying Interceptors

```typescript
// At controller level
@UseInterceptors(LoggingInterceptor)
@Controller('cats')
export class CatsController {}

// At route level
@Get()
@UseInterceptors(LoggingInterceptor)
findAll() {}

// Globally
app.useGlobalInterceptors(new LoggingInterceptor());
```

## 4. Exception Filters

Exception filters catch and handle exceptions thrown by route handlers.

### Creating an Exception Filter

```typescript
import { ExceptionFilter, Catch, ArgumentsHost, HttpException } from '@nestjs/common';
import { Request, Response } from 'express';

@Catch(HttpException)
export class HttpExceptionFilter implements ExceptionFilter {
  catch(exception: HttpException, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();
    const request = ctx.getRequest<Request>();
    const status = exception.getStatus();

    response.status(status).json({
      statusCode: status,
      timestamp: new Date().toISOString(),
      path: request.url,
    });
  }
}
```

### Applying Exception Filters

```typescript
// At controller level
@UseFilters(HttpExceptionFilter)
@Controller('cats')
export class CatsController {}

// At route level
@Get()
@UseFilters(HttpExceptionFilter)
findAll() {}

// Globally
app.useGlobalFilters(new HttpExceptionFilter());
```

## 5. Pipes

Pipes transform or validate input data.

### Built-in Pipes

**ValidationPipe**
```typescript
import { ValidationPipe } from '@nestjs/common';

app.useGlobalPipes(new ValidationPipe());

// Or with options
app.useGlobalPipes(new ValidationPipe({
  whitelist: true,
  forbidNonWhitelisted: true,
  transform: true,
}));
```

**ParseIntPipe**
```typescript
@Get(':id')
findOne(@Param('id', ParseIntPipe) id: number) {
  return this.catsService.findOne(id);
}
```

**DefaultValuePipe**
```typescript
@Get()
findAll(@Query('page', new DefaultValuePipe(1), ParseIntPipe) page: number) {
  return this.catsService.findAll(page);
}
```

### Creating Custom Pipes

```typescript
import { PipeTransform, Injectable, ArgumentMetadata, BadRequestException } from '@nestjs/common';

@Injectable()
export class CustomPipe implements PipeTransform<any> {
  transform(value: any, metadata: ArgumentMetadata) {
    if (typeof value !== 'string') {
      throw new BadRequestException('Validation failed');
    }
    return value;
  }
}
```

### Applying Pipes

```typescript
// At controller level
@UsePipes(new ValidationPipe())
@Controller('cats')
export class CatsController {}

// At route level
@Post()
create(@Body(new ValidationPipe()) createCatDto: CreateCatDto) {}

// At parameter level
@Get(':id')
findOne(@Param('id', new ParseIntPipe()) id: number) {}
```

## 6. Custom Decorators

### Simple Decorators

```typescript
import { createParamDecorator, ExecutionContext } from '@nestjs/common';

export const User = createParamDecorator(
  (data: unknown, ctx: ExecutionContext) => {
    const request = ctx.switchToHttp().getRequest();
    return request.user;
  },
);

// Usage
@Get()
findAll(@User() user: User) {
  return this.catsService.findAll(user);
}
```

### Data-Driven Decorators

```typescript
export const Roles = (...roles: string[]) => SetMetadata('roles', roles);

// Usage
@Roles('admin')
@Get()
findAll() {}
```

## Request Object Decorators

| Decorator | Description |
|-----------|-------------|
| `@Request()` / `@Req()` | Access the request object |
| `@Response()` / `@Res()` | Access the response object |
| `@Next()` | Access the next middleware function |
| `@Session()` | Access the session object |
| `@Param(key?: string)` | Access route parameters |
| `@Body(key?: string)` | Access request body |
| `@Query(key?: string)` | Access query parameters |
| `@Headers(name?: string)` | Access request headers |
| `@Ip()` | Access the client IP |
| `@HostParam()` | Access host parameters |

## Best Practices

1. **Middleware** - Use for cross-cutting concerns like logging, CORS
2. **Guards** - Use for authentication and authorization
3. **Interceptors** - Use for response transformation, logging, caching
4. **Exception Filters** - Use for consistent error handling
5. **Pipes** - Use for validation and transformation
6. **Custom Decorators** - Use to encapsulate complex logic

## References

- [NestJS Middleware](https://docs.nestjs.com/middleware)
- [NestJS Guards](https://docs.nestjs.com/guards)
- [NestJS Interceptors](https://docs.nestjs.com/interceptors)
- [NestJS Exception Filters](https://docs.nestjs.com/exception-filters)
- [NestJS Pipes](https://docs.nestjs.com/pipes)
- [NestJS Custom Decorators](https://docs.nestjs.com/custom-decorators)
