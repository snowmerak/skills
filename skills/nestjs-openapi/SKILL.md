---
name: nestjs-openapi
description: NestJS OpenAPI (Swagger) integration for API documentation. Use when generating API documentation, documenting endpoints, or creating interactive API specs in NestJS.
license: MIT
metadata:
  author: snowmerak
  version: "1.0"
  framework: nestjs
  category: openapi
---

# NestJS OpenAPI (Swagger) Skills

This skill covers OpenAPI/Swagger documentation generation in NestJS applications.

## Overview

NestJS provides built-in Swagger integration through the `@nestjs/swagger` package, making it easy to generate comprehensive API documentation.

## Installation

```bash
npm install --save @nestjs/swagger swagger-ui-express
```

## 1. Basic Setup

### Swagger Module Configuration

```typescript
// app.module.ts
import { Module } from '@nestjs/common';
import { SwaggerModule, DocumentBuilder } from '@nestjs/swagger';
import { CatsModule } from './cats/cats.module';

@Module({
  imports: [CatsModule],
})
export class AppModule {
  configure(app: any) {
    const config = new DocumentBuilder()
      .setTitle('Cats example')
      .setDescription('The cats API description')
      .setVersion('1.0')
      .addTag('cats')
      .addBearerAuth()
      .build();
    
    const document = SwaggerModule.createDocument(app, config);
    SwaggerModule.setup('api', app, document);
  }
}
```

### Alternative Setup in main.ts

```typescript
// main.ts
import { NestFactory } from '@nestjs/core';
import { SwaggerModule, DocumentBuilder } from '@nestjs/swagger';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  
  const config = new DocumentBuilder()
    .setTitle('Cats API')
    .setDescription('The cats API description')
    .setVersion('1.0')
    .addTag('cats')
    .addBearerAuth()
    .build();
  
  const document = SwaggerModule.createDocument(app, config);
  SwaggerModule.setup('api', app, document);
  
  await app.listen(3000);
}
bootstrap();
```

## 2. API Documentation Decorators

### Controller Documentation

```typescript
import { Controller, Get, Post, Body, Param } from '@nestjs/common';
import { ApiTags, ApiOperation, ApiResponse } from '@nestjs/swagger';
import { CatsService } from './cats.service';
import { CreateCatDto } from './dto/create-cat.dto';
import { Cat } from './entities/cat.entity';

@ApiTags('cats')
@Controller('cats')
export class CatsController {
  constructor(private readonly catsService: CatsService) {}

  @Post()
  @ApiOperation({ summary: 'Create a cat' })
  @ApiResponse({ status: 201, description: 'Cat created successfully', type: Cat })
  @ApiResponse({ status: 400, description: 'Bad request' })
  create(@Body() createCatDto: CreateCatDto): Cat {
    return this.catsService.create(createCatDto);
  }

  @Get()
  @ApiOperation({ summary: 'Get all cats' })
  @ApiResponse({ status: 200, description: 'Return all cats', type: [Cat] })
  findAll(): Cat[] {
    return this.catsService.findAll();
  }

  @Get(':id')
  @ApiOperation({ summary: 'Get a cat by ID' })
  @ApiResponse({ status: 200, description: 'Return cat by ID' })
  @ApiResponse({ status: 404, description: 'Cat not found' })
  findOne(@Param('id') id: string): Cat {
    return this.catsService.findOne(id);
  }
}
```

### DTO Documentation

```typescript
import { IsString, IsInt, Min, Max } from 'class-validator';
import { ApiProperty, ApiPropertyOptional } from '@nestjs/swagger';

export class CreateCatDto {
  @ApiProperty({
    description: 'Name of the cat',
    example: 'Kitty',
    required: true,
  })
  @IsString()
  name: string;

  @ApiProperty({
    description: 'Age of the cat',
    example: 2,
    minimum: 0,
    maximum: 30,
  })
  @IsInt()
  @Min(0)
  @Max(30)
  age: number;

  @ApiProperty({
    description: 'Breed of the cat',
    example: 'Persian',
  })
  @IsString()
  breed: string;

  @ApiPropertyOptional({
    description: 'Email of the owner',
    example: 'owner@example.com',
  })
  @IsString()
  ownerEmail?: string;
}
```

### Entity Documentation

```typescript
import { Entity, PrimaryGeneratedColumn, Column } from 'typeorm';
import { ApiProperty } from '@nestjs/swagger';

export class Cat {
  @ApiProperty({ example: 1, description: 'Cat ID' })
  @PrimaryGeneratedColumn()
  id: number;

  @ApiProperty({ example: 'Kitty', description: 'Cat name' })
  @Column()
  name: string;

  @ApiProperty({ example: 2, description: 'Cat age' })
  @Column({ type: 'int' })
  age: number;

  @ApiProperty({ example: 'Persian', description: 'Cat breed' })
  @Column()
  breed: string;
}
```

## 3. Advanced Documentation

### Schema References

