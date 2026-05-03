---
name: drizzle-nestjs
description: NestJS integration with Drizzle ORM including module setup, service patterns, repository implementations, and best practices for combining NestJS dependency injection with Drizzle query builder. Use when integrating Drizzle ORM into NestJS applications, creating database services, or implementing data access layers.
license: MIT
metadata:
  author: snowmerak
  version: "1.0"
  framework: drizzle-orm
  category: nestjs-integration
---

# Drizzle + NestJS Integration Skills

This skill covers integrating Drizzle ORM with NestJS applications, including module setup, service patterns, and best practices.

## Installation

```bash
# Install NestJS and Drizzle dependencies
npm install @nestjs/common @nestjs/core reflectx drizzle-orm pg
npm install -D drizzle-kit @types/pg
```

### PostgreSQL Adapter

```bash
npm install drizzle-orm pg
```

### MySQL Adapter

```bash
npm install drizzle-orm mysql2
```

### SQLite Adapter

```bash
npm install drizzle-orm better-sqlite3
npm install -D @types/better-sqlite3
```

## Basic Setup

### Database Module

```typescript
// database/database.module.ts
import { Module, Global } from '@nestjs/common';
import { DrizzleModule } from './drizzle.module';
import { DATABASE_CONNECTION } from './constants';

@Global()
@Module({
  imports: [
    DrizzleModule.forRoot({
      name: DATABASE_CONNECTION,
      config: {
        type: 'pg',
        connectionString: process.env.DATABASE_URL,
      },
    }),
  ],
  exports: [DrizzleModule],
})
export class DatabaseModule {}
```

### Drizzle Service Provider

```typescript
// database/drizzle.provider.ts
import { Injectable, OnModuleInit, OnModuleDestroy } from '@nestjs/common';
import { drizzle } from 'drizzle-orm/node-postgres';
import { Pool } from 'pg';

@Injectable()
export class DrizzleService implements OnModuleInit, OnModuleDestroy {
  private pool: Pool;

  constructor() {
    this.pool = new Pool({
      connectionString: process.env.DATABASE_URL,
      max: 20, // Connection pool size
      idleTimeoutMillis: 30000,
      connectionTimeoutMillis: 2000,
    });
  }

  async onModuleInit() {
    await this.pool.connect();
  }

  async onModuleDestroy() {
    await this.pool.end();
  }

  getDb() {
    return drizzle(this.pool);
  }
}
```

### Register in App Module

```typescript
// app.module.ts
import { Module } from '@nestjs/common';
import { DatabaseModule } from './database/database.module';
import { UsersModule } from './users/users.module';

@Module({
  imports: [DatabaseModule, UsersModule],
})
export class AppModule {}
```

## Service Pattern

### Repository Service

