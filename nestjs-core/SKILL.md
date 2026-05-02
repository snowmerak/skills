---
name: nestjs-core
description: Core NestJS concepts including controllers, modules, routing, and basic application structure. Use when building NestJS applications, creating controllers, defining routes, or understanding NestJS architecture.
license: MIT
metadata:
  author: snowmerak
  version: "1.0"
  framework: nestjs
  category: core
---

# NestJS Core Skills

This skill covers the fundamental concepts of NestJS framework for building efficient, scalable server-side applications.

## Overview

NestJS is a progressive Node.js framework for building efficient, scalable server-side applications. It uses TypeScript and combines elements of OOP, FP, and FRP. Under the hood, NestJS uses Express (default) or Fastify as the HTTP server platform.

## Key Concepts

### 1. Modules

Modules are the foundation of NestJS architecture. Each application has at least one module (the root module).

```typescript
import { Module } from '@nestjs/common';
import { CatsController } from './cats.controller';
import { CatsService } from './cats.service';

@Module({
  controllers: [CatsController],
  providers: [CatsService],
})
export class AppModule {}
```

**Module metadata:**
- `controllers`: Controllers belonging to this module
- `providers`: Providers (services, repositories, factories) belonging to this module
- `imports`: Other modules that this module needs
- `exports`: Providers exported from this module (made available to importing modules)

### 2. Controllers

Controllers are responsible for handling incoming requests and returning responses to the client.

```typescript
import { Controller, Get, Post, Body, Param, Delete } from '@nestjs/common';

@Controller('cats')
export class CatsController {
  @Post()
  create(@Body() createCatDto: CreateCatDto) {
    return 'This action adds a new cat';
  }

  @Get()
  findAll() {
    return `This action returns all cats`;
  }

  @Get(':id')
  findOne(@Param('id') id: string) {
    return `This action returns a cat with ID: ${id}`;
  }

  @Delete(':id')
  remove(@Param('id') id: string) {
    return `This action removes a cat with ID: ${id}`;
  }
}
```

**HTTP Method Decorators:**
- `@Get()` - Handle GET requests
- `@Post()` - Handle POST requests
- `@Put()` - Handle PUT requests
- `@Delete()` - Handle DELETE requests
- `@Patch()` - Handle PATCH requests
- `@Options()` - Handle OPTIONS requests
- `@Head()` - Handle HEAD requests
- `@All()` - Handle all HTTP methods

**Request Parameter Decorators:**
- `@Request()` / `@Req()` - Access the request object
- `@Response()` / `@Res()` - Access the response object
- `@Next()` - Access the next middleware function
- `@Session()` - Access the session object
- `@Param(key?: string)` - Access route parameters
- `@Body(key?: string)` - Access request body
- `@Query(key?: string)` - Access query parameters
- `@Headers(name?: string)` - Access request headers
- `@Ip()` - Access the client IP

### 3. Providers (Services)

Providers are classes that can be injected with dependencies using constructor injection.

```typescript
import { Injectable } from '@nestjs/common';
import { CreateCatDto } from './dto/create-cat.dto';

@Injectable()
export class CatsService {
  private readonly cats: string[] = [];

  create(cat: CreateCatDto) {
    this.cats.push(cat.name);
  }

  findAll() {
    return this.cats;
  }
}
```

**Common Provider Decorators:**
- `@Injectable()` - Mark a class as a provider
- `@Injectable({ scope: Scope.REQUEST })` - Scoped providers
- `@Injectable({ transient: true })` - Transient providers

### 4. Application Entry Point

```typescript
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  await app.listen(process.env.PORT ?? 3000);
}
bootstrap();
```

**Platform Options:**
- Express (default): `NestFactory.create(AppModule)`
- Fastify: `NestFactory.create(AppModule, { adapter: new FastifyAdapter() })`

## Project Structure

A typical NestJS project structure:

```
src/
├── common/           # Shared utilities, guards, pipes, etc.
├── config/           # Configuration files
├── modules/          # Feature modules
│   ├── cats/
│   │   ├── cats.controller.ts
│   │   ├── cats.service.ts
│   │   ├── dto/
│   │   ├── interfaces/
│   │   └── cats.module.ts
│   └── users/
├── main.ts           # Application entry point
└── app.module.ts     # Root module
```

## Best Practices

1. **Keep controllers thin** - Delegate business logic to services
2. **Use DTOs** - Validate and serialize request/response data
3. **Module organization** - Group related functionality into feature modules
4. **Dependency injection** - Use constructor injection for dependencies
5. **Error handling** - Use exception filters for consistent error responses
6. **Environment variables** - Use ConfigModule for configuration management

## Routing

Route paths are determined by combining the controller prefix with the method decorator path:

```typescript
@Controller('cats')
export class CatsController {
  @Get()           // Maps to GET /cats
  @Get('breed')    // Maps to GET /cats/breed
  @Get(':id')      // Maps to GET /cats/:id
}
```

## Response Handling

**Standard approach (recommended):**
- Objects/arrays are automatically serialized to JSON
- Primitives are sent as-is
- Status code is 200 by default (201 for POST)

**Library-specific approach:**
- Use `@Res()` decorator to access native response object
- Set `passthrough: true` to use both approaches together

## Status Codes

Override default status codes with `@HttpCode()`:

```typescript
@Post()
@HttpCode(204)  // No Content
create() {
  return;
}
```

## Response Headers

Set custom response headers with `@Header()`:

```typescript
@Get()
@Header('Cache-Control', 'none')
findAll() {
  return [];
}
```

## Redirection

Redirect responses with `@Redirect()`:

```typescript
@Get('docs')
@Redirect('https://docs.nestjs.com', 302)
getDocs(@Query('version') version: string) {
  if (version === '5') {
    return { url: 'https://docs.nestjs.com/v5/' };
  }
}
```

## Route Wildcards

Support pattern-based routes:

```typescript
@Get('abcd/*')  // Matches abcd/, abcd/123, abcd/abc, etc.
findAll() {}
```

## References

- [NestJS Documentation](https://docs.nestjs.com)
- [NestJS GitHub](https://github.com/nestjs/nest)
- [NestJS Official Courses](https://courses.nestjs.com)