```typescript
import { ApiProperty, getSchemaPath } from '@nestjs/swagger';

export class CatResponseDto {
  @ApiProperty()
  id: number;

  @ApiProperty()
  name: string;

  @ApiProperty({
    example: {
      id: 1,
      name: 'Kitty',
      age: 2,
    },
  })
  cat: Cat;
}
```

### Response Types

```typescript
@Get()
@ApiResponse({
  status: 200,
  description: 'Array of cats',
  schema: {
    type: 'array',
    items: { $ref: getSchemaPath(Cat) },
  },
})
findAll(): Cat[] {}
```

### Request Body Documentation

```typescript
@Post()
@ApiOperation({ summary: 'Create a new cat' })
@ApiResponse({ status: 201, type: Cat })
@ApiBody({
  type: CreateCatDto,
  examples: {
    persian: {
      summary: 'Persian cat',
      value: {
        name: 'Fluffy',
        age: 3,
        breed: 'Persian',
      },
    },
    siamese: {
      summary: 'Siamese cat',
      value: {
        name: 'Kitty',
        age: 2,
        breed: 'Siamese',
      },
    },
  },
})
create(@Body() createCatDto: CreateCatDto): Cat {
  return this.catsService.create(createCatDto);
}
```

### Query Parameters

```typescript
@Get()
@ApiOperation({ summary: 'Filter cats' })
@ApiQuery({
  name: 'age',
  required: false,
  type: Number,
  description: 'Filter by age',
})
@ApiQuery({
  name: 'breed',
  required: false,
  type: String,
  description: 'Filter by breed',
})
findAll(@Query('age') age?: number, @Query('breed') breed?: string): Cat[] {}
```

### Path Parameters

```typescript
@Get(':id')
@ApiOperation({ summary: 'Get cat by ID' })
@ApiParam({
  name: 'id',
  required: true,
  type: Number,
  description: 'Cat ID',
})
findOne(@Param('id') id: string): Cat {}
```

### Headers

```typescript
@Get()
@ApiOperation({ summary: 'Get cats with pagination' })
@ApiHeader({
  name: 'X-Request-ID',
  required: false,
  description: 'Request ID for tracking',
})
findAll(): Cat[] {}
```

## 4. Authentication Documentation

### Bearer Auth

```typescript
const config = new DocumentBuilder()
  .addBearerAuth({
    type: 'http',
    scheme: 'bearer',
    bearerFormat: 'JWT',
    name: 'JWT',
    description: 'Enter JWT token',
    in: 'header',
  })
  .build();
```

### Applying Auth to Endpoints

```typescript
@Get()
@ApiOperation({ summary: 'Get all cats' })
@ApiResponse({ status: 200, type: [Cat] })
@ApiBearerAuth()
findAll(): Cat[] {}
```

## 5. Conditional Documentation

### Environment-based Swagger

```typescript
// main.ts
if (app.get(ConfigService).get('ENABLE_SWAGGER')) {
  const config = new DocumentBuilder()
    .setTitle('API')
    .setDescription('API Documentation')
    .setVersion('1.0')
    .build();
  
  const document = SwaggerModule.createDocument(app, config);
  SwaggerModule.setup('api', app, document);
}
```

### Feature Flags

```typescript
@ApiHideProperty()
@Exclude()
internalField: string;

@ApiIgnore()
@Get('internal')
internalEndpoint() {}
```

## 6. Custom Options

### Swagger UI Options

```typescript
SwaggerModule.setup('api', app, document, {
  swaggerOptions: {
    persistAuthorization: true,
    docExpansion: 'list',
    defaultModelsExpandDepth: -1,
    displayRequestDuration: true,
    filter: true,
  },
  customSiteTitle: 'My API Documentation',
});
```

### Multiple Documents

```typescript
const config1 = new DocumentBuilder()
  .setTitle('Public API')
  .addTag('public')
  .build();

const config2 = new DocumentBuilder()
  .setTitle('Admin API')
  .addTag('admin')
  .addBearerAuth()
  .build();

SwaggerModule.setup('api/public', app, document1);
SwaggerModule.setup('api/admin', app, document2);
```

## 7. NestJS Devtools Integration

NestJS Devtools provides enhanced Swagger experience:
- Graph visualizer
- Routes navigator
- Interactive playground
- CI/CD integration

## Best Practices

1. **Document all endpoints** - Use `@ApiOperation` and `@ApiResponse`
2. **Provide examples** - Use `example` property in `@ApiProperty`
3. **Use tags** - Group related endpoints with `@ApiTags`
4. **Document authentication** - Show auth requirements clearly
5. **Keep DTOs clean** - Separate validation DTOs from response DTOs
6. **Use schema references** - For complex nested objects
7. **Test documentation** - Ensure examples are accurate
8. **Update on changes** - Keep docs in sync with code

## References

- [NestJS Swagger Documentation](https://docs.nestjs.com/openapi/introduction)
- [OpenAPI Specification](https://swagger.io/specification/)
- [Swagger UI](https://swagger.io/tools/swagger-ui/)