```typescript
// users/users.service.ts
import { Injectable } from '@nestjs/common';
import { Inject } from '@nestjs/core';
import { eq, and, or, like, asc, desc, count } from 'drizzle-orm';
import { DrizzleService } from '../database/drizzle.provider';
import { users, posts } from '../schema';
import type { User, CreateUserDto } from './dto';

@Injectable()
export class UsersService {
  constructor(private readonly drizzle: DrizzleService) {}

  // Create user
  async create(createUserDto: CreateUserDto): Promise<User> {
    const db = this.drizzle.getDb();
    
    const [newUser] = await db.insert(users).values({
      name: createUserDto.name,
      email: createUserDto.email,
      passwordHash: createUserDto.passwordHash,
    }).returning();

    return newUser;
  }

  // Find all users with pagination
  async findAll(page: number = 1, limit: number = 10): Promise<{
    users: User[];
    total: number;
    page: number;
    totalPages: number;
  }> {
    const db = this.drizzle.getDb();
    
    const [userList, totalCount] = await Promise.all([
      db.select()
        .from(users)
        .orderBy(desc(users.createdAt))
        .limit(limit)
        .offset((page - 1) * limit),
      
      db.select({ count: count() })
        .from(users),
    ]);

    return {
      users: userList,
      total: totalCount[0].count,
      page,
      totalPages: Math.ceil(totalCount[0].count / limit),
    };
  }

  // Find user by ID
  async findById(id: number): Promise<User | null> {
    const db = this.drizzle.getDb();
    
    const [user] = await db.select()
      .from(users)
      .where(eq(users.id, id))
      .limit(1);

    return user || null;
  }

  // Find user by email
  async findByEmail(email: string): Promise<User | null> {
    const db = this.drizzle.getDb();
    
    const [user] = await db.select()
      .from(users)
      .where(eq(users.email, email))
      .limit(1);

    return user || null;
  }

  // Search users
  async search(query: string): Promise<User[]> {
    const db = this.drizzle.getDb();
    
    return db.select()
      .from(users)
      .where(or(
        like(users.name, `%${query}%`),
        like(users.email, `%${query}%`)
      ))
      .limit(20);
  }

  // Update user
  async update(id: number, updateUserDto: Partial<CreateUserDto>): Promise<User> {
    const db = this.drizzle.getDb();
    
    const [updatedUser] = await db.update(users)
      .set({
        name: updateUserDto.name,
        email: updateUserDto.email,
        updatedAt: new Date(),
      })
      .where(eq(users.id, id))
      .returning();

    return updatedUser;
  }

  // Delete user
  async remove(id: number): Promise<void> {
    const db = this.drizzle.getDb();
    
    await db.delete(users)
      .where(eq(users.id, id));
  }

  // Get user with posts
  async findWithPosts(userId: number) {
    const db = this.drizzle.getDb();
    
    return db.select({
      user: users,
      posts: posts,
    })
    .from(users)
    .leftJoin(posts, eq(users.id, posts.authorId))
    .where(eq(users.id, userId));
  }

  // Get user statistics
  async getStatistics(userId: number) {
    const db = this.drizzle.getDb();
    
    return db.select({
      totalPosts: count(posts.id),
      totalViews: count(posts.views),
      avgViews: count(posts.views),
    })
    .from(users)
    .leftJoin(posts, eq(users.id, posts.authorId))
    .where(eq(users.id, userId));
  }
}
```

### Controller

```typescript
// users/users.controller.ts
import { 
  Controller, 
  Get, 
  Post, 
  Put, 
  Delete, 
  Body, 
  Param, 
  Query,
  UseGuards,
} from '@nestjs/common';
import { UsersService } from './users.service';
import { CreateUserDto, UpdateUserDto } from './dto';

@Controller('users')
export class UsersController {
  constructor(private readonly usersService: UsersService) {}

  @Post()
  create(@Body() createUserDto: CreateUserDto) {
    return this.usersService.create(createUserDto);
  }

  @Get()
  findAll(
    @Query('page') page: string,
    @Query('limit') limit: string,
  ) {
    return this.usersService.findAll(parseInt(page), parseInt(limit));
  }

  @Get(':id')
  findOne(@Param('id') id: string) {
    return this.usersService.findById(parseInt(id));
  }

  @Put(':id')
  update(
    @Param('id') id: string,
    @Body() updateUserDto: UpdateUserDto,
  ) {
    return this.usersService.update(parseInt(id), updateUserDto);
  }

  @Delete(':id')
  remove(@Param('id') id: string) {
    return this.usersService.remove(parseInt(id));
  }

  @Get(':id/posts')
  findWithPosts(@Param('id') id: string) {
    return this.usersService.findWithPosts(parseInt(id));
  }
}
```

### Module

```typescript
// users/users.module.ts
import { Module } from '@nestjs/common';
import { UsersController } from './users.controller';
import { UsersService } from './users.service';
import { DrizzleModule } from '../database/drizzle.module';

@Module({
  imports: [DrizzleModule],
  controllers: [UsersController],
  providers: [UsersService],
  exports: [UsersService],
})
export class UsersModule {}
```

## Advanced Patterns

### Repository Pattern with Dependency Injection

```typescript
// database/drizzle.module.ts
import { Module, DynamicModule, Provider } from '@nestjs/common';
import { drizzle } from 'drizzle-orm/node-postgres';
import { Pool } from 'pg';

export const DATABASE_CONNECTION = 'DATABASE_CONNECTION';

export interface DrizzleConfig {
  type: 'pg' | 'mysql2' | 'libsql';
  connectionString?: string;
  url?: string;
}

@Module({})
export class DrizzleModule {
  static forRoot(config: DrizzleConfig): DynamicModule {
    const dbProvider: Provider = {
      provide: DATABASE_CONNECTION,
      useFactory: () => {
        let pool: Pool;
        
        switch (config.type) {
          case 'pg':
            pool = new Pool({ connectionString: config.connectionString });
            break;
          case 'mysql2':
            // MySQL setup
            break;
          case 'libsql':
            // SQLite setup
            break;
        }
        
        return drizzle(pool);
      },
    };

    return {
      module: DrizzleModule,
      providers: [dbProvider],
      exports: [DATABASE_CONNECTION],
    };
  }
}
```

