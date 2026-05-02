---
name: nestjs-validation
description: NestJS data validation and transformation using class-validator, class-transformer, and custom validation pipes. Use when validating request data, creating DTOs, or implementing custom validators in NestJS.
license: MIT
metadata:
  author: snowmerak
  version: "1.0"
  framework: nestjs
  category: techniques
---

# NestJS Validation Skills

This skill covers data validation and transformation in NestJS applications using class-validator and class-transformer.

## Overview

NestJS provides powerful validation through the `ValidationPipe` which uses `class-validator` and `class-transformer` libraries for decorator-based validation.

## Installation

```bash
npm install --save class-validator class-transformer
```

## 1. Basic Validation

### DTO with Validation Decorators

```typescript
// create-cat.dto.ts
import { IsString, IsInt, Min, Max, IsOptional, IsEmail } from 'class-validator';

export class CreateCatDto {
  @IsString()
  name: string;

  @IsInt()
  @Min(0)
  @Max(30)
  age: number;

  @IsString()
  breed: string;

  @IsEmail()
  @IsOptional()
  ownerEmail?: string;
}
```

### Applying ValidationPipe

```typescript
// cats.controller.ts
import { Controller, Post, Body } from '@nestjs/common';
import { ValidationPipe } from '@nestjs/common';
import { CreateCatDto } from './dto/create-cat.dto';

@Controller('cats')
export class CatsController {
  @Post()
  create(@Body(new ValidationPipe()) createCatDto: CreateCatDto) {
    return this.catsService.create(createCatDto);
  }
}
```

### Global ValidationPipe

```typescript
// main.ts
import { NestFactory } from '@nestjs/core';
import { ValidationPipe } from '@nestjs/common';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  
  app.useGlobalPipes(
    new ValidationPipe({
      whitelist: true,      // Strip non-whitelisted properties
      forbidNonWhitelisted: true,  // Throw error if non-whitelisted
      transform: true,       // Transform payloads to DTO instances
      validationError: { target: true, value: true },
    }),
  );
  
  await app.listen(3000);
}
bootstrap();
```

## 2. Validation Decorators

### String Validators

```typescript
import {
  IsString,
  IsNotEmpty,
  IsIn,
  MinLength,
  MaxLength,
  Matches,
  IsAlpha,
  IsAlphanumeric,
  IsNumeric,
  IsLowercase,
  IsUppercase,
} from 'class-validator';

class UserDto {
  @IsString()
  name: string;

  @IsNotEmpty()
  username: string;

  @IsIn(['admin', 'user', 'guest'])
  role: string;

  @MinLength(8)
  @MaxLength(50)
  password: string;

  @Matches(/^[a-z]+$/)
  slug: string;

  @IsAlpha()
  country: string;

  @IsAlphanumeric()
  code: string;

  @IsNumeric()
  zipCode: string;

  @IsLowercase()
  email: string;

  @IsUppercase()
  countryCode: string;
}
```

### Number Validators

```typescript
import { IsNumber, Min, Max, IsPositive, IsNegative, IsFinite, IsInt } from 'class-validator';

class ProductDto {
  @IsNumber()
  price: number;

  @Min(0)
  @Max(1000000)
  stock: number;

  @IsPositive()
  quantity: number;

  @IsNegative()
  discount: number;

  @IsFinite()
  rating: number;

  @IsInt()
  count: number;
}
```

### Array Validators

```typescript
import { IsArray, ArrayMinSize, ArrayMaxSize, ArrayUnique } from 'class-validator';

class TagsDto {
  @IsArray()
  @ArrayMinSize(1)
  @ArrayMaxSize(5)
  @ArrayUnique()
  tags: string[];
}
```

### Date Validators

```typescript
import { IsDate, IsDateString, MinDate, MaxDate } from 'class-validator';

class EventDto {
  @IsDate()
  startDate: Date;

  @IsDateString()
  endDate: string;

  @MinDate(new Date('2024-01-01'))
  @MaxDate(new Date('2024-12-31'))
  eventDate: Date;
}
```

### Object Validators

```typescript
import { IsObject, ValidateNested } from 'class-validator';
import { Type } from 'class-transformer';

class AddressDto {
  @IsString()
  street: string;

  @IsString()
  city: string;
}

class UserDto {
  @IsObject()
  @ValidateNested()
  @Type(() => AddressDto)
  address: AddressDto;
}
```

