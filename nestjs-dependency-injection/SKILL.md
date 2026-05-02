---
name: nestjs-dependency-injection
description: NestJS dependency injection system, providers, injection tokens, scoped providers, and advanced DI patterns. Use when working with dependency injection, creating services, or managing provider lifecycles in NestJS.
license: MIT
metadata:
  author: snowmerak
  version: "1.0"
  framework: nestjs
  category: fundamentals
---

# NestJS Dependency Injection Skills

This skill covers NestJS's powerful dependency injection (DI) system, providers, and advanced DI patterns.

## Overview

NestJS uses a runtime dependency injection system inspired by Angular's DI. It maintains a hierarchical dependency injection tree that mirrors the module structure.

## Providers

Providers are classes, values, factories, or tokens that can be injected into dependencies.

### Injectable Decorator

```typescript
import { Injectable } from '@nestjs/common';

@Injectable()
export class CatsService {
  private readonly cats: string[] = [];

  create(cat: string) {
    this.cats.push(cat);
  }

  findAll() {
    return this.cats;
  }
}
```

### Provider Types

**1. Class Providers (Default)**
```typescript
@Module({
  providers: [CatsService],
})
export class AppModule {}
```

**2. Value Providers**
```typescript
@Module({
  providers: [
    {
      provide: 'CONFIG',
      useValue: { port: 3000 },
    },
  ],
})
export class AppModule {}
```

**3. Factory Providers**
```typescript
@Module({
  providers: [
    {
      provide: 'CONFIG',
      useFactory: () => {
        return { port: process.env.PORT };
      },
    },
  ],
})
export class AppModule {}
```

**4. Alias Providers**
```typescript
@Module({
  providers: [
    CatsService,
    {
      provide: 'ALIAS',
      useExisting: CatsService,
    },
  ],
})
export class AppModule {}
```

## Injection Tokens

### String Tokens

```typescript
const CONFIG_TOKEN = 'CONFIG';

@Module({
  providers: [
    {
      provide: CONFIG_TOKEN,
      useValue: { port: 3000 },
    },
  ],
})
export class AppModule {}

// Injection
constructor(@Inject('CONFIG') private config: Config) {}
```

### Symbol Tokens

```typescript
export const CONFIG_TOKEN = Symbol('CONFIG');

@Module({
  providers: [
    {
      provide: CONFIG_TOKEN,
      useValue: { port: 3000 },
    },
  ],
})
export class AppModule {}

// Injection
constructor(@Inject(CONFIG_TOKEN) private config: Config) {}
```

### Class Tokens

```typescript
// Direct injection by class type
constructor(private catsService: CatsService) {}
```

## Scoped Providers

### Per-Request Scope (Transient - Default)

```typescript
@Injectable({ scope: Scope.REQUEST })
export class RequestService {
  // A new instance is created for each request
}
```

### Singleton Scope

```typescript
@Injectable({ scope: Scope.DEFAULT })
export class SingletonService {
  // One instance per module (default behavior)
}
```

### Transient Scope

```typescript
@Injectable({ scope: Scope.TRANSIENT })
export class TransientService {
  // New instance created each time it's injected
}
```

## Module-Level DI

### Hierarchical DI Tree

```typescript
// Root Module
@Module({
  imports: [CatsModule],
})
export class AppModule {}

// Cats Module
@Module({
  providers: [CatsService],
  exports: [CatsService],
})
export class CatsModule {}
```

### Exporting Providers

```typescript
@Module({
  providers: [CatsService, CatsRepository],
  exports: [CatsService], // Only CatsService is available to importing modules
})
export class CatsModule {}
```

### Global Modules

```typescript
import { Module, Global } from '@nestjs/common';

@Global()
@Module({
  providers: [ConfigService],
  exports: [ConfigService],
})
export class ConfigModule {}

// Now ConfigService is available in all modules without importing
```

## Advanced DI Patterns

### Dynamic Modules

```typescript
import { Module, DynamicModule } from '@nestjs/common';

@Module({})
export class DatabaseModule {
  static forRoot(options: DatabaseOptions): DynamicModule {
    return {
      module: DatabaseModule,
      providers: [
        {
          provide: 'DATABASE_OPTIONS',
          useValue: options,
        },
      ],
    };
  }
}

// Usage
@Module({
  imports: [DatabaseModule.forRoot({ host: 'localhost' })],
})
export class AppModule {}
```

### Conditional Providers

```typescript
@Module({
  providers: [
    {
      provide: 'API_SERVICE',
      useFactory: (config: ConfigService) => {
        return config.get('API_USE_MOCK') ? new MockApiService() : new RealApiService();
      },
      inject: [ConfigService],
    },
  ],
})
export class AppModule {}
```

### Async Providers

```typescript
@Module({
  providers: [
    {
      provide: 'ASYNC_CONFIG',
      useFactory: async () => {
        const config = await loadConfig();
        return config;
      },
    },
  ],
})
export class AppModule {}
```

### Custom Injectors

```typescript
import { Inject } from '@nestjs/common';

export const InjectLogger = () => Inject('LOGGER');

class MyClass {
  constructor(@InjectLogger() private logger: Logger) {}
}
```

## Best Practices

1. **Use constructor injection** - Always use constructor injection over property injection
2. **Keep services focused** - Each service should have a single responsibility
3. **Use interfaces** - Inject interfaces rather than concrete implementations
4. **Avoid circular dependencies** - Refactor code to break circular dependencies
5. **Use dynamic modules** - For configuration-based modules
6. **Make modules global sparingly** - Only for truly global services like Config

## Common Patterns

### Repository Pattern

```typescript
@Injectable()
export class CatsRepository {
  private readonly cats: Cat[] = [];

  findAll(): Cat[] {
    return this.cats;
  }

  findOne(id: string): Cat | undefined {
    return this.cats.find(cat => cat.id === id);
  }

  create(cat: CreateCatDto): Cat {
    const newCat = { ...cat, id: randomId() };
    this.cats.push(newCat);
    return newCat;
  }
}
```

### Factory Pattern

```typescript
@Injectable()
export class NotificationFactory {
  create(type: NotificationType): NotificationService {
    switch (type) {
      case 'email':
        return new EmailNotificationService();
      case 'sms':
        return new SmsNotificationService();
      default:
        throw new Error('Unknown notification type');
    }
  }
}
```

## References

- [NestJS Providers Documentation](https://docs.nestjs.com/providers)
- [NestJS Dependency Injection](https://docs.nestjs.com/fundamentals/custom-providers)
- [NestJS Dynamic Modules](https://docs.nestjs.com/fundamentals/dynamic-modules)
