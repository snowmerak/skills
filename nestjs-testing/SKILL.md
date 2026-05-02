---
name: nestjs-testing
description: NestJS testing strategies including unit tests, integration tests, e2e tests, and mocking. Use when writing tests for NestJS applications, mocking dependencies, or testing controllers, services, and modules.
license: MIT
metadata:
  author: snowmerak
  version: "1.0"
  framework: nestjs
  category: testing
---

# NestJS Testing Skills

This skill covers testing strategies for NestJS applications including unit tests, integration tests, and e2e tests.

## Test Types

### 1. Unit Tests

Unit tests isolate individual components (services, controllers) without external dependencies.

#### Testing Services

```typescript
// cats.service.ts
import { Injectable } from '@nestjs/common';

@Injectable()
export class CatsService {
  private readonly cats: string[] = [];

  create(cat: string) {
    this.cats.push(cat);
  }

  findAll(): string[] {
    return this.cats;
  }
}
```

```typescript
// cats.service.spec.ts
import { Test, TestingModule } from '@nestjs/testing';
import { CatsService } from './cats.service';

describe('CatsService', () => {
  let service: CatsService;

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [CatsService],
    }).compile();

    service = module.get<CatsService>(CatsService);
  });

  it('should be defined', () => {
    expect(service).toBeDefined();
  });

  it('should return all cats', () => {
    service.create('Kitty');
    expect(service.findAll()).toEqual(['Kitty']);
  });
});
```

#### Testing Controllers

```typescript
// cats.controller.ts
import { Controller, Get, Post, Body } from '@nestjs/common';
import { CatsService } from './cats.service';

@Controller('cats')
export class CatsController {
  constructor(private readonly catsService: CatsService) {}

  @Post()
  create(@Body() createCatDto: string) {
    this.catsService.create(createCatDto);
  }

  @Get()
  findAll() {
    return this.catsService.findAll();
  }
}
```

```typescript
// cats.controller.spec.ts
import { Test, TestingModule } from '@nestjs/testing';
import { CatsController } from './cats.controller';
import { CatsService } from './cats.service';

describe('CatsController', () => {
  let controller: CatsController;
  let service: CatsService;

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      controllers: [CatsController],
      providers: [
        {
          provide: CatsService,
          useValue: {
            create: jest.fn(),
            findAll: jest.fn().mockReturnValue(['Kitty']),
          },
        },
      ],
    }).compile();

    controller = module.get<CatsController>(CatsController);
    service = module.get<CatsService>(CatsService);
  });

  it('should create a cat', () => {
    controller.create('Kitty');
    expect(service.create).toHaveBeenCalledWith('Kitty');
  });

  it('should return all cats', () => {
    const result = controller.findAll();
    expect(result).toEqual(['Kitty']);
  });
});
```

### 2. Integration Tests

Integration tests test multiple components working together.

```typescript
import { Test, TestingModule } from '@nestjs/testing';
import { CatsModule } from './cats.module';
import { CatsService } from './cats.service';

describe('CatsModule Integration', () => {
  let module: TestingModule;
  let service: CatsService;

  beforeEach(async () => {
    module = await Test.createTestingModule({
      imports: [CatsModule],
    }).compile();

    service = module.get<CatsService>(CatsService);
  });

  it('should work with real dependencies', () => {
    service.create('Kitty');
    expect(service.findAll()).toEqual(['Kitty']);
  });
});
```

### 3. E2E Tests

E2E tests test the entire application with a real HTTP server.

#### Setup

```typescript
// test/jest-e2e.json
{
  "moduleFileExtensions": ["js", "json", "ts"],
  "rootDir": ".",
  "testEnvironment": "node",
  "testRegex": ".e2e-spec.ts$",
  "transform": {
    "^.+\\.(t|j)s$": "ts-jest"
  }
}
```

#### E2E Test Example

```typescript
// cats.e2e-spec.ts
import { Test, TestingModule } from '@nestjs/testing';
import { INestApplication } from '@nestjs/common';
import * as request from 'supertest';
import { AppModule } from '../src/app.module';

describe('Cats E2E', () => {
  let app: INestApplication;

  beforeEach(async () => {
    const moduleFixture: TestingModule = await Test.createTestingModule({
      imports: [AppModule],
    }).compile();

    app = moduleFixture.createNestApplication();
    await app.init();
  });

  afterEach(async () => {
    await app.close();
  });

  it('/cats (GET)', () => {
    return request(app.getHttpServer())
      .get('/cats')
      .expect(200)
      .expect([]);
  });

  it('/cats (POST)', () => {
    return request(app.getHttpServer())
      .post('/cats')
      .send({ name: 'Kitty', age: 2, breed: 'Persian' })
      .expect(201)
      .expect({ message: 'Cat created successfully' });
  });
});
```