### Custom Validators

```typescript
import {
  ValidatorConstraint,
  ValidatorConstraintInterface,
  ValidationArguments,
} from 'class-validator';

@ValidatorConstraint({ name: 'isStrongPassword', async: false })
export class IsStrongPasswordConstraint implements ValidatorConstraintInterface {
  validate(password: string, args: ValidationArguments) {
    return (
      password.length >= 8 &&
      /[A-Z]/.test(password) &&
      /[a-z]/.test(password) &&
      /[0-9]/.test(password) &&
      /[^A-Za-z0-9]/.test(password)
    );
  }

  defaultMessage(args: ValidationArguments) {
    return 'Password must be at least 8 characters long and contain uppercase, lowercase, numbers, and special characters';
  }
}

// Usage
import { IsStrongPassword } from './is-strong-password.decorator';

class RegisterDto {
  @IsStrongPassword()
  password: string;
}
```

## 3. Class-Transformer

### Transformation Decorators

```typescript
import {
  Transform,
  Expose,
  Exclude,
  Type,
  TransformFnParams,
} from 'class-transformer';

class UserDto {
  @Expose()
  id: number;

  @Expose()
  @Transform(({ value }) => value?.toUpperCase())
  username: string;

  @Exclude()
  password: string;

  @Expose()
  @Transform(({ value }) => new Date(value))
  createdAt: string;

  @Expose()
  @Type(() => AddressDto)
  address: AddressDto;
}
```

### Using Transform

```typescript
import { Transform } from 'class-transformer';

class ProductDto {
  @Transform(({ value }) => parseFloat(value))
  price: string;

  @Transform(({ value }) => value.split(','))
  tags: string;

  @Transform(({ value }) => value === 'true')
  isActive: string;
}
```

### Class Validation Options

```typescript
import { plainToInstance } from 'class-transformer';
import { validateSync } from 'class-validator';

function validate(data: any, dtoClass: any) {
  const dto = plainToInstance(dtoClass, data, { excludeExtraneousValues: true });
  const errors = validateSync(dto);

  if (errors.length > 0) {
    throw new BadRequestException(errors);
  }

  return dto;
}
```

## 4. Custom Validation Pipes

```typescript
import {
  PipeTransform,
  Injectable,
  ArgumentMetadata,
  BadRequestException,
} from '@nestjs/common';

@Injectable()
export class CustomValidationPipe implements PipeTransform<any> {
  transform(value: any, metadata: ArgumentMetadata) {
    const { metatype } = metadata;
    if (!metatype || !this.toValidate(metatype)) {
      return value;
    }
    const object = plainToInstance(metatype, value);
    const errors = validateSync(object);
    if (errors.length > 0) {
      throw new BadRequestException('Validation failed');
    }
    return object;
  }

  private toValidate(metatype: any): boolean {
    const types = [String, Boolean, Number, Array, Object];
    return !types.includes(metatype);
  }
}
```

## 5. Validation Groups

```typescript
import { ValidationGroup } from 'class-validator';

class UserDto {
  @IsString()
  @ValidateIf((o, v) => o.role === 'admin')
  adminKey: string;

  @IsString()
  @IsOptional()
  @ValidateIf((o) => o.updateProfile)
  bio: string;
}
```

## 6. Error Response Format

```typescript
// Exception filter for validation errors
@Catch(ValidationException)
export class ValidationExceptionFilter implements ExceptionFilter {
  catch(exception: ValidationException, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();
    const status = exception.getStatus();

    const errors = exception.getResponse() as any;

    response.status(status).json({
      statusCode: status,
      timestamp: new Date().toISOString(),
      path: ctx.getRequest().url,
      errors: errors.message || errors,
    });
  }
}
```

## Best Practices

1. **Always validate input data** - Never trust client input
2. **Use DTOs for validation** - Keep validation logic separate
3. **Provide clear error messages** - Help clients understand validation failures
4. **Use whitelist mode** - Strip unexpected properties
5. **Transform payloads** - Convert to proper DTO instances
6. **Create custom validators** - For complex validation logic
7. **Use validation groups** - For different validation rules in different contexts
8. **Test validation** - Write tests for validation logic

## References

- [NestJS Validation Documentation](https://docs.nestjs.com/techniques/validation)
- [class-validator Documentation](https://github.com/typestack/class-validator)
- [class-transformer Documentation](https://github.com/typestack/class-transformer)