### Using Injected Database Instance

```typescript
// users/users.service.ts
import { Injectable } from '@nestjs/common';
import { Inject } from '@nestjs/core';
import type { NodePgDatabase } from 'drizzle-orm/node-postgres';
import { DATABASE_CONNECTION } from '../database/drizzle.module';
import { users, posts } from '../schema';

@Injectable()
export class UsersService {
  constructor(
    @Inject(DATABASE_CONNECTION)
    private readonly db: NodePgDatabase,
  ) {}

  async findAll() {
    return this.db.select().from(users);
  }

  async create(name: string, email: string) {
    return this.db.insert(users).values({ name, email }).returning();
  }
}
```

### Transaction Service

```typescript
// database/transaction.service.ts
import { Injectable } from '@nestjs/common';
import { Inject } from '@nestjs/core';
import type { NodePgDatabase } from 'drizzle-orm/node-postgres';
import { DATABASE_CONNECTION } from './drizzle.module';

@Injectable()
export class TransactionService {
  constructor(
    @Inject(DATABASE_CONNECTION)
    private readonly db: NodePgDatabase,
  ) {}

  async execute<T>(callback: (tx: NodePgDatabase) => Promise<T>): Promise<T> {
    // Get raw connection for transaction
    const client = await this.db.$client.connect();
    
    try {
      await client.query('BEGIN');
      
      // Create transaction-aware database instance
      const txDb = this.db.$client ? 
        this.db : 
        this.createTransactionDb(client);
      
      const result = await callback(txDb);
      
      await client.query('COMMIT');
      return result;
    } catch (error) {
      await client.query('ROLLBACK');
      throw error;
    } finally {
      client.release();
    }
  }

  private createTransactionDb(client: any) {
    // Create transaction-aware database instance
    return this.db;
  }
}
```

### Using Transaction Service

```typescript
// orders/orders.service.ts
import { Injectable } from '@nestjs/common';
import { Inject } from '@nestjs/core';
import { DATABASE_CONNECTION, type DrizzleService } from '../database/drizzle.provider';
import { TransactionService } from '../database/transaction.service';
import { orders, orderItems, products, users } from '../schema';

@Injectable()
export class OrdersService {
  constructor(
    @Inject(DATABASE_CONNECTION)
    private readonly db: DrizzleService,
    private readonly transactionService: TransactionService,
  ) {}

  async createOrder(userId: number, items: Array<{ productId: number; quantity: number }>) {
    return this.transactionService.execute(async (tx) => {
      // Create order
      const [order] = await tx.insert(orders).values({
        userId,
        status: 'pending',
        totalAmount: 0,
      }).returning();

      let totalAmount = 0;

      // Create order items and update stock
      for (const item of items) {
        const product = await tx.select()
          .from(products)
          .where(eq(products.id, item.productId))
          .limit(1);

        if (!product[0]) {
          throw new Error(`Product ${item.productId} not found`);
        }

        if (product[0].stock < item.quantity) {
          throw new Error(`Insufficient stock for product ${item.productId}`);
        }

        // Update stock
        await tx.update(products)
          .set({ stock: product[0].stock - item.quantity })
          .where(eq(products.id, item.productId));

        // Create order item
        totalAmount += product[0].price * item.quantity;
        
        await tx.insert(orderItems).values({
          orderId: order.id,
          productId: item.productId,
          quantity: item.quantity,
          price: product[0].price,
        });
      }

      // Update order total
      await tx.update(orders)
        .set({ totalAmount })
        .where(eq(orders.id, order.id));

      return order;
    });
  }
}
```

## Configuration Management

### Using NestJS ConfigModule

```typescript
// config/database.config.ts
import { registerAs } from '@nestjs/config';

export default registerAs('database', () => ({
  type: process.env.DB_TYPE || 'pg',
  host: process.env.DB_HOST,
  port: parseInt(process.env.DB_PORT || '5432'),
  username: process.env.DB_USERNAME,
  password: process.env.DB_PASSWORD,
  database: process.env.DB_NAME,
  url: process.env.DATABASE_URL,
  pool: {
    min: parseInt(process.env.DB_POOL_MIN || '2'),
    max: parseInt(process.env.DB_POOL_MAX || '10'),
  },
}));
```

