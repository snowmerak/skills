---
name: nestjs-database
description: NestJS database integration with TypeORM, Prisma, Mongoose, and SQL drivers. Use when working with databases, creating repositories, managing migrations, or implementing data access layers in NestJS.
license: MIT
metadata:
  author: snowmerak
  version: "1.0"
  framework: nestjs
  category: techniques
---

# NestJS Database Integration Skills

This skill covers database integration in NestJS using TypeORM, Prisma, Mongoose, and other database drivers.

## Overview

NestJS supports multiple database solutions through modules and providers. The choice depends on your project requirements.

## 1. TypeORM Integration

### Installation

```bash
npm install --save @nestjs/typeorm typeorm sqlite3
```

### Configuration

```typescript
// app.module.ts
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { CatsModule } from './cats/cats.module';

@Module({
  imports: [
    TypeOrmModule.forRoot({
      type: 'sqlite',
      database: 'database.sqlite',
      entities: [__dirname + '/**/*.entity{.ts,.js}'],
      synchronize: true,
    }),
    CatsModule,
  ],
})
export class AppModule {}
```

### Creating Entities

```typescript
// cat.entity.ts
import { Entity, PrimaryGeneratedColumn, Column, OneToMany } from 'typeorm';
import { CatToy } from './cat-toy.entity';

@Entity()
export class Cat {
  @PrimaryGeneratedColumn()
  id: number;

  @Column()
  name: string;

  @Column({ type: 'int' })
  age: number;

  @Column()
  breed: string;

  @OneToMany(() => CatToy, catToy => catToy.cat)
  catToys: CatToy[];
}
```

### Using Repositories

```typescript
// cats.service.ts
import { Injectable } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { Cat } from './entities/cat.entity';
import { CreateCatDto } from './dto/create-cat.dto';

@Injectable()
export class CatsService {
  constructor(
    @InjectRepository(Cat)
    private catsRepository: Repository<Cat>,
  ) {}

  async create(createCatDto: CreateCatDto): Promise<Cat> {
    const cat = this.catsRepository.create(createCatDto);
    return this.catsRepository.save(cat);
  }

  async findAll(): Promise<Cat[]> {
    return this.catsRepository.find();
  }

  async findOne(id: number): Promise<Cat> {
    return this.catsRepository.findOneBy({ id });
  }

  async remove(id: number): Promise<void> {
    await this.catsRepository.delete(id);
  }
}
```

### TypeORM Module Options

```typescript
// Dynamic module configuration
TypeOrmModule.forRootAsync({
  imports: [ConfigModule],
  inject: [ConfigService],
  useFactory: (config: ConfigService) => ({
    type: 'postgres',
    host: config.get('DATABASE_HOST'),
    port: config.get('DATABASE_PORT'),
    username: config.get('DATABASE_USERNAME'),
    password: config.get('DATABASE_PASSWORD'),
    database: config.get('DATABASE_NAME'),
    entities: [__dirname + '/**/*.entity{.ts,.js}'],
    synchronize: config.get('DATABASE_SYNCHRONIZE'),
  }),
})
```

### Custom Repository

```typescript
// cat.repository.ts
import { EntityRepository, Repository } from 'typeorm';
import { Cat } from './cat.entity';

@EntityRepository(Cat)
export class CatRepository extends Repository<Cat> {
  async findAllWithToys(): Promise<Cat[]> {
    return this.createQueryBuilder('cat')
      .leftJoinAndSelect('cat.catToys', 'catToy')
      .getMany();
  }

  async findByName(name: string): Promise<Cat[]> {
    return this.findBy({ name });
  }
}
```

## 2. Prisma Integration

### Installation

```bash
npm install --save @nestjs/prisma prisma
npm install --save-dev @types/prisma
npx prisma init
```

### Configuration

```typescript
// prisma.service.ts
import { Injectable, OnModuleInit } from '@nestjs/common';
import { PrismaClient } from '@prisma/client';

@Injectable()
export class PrismaService extends PrismaClient implements OnModuleInit {
  async onModuleInit() {
    await this.$connect();
  }
}
```

```typescript
// app.module.ts
import { Module } from '@nestjs/common';
import { PrismaService } from './prisma.service';
import { CatsModule } from './cats/cats.module';

@Module({
  imports: [CatsModule],
  providers: [PrismaService],
})
export class AppModule {}
```

### Using Prisma

```typescript
// cats.service.ts
import { Injectable } from '@nestjs/common';
import { PrismaService } from '../prisma.service';
import { CreateCatDto } from './dto/create-cat.dto';

@Injectable()
export class CatsService {
  constructor(private prisma: PrismaService) {}

  async create(createCatDto: CreateCatDto) {
    return this.prisma.cat.create({
      data: createCatDto,
    });
  }

  async findAll() {
    return this.prisma.cat.findMany();
  }

  async findOne(id: number) {
    return this.prisma.cat.findUnique({
      where: { id },
    });
  }

  async remove(id: number) {
    await this.prisma.cat.delete({
      where: { id },
    });
  }
}
```

## 3. Mongoose (MongoDB) Integration

### Installation