## Mocking Strategies

### Mocking Services

```typescript
const mockCatsService = {
  findAll: jest.fn().mockReturnValue(['Kitty']),
  create: jest.fn(),
};

beforeEach(async () => {
  const module: TestingModule = await Test.createTestingModule({
    controllers: [CatsController],
    providers: [
      {
        provide: CatsService,
        useValue: mockCatsService,
      },
    ],
  }).compile();
});
```

### Mocking Modules

```typescript
import { TypeOrmModule } from '@nestjs/typeorm';

beforeEach(async () => {
  const module: TestingModule = await Test.createTestingModule({
    imports: [
      // Mock TypeORM module
      TypeOrmModule.forRoot({
        type: 'sqlite',
        database: ':memory:',
        entities: [Cat],
        synchronize: true,
      }),
    ],
  }).compile();
});
```

### Mocking External Services

```typescript
import { HttpService } from '@nestjs/axios';

beforeEach(async () => {
  const module: TestingModule = await Test.createTestingModule({
    providers: [
      MyService,
      {
        provide: HttpService,
        useValue: {
          get: jest.fn().mockResolvedValue({ data: { result: 'success' } }),
        },
      },
    ],
  }).compile();
});
```

## Testing Utilities

### Testing Module Builder

```typescript
// Create with specific providers
const module = await Test.createTestingModule({
  providers: [MyService, MockDependency],
}).compile();

// Import modules
const module = await Test.createTestingModule({
  imports: [MyModule],
}).compile();

// Override providers
const module = await Test.createTestingModule({
  imports: [MyModule],
}).overrideProvider(MyService).useValue(mockService).compile();

// Override guards
const module = await Test.createTestingModule({
  imports: [MyModule],
}).overrideGuard(AuthGuard).useValue({ canActivate: () => true }).compile();

// Override pipes
const module = await Test.createTestingModule({
  imports: [MyModule],
}).overridePipe(ValidationPipe).useValue(new ValidationPipe({ whitelist: true })).compile();
```

### Testing Controllers with Supertest

```typescript
import * as request from 'supertest';

describe('Controller E2E', () => {
  let app: INestApplication;

  beforeAll(async () => {
    app = await Test.createTestingModule({
      imports: [AppModule],
    }).compile();
    await app.init();
  });

  it('GET /cats', () => {
    return request(app.getHttpServer())
      .get('/cats')
      .expect(200)
      .expect('Content-Type', /json/)
      .expect([
        { name: 'Kitty', age: 2 },
      ]);
  });
});
```

## Best Practices

1. **Test behavior, not implementation** - Focus on what the code does, not how
2. **Use mocks for external dependencies** - Isolate units under test
3. **Test edge cases** - Test null, undefined, empty arrays, errors
4. **Use descriptive test names** - Explain what behavior is being tested
5. **Keep tests independent** - Each test should be able to run alone
6. **Test error scenarios** - Test exception handling
7. **Use beforeEach/afterEach** - Clean up between tests

## Testing Commands

```bash
# Run unit tests
npm run test

# Run unit tests in watch mode
npm run test:watch

# Run unit tests with coverage
npm run test:cov

# Run e2e tests
npm run test:e2e

# Run all tests
npm run test
```

## Jest Configuration

```json
{
  "jest": {
    "moduleFileExtensions": ["js", "json", "ts"],
    "rootDir": "src",
    "testRegex": ".*\\.spec\\.ts$",
    "transform": {
      "^.+\\.(t|j)s$": "ts-jest"
    },
    "collectCoverageFrom": [
      "**/*.(t|j)s"
    ],
    "coverageDirectory": "../coverage",
    "testEnvironment": "node"
  }
}
```

## References

- [NestJS Testing Documentation](https://docs.nestjs.com/fundamentals/testing)
- [Jest Documentation](https://jestjs.io)
- [Supertest Documentation](https://github.com/ladjs/supertest)