### App Module with Config

```typescript
// app.module.ts
import { Module } from '@nestjs/common';
import { ConfigModule, ConfigService } from '@nestjs/config';
import { DatabaseModule } from './database/database.module';
import { UsersModule } from './users/users.module';
import databaseConfig from './config/database.config';

@Module({
  imports: [
    ConfigModule.forRoot({
      isGlobal: true,
      load: [databaseConfig],
    }),
    DatabaseModule,
    UsersModule,
  ],
})
export class AppModule {}
```

### Dynamic Drizzle Module with Config

```typescript
// database/drizzle.module.ts
import { Module, DynamicModule, Provider } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import { drizzle } from 'drizzle-orm/node-postgres';
import { Pool } from 'pg';

export const DRIZZLE_CONNECTION = 'DRIZZLE_CONNECTION';

@Module({})
export class DrizzleModule {
  static forRootAsync(): DynamicModule {
    return {
      module: DrizzleModule,
      imports: [], // ConfigModule should be imported separately
      providers: [
        {
          provide: DRIZZLE_CONNECTION,
          inject: [ConfigService],
          useFactory: (configService: ConfigService) => {
            const dbConfig = configService.get('database');
            
            const pool = new Pool({
              host: dbConfig.host,
              port: dbConfig.port,
              user: dbConfig.username,
              password: dbConfig.password,
              database: dbConfig.database,
              max: dbConfig.pool.max,
              min: dbConfig.pool.min,
            });

            return drizzle(pool);
          },
        },
      ],
      exports: [DRIZZLE_CONNECTION],
    };
  }
}
```

## Best Practices

### 1. Connection Pooling

```typescript
// Always use connection pooling in production
const pool = new Pool({
  host: config.host,
  port: config.port,
  user: config.username,
  password: config.password,
  database: config.database,
  max: 20, // Maximum connections
  idleTimeoutMillis: 30000,
  connectionTimeoutMillis: 2000,
});

// Use the pool with Drizzle
const db = drizzle(pool);
```

### 2. Error Handling

```typescript
// users/users.service.ts
import { Injectable, NotFoundException } from '@nestjs/common';

@Injectable()
export class UsersService {
  async findById(id: number): Promise<User> {
    const user = await this.db.select()
      .from(users)
      .where(eq(users.id, id))
      .limit(1);

    if (!user[0]) {
      throw new NotFoundException(`User with ID ${id} not found`);
    }

    return user[0];
  }
}
```

### 3. Type Safety

```typescript
// Define types from schema
import type { InferModel } from 'drizzle-orm';
import { users, posts } from '../schema';

export type User = InferModel<typeof users>;
export type CreateUserDto = InferModel<typeof users, 'insert'>;
export type UpdateUserDto = Partial<InferModel<typeof users, 'update'>>;

export type Post = InferModel<typeof posts>;
```

### 4. Repository Pattern

```typescript
// database/repository.base.ts
import { eq } from 'drizzle-orm';
import type { NodePgDatabase } from 'drizzle-orm/node-postgres';

export abstract class BaseRepository<T, CreateDto, UpdateDto> {
  protected constructor(protected readonly db: NodePgDatabase) {}

  abstract get table(): any;

  async findAll(): Promise<T[]> {
    return this.db.select().from(this.table);
  }

  async findById(id: number): Promise<T | null> {
    const [result] = await this.db
      .select()
      .from(this.table)
      .where(eq(this.table.id, id))
      .limit(1);
    
    return result || null;
  }

  async create(data: CreateDto): Promise<T> {
    const [result] = await this.db
      .insert(this.table)
      .values(data)
      .returning();
    
    return result;
  }

  async update(id: number, data: UpdateDto): Promise<T> {
    const [result] = await this.db
      .update(this.table)
      .set(data)
      .where(eq(this.table.id, id))
      .returning();
    
    return result;
  }

  async remove(id: number): Promise<void> {
    await this.db
      .delete(this.table)
      .where(eq(this.table.id, id));
  }
}
```

### 5. Use with NestJS Lifecycle Hooks