```bash
npm install --save @nestjs/mongoose mongoose
```

### Configuration

```typescript
// app.module.ts
import { Module } from '@nestjs/common';
import { MongooseModule } from '@nestjs/mongoose';
import { CatsModule } from './cats/cats.module';

@Module({
  imports: [
    MongooseModule.forRoot('mongodb://localhost:27017/nest'),
    CatsModule,
  ],
})
export class AppModule {}
```

### Creating Schemas

```typescript
// cat.schema.ts
import { Prop, Schema, SchemaFactory } from '@nestjs/mongoose';
import { Document } from 'mongoose';

export type CatDocument = Cat & Document;

@Schema()
export class Cat extends Document {
  @Prop()
  name: string;

  @Prop()
  age: number;

  @Prop()
  breed: string;
}

export const CatSchema = SchemaFactory.createForClass(Cat);
```

### Using Mongoose Documents

```typescript
// cats.service.ts
import { Injectable } from '@nestjs/common';
import { InjectModel } from '@nestjs/mongoose';
import { Model } from 'mongoose';
import { Cat, CatDocument } from './schemas/cat.schema';
import { CreateCatDto } from './dto/create-cat.dto';

@Injectable()
export class CatsService {
  constructor(
    @InjectModel(Cat.name)
    private catModel: Model<CatDocument>,
  ) {}

  async create(createCatDto: CreateCatDto): Promise<Cat> {
    const createdCat = new this.catModel(createCatDto);
    return createdCat.save();
  }

  async findAll(): Promise<Cat[]> {
    return this.catModel.find().exec();
  }

  async findOne(id: string): Promise<Cat> {
    return this.catModel.findById(id).exec();
  }

  async remove(id: string): Promise<Cat> {
    return this.catModel.findByIdAndUpdate(id, {}, { new: true });
  }
}
```

## 4. SQL Drivers (knex, pg, mysql2)

### Using Knex

```typescript
// database.module.ts
import { Module } from '@nestjs/common';
import { knex } from 'knex';

const databaseProvider = {
  provide: 'KNEX',
  useFactory: () => {
    return knex({
      client: 'pg',
      connection: {
        host: process.env.DB_HOST,
        port: parseInt(process.env.DB_PORT),
        database: process.env.DB_NAME,
        user: process.env.DB_USER,
        password: process.env.DB_PASSWORD,
      },
    });
  },
};

@Module({
  providers: [databaseProvider],
  exports: ['KNEX'],
})
export class DatabaseModule {}
```

### Using Knex in Service

```typescript
// cats.service.ts
import { Injectable, Inject } from '@nestjs/common';

@Injectable()
export class CatsService {
  constructor(@Inject('KNEX') private readonly knex: any) {}

  async findAll() {
    return this.knex('cats').select('*');
  }

  async create(data: any) {
    return this.knex('cats').insert(data).returning('*');
  }
}
```

## Database Best Practices

### 1. Use DTOs for Validation

```typescript
// create-cat.dto.ts
import { IsString, IsInt, Min, Max } from 'class-validator';

export class CreateCatDto {
  @IsString()
  name: string;

  @IsInt()
  @Min(0)
  @Max(30)
  age: number;

  @IsString()
  breed: string;
}
```

### 2. Use Transactions

```typescript
async createWithTransaction(data: CreateCatDto) {
  const queryRunner = this.dataSource.createQueryRunner();
  await queryRunner.connect();
  await queryRunner.startTransaction();

  try {
    const cat = await queryRunner.manager.save(Cat, data);
    await queryRunner.commitTransaction();
    return cat;
  } catch (error) {
    await queryRunner.rollbackTransaction();
    throw error;
  } finally {
    await queryRunner.release();
  }
}
```

### 3. Handle Connections Properly

```typescript
// OnModuleDestroy for cleanup
import { Injectable, OnModuleDestroy } from '@nestjs/common';
import { DataSource } from 'typeorm';

@Injectable()
export class CatsService implements OnModuleDestroy {
  constructor(private dataSource: DataSource) {}

  async onModuleDestroy() {
    await this.dataSource.destroy();
  }
}
```

### 4. Use Indexes

```typescript
@Entity()
@Index(['name', 'breed']) // Compound index
export class Cat {
  @PrimaryGeneratedColumn()
  id: number;

  @Column()
  @Index() // Single column index
  name: string;

  @Column()
  breed: string;
}
```

## Database Migration Commands

### TypeORM

```bash
# Generate migration
npx typeorm migration:generate -n InitialMigration

# Run migrations
npx typeorm migration:run

# Revert last migration
npx typeorm migration:revert
```

### Prisma

```bash
# Create migration
npx prisma migrate dev --name init

# Apply migrations
npx prisma migrate deploy

# Reset database
npx prisma migrate reset
```

## References

- [NestJS TypeORM Documentation](https://docs.nestjs.com/techniques/typeorm)
- [NestJS Mongoose Documentation](https://docs.nestjs.com/techniques/mongodb)
- [TypeORM Documentation](https://typeorm.io)
- [Prisma Documentation](https://www.prisma.io)
- [Mongoose Documentation](https://mongoosejs.com)