```typescript
// database/drizzle.service.ts
import { Injectable, OnModuleInit, OnModuleDestroy } from '@nestjs/common';
import { drizzle } from 'drizzle-orm/node-postgres';
import { Pool } from 'pg';

@Injectable()
export class DrizzleService implements OnModuleInit, OnModuleDestroy {
  private pool: Pool;
  private db: ReturnType<typeof drizzle>;

  constructor() {
    this.pool = new Pool({
      connectionString: process.env.DATABASE_URL,
    });
  }

  async onModuleInit() {
    // Test connection
    const client = await this.pool.connect();
    await client.query('SELECT 1');
    client.release();
    
    this.db = drizzle(this.pool);
  }

  async onModuleDestroy() {
    await this.pool.end();
  }

  getDb() {
    return this.db;
  }
}
```

## Testing with Drizzle

### Test Database Setup

```typescript
// test/setup.ts
import { drizzle } from 'drizzle-orm/better-sqlite3';
import Database from 'better-sqlite3';

export const createTestDb = () => {
  const sqlite = new Database(':memory:');
  return drizzle(sqlite);
};

// Reset database between tests
export const resetDatabase = async (db: ReturnType<typeof drizzle>) => {
  // Drop all tables and recreate from schema
  await db.execute(`DROP TABLE IF EXISTS users, posts, comments`);
  // Run migrations or create tables from schema
};
```

### Testing Service

```typescript
// users/users.service.spec.ts
import { Test, TestingModule } from '@nestjs/testing';
import { UsersService } from './users.service';
import { DrizzleService } from '../database/drizzle.provider';
import { createTestDb } from '../../test/setup';

describe('UsersService', () => {
  let service: UsersService;
  let db: ReturnType<typeof drizzle>;

  beforeEach(async () => {
    db = createTestDb();
    
    const module: TestingModule = await Test.createTestingModule({
      providers: [
        UsersService,
        {
          provide: DrizzleService,
          useValue: { getDb: () => db },
        },
      ],
    }).compile();

    service = module.get<UsersService>(UsersService);
  });

  it('should create a user', async () => {
    const user = await service.create({
      name: 'Test User',
      email: 'test@example.com',
      passwordHash: 'hashed_password',
    });

    expect(user).toBeDefined();
    expect(user.name).toBe('Test User');
  });
});
```

## Migration in NestJS

### Run Migrations on Startup

```typescript
// main.ts
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';
import * as dotenv from 'dotenv';

async function bootstrap() {
  dotenv.config();
  
  const app = await NestFactory.create(AppModule);
  
  // Run migrations before starting the app
  if (process.env.RUN_MIGRATIONS === 'true') {
    const { exec } = require('child_process');
    await new Promise((resolve, reject) => {
      exec('npx drizzle-kit migrate', (error: Error | null) => {
        if (error) reject(error);
        else resolve(true);
      });
    });
  }
  
  await app.listen(3000);
}
bootstrap();
```

### Migration Service

```typescript
// database/migration.service.ts
import { Injectable, OnModuleInit } from '@nestjs/common';
import { exec } from 'child_process';
import * as util from 'util;

const execPromise = util.promisify(exec);

@Injectable()
export class MigrationService implements OnModuleInit {
  async onModuleInit() {
    if (process.env.RUN_MIGRATIONS !== 'false') {
      await this.runMigrations();
    }
  }

  private async runMigrations() {
    try {
      const { stdout, stderr } = await execPromise('npx drizzle-kit migrate');
      console.log('Migrations completed:', stdout);
    } catch (error) {
      console.error('Migration error:', error);
      throw error;
    }
  }
}
```

## Best Practices Summary

1. **Use connection pooling** - Always configure pool size appropriately
2. **Implement repository pattern** - Abstract database logic into services
3. **Handle transactions properly** - Use try/catch/finally for rollback
4. **Type safety** - Leverage TypeScript types from schema definitions
5. **Error handling** - Throw appropriate NestJS exceptions
6. **Configuration management** - Use ConfigModule for environment-specific settings
7. **Testing** - Use in-memory databases for unit tests
8. **Migration strategy** - Run migrations on startup or via CI/CD pipeline

## References

- [NestJS Documentation](https://docs.nestjs.com)
- [Drizzle ORM Documentation](https://orm.drizzle.team)
- [Drizzle + NestJS Integration Guide](https://orm.drizzle.team/docs/get-started-postgres)
