---
name: nestjs-validation
description: NestJS data validation using class-validator and class-transformer with DTOs. Use when validating request payloads, creating typed DTOs, or implementing custom validators in NestJS.
license: MIT
metadata:
  author: snowmerak
  version: '1.0'
  category: nestjs
  tags: [validation, dto, class-validator, class-transformer]
---

# NestJS Validation - DTO-Based Request Validation

## Overview

NestJS의 `ValidationPipe`는 `class-validator`와 `class-transformer`를 사용하여 데코레이터 기반 검증과 타입 변환을 제공합니다. 모든 요청 데이터 검증을 위한 표준 패턴입니다.

---

## SOP: Step-by-Step Procedures

### SOP-1: 전역 ValidationPipe 설정 (필수)

```bash
npm install class-validator class-transformer
```

**main.ts에 반드시 적용:**
```typescript
async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  app.useGlobalPipes(new ValidationPipe({
    whitelist: true,               // DTO에 없는 필드 자동 삭제 (보안 필수)
    forbidNonWhitelisted: true,     // 모르는 필드가 있으면 400 에러 반환
    transform: true,                // plain object → DTO 인스턴스로 자동 변환
    disableErrorMessages: false,     // 프로덕션에서는 true로 변경 고려
  }));

  await app.listen(3000);
}
```

> ⚠️ `whitelist` + `forbidNonWhitelisted`는 **보안상 항상 함께 활성화**하세요. 하나만 켜면 공격자가 추가 필드를 주입할 수 있습니다.

### SOP-2: DTO 작성 (검증 데코레이터 활용)

```typescript
import { IsString, IsInt, Min, Max, IsEmail, IsOptional } from 'class-validator';
import { Transform } from 'class-transformer';
import { Type } from 'class-transformer';

export class CreateCatDto {
  @IsString()
  @MinLength(2)
  name: string;

  @IsInt()
  @Min(0)
  @Max(30)
  age: number;

  @IsOptional()                    // ← 없어도 에러 안 남
  @Transform(({ value }) => (value ? value.trim() : undefined))
  breed?: string;

  @IsEmail()                       // ← 이메일 형식 검증
  ownerEmail?: string;
}
```

### SOP-3: 자주 사용하는 검증 데코레이터

| 카테고리 | 데코레이터 | 설명 |
|----------|-----------|------|
| **String** | `@IsString()` | 문자열인지 확인 |
| | `@MinLength(n)`, `@MaxLength(n)` | 길이 제한 |
| | `@Matches(/regex/)` | 정규식 매칭 |
| | `@IsEmail()` | 이메일 형식 검증 |
| | `@IsURL()` | URL 형식 검증 |
| **Number** | `@IsInt()` | 정수인지 확인 |
| | `@Min(n)`, `@Max(n)` | 범위 제한 |
| | `@IsPositive()`, `@IsNegative()` | 부호 체크 |
| **Optional** | `@IsOptional()` | 필드 존재하지 않아도 에러 안 남 |
| | `@IsNotEmpty()` | 빈 문자열/null/undefined 금지 |
| **Array** | `@IsArray()` | 배열인지 확인 |
| | `@ArrayMinSize(n)`, `@ArrayMaxSize(n)` | 길이 제한 |
| | `@Type(() => ItemDto)` | 중첩 DTO 자동 변환 (필수!) |

### SOP-4: 중첩 객체 검증

```typescript
// ⚠️ @Type() 없으면 중첩 객체가 plain object로 남아서 검증 안 됨!
export class CreateOrderDto {
  @IsString() productId: string;

  @Type(() => AddressDto)          // ← 필수! 중첩 DTO 변환
  address: AddressDto;

  @IsArray()
  @Type(() => OrderItemDto)        // ← 배열의 각 요소도 DTO로 변환
  items: OrderItemDto[];
}

export class AddressDto {
  @IsString() street: string;
  @IsPostalCode('ko-KR') zipCode: string;
}
```

### SOP-5: 커스텀 검증 데코레이터

```typescript
// 1. RegisterDecorator 패턴 (재사용 가능)
export function IsPhoneNumber(validationGroups?: string[]) {
  return validateBy({ name: 'isPhoneNumber', validator: isKoreanPhone }, validationGroups);
}

// 2. Constraint 클래스 직접 구현
@ValidateIf(o => o.password !== o.confirmPassword, { message: '비밀번호가 일치하지 않습니다' })
confirmPassword?: string;

// 또는 ValidationArguments를 받는 함수
registerDecorator('IsStrongPassword', value: unknown, args: ValidationArguments) {
  const isStrong = (v: string) => /^.{8,}/.test(v);
  validateOrTransform(value, isStrong, args);
}, []);
```

### SOP-6: 검증 에러 응답 포맷

ValidationPipe가 실패하면 표준 HTTP 400 응답을 반환합니다:

```json
{
  "statusCode": 400,
  "message": ["name must be a string", "age must not be greater than 30"],
  "error": "Bad Request"
}
```

**커스텀 에러 메시지:**
```typescript
@IsString({ message: '이름은 문자열이어야 합니다' })
name: string;

@Min(0, { message: '나이는 0 이상이어야 합니다' })
age: number;
```

---

## Tool Integration

| 작업 | 도구 | 예시 |
|------|------|------|
| DTO 파일 탐색 | `search_files` | `search_files("class.*Dto", "src/**/*.dto.ts")` |
| 검증 설정 확인 | `read_file` | `main.ts`의 `useGlobalPipes` 설정 읽기 |
| 누락된 DTO 발견 | `search_files` | `@Body()` 데코레이터가 있는 메서드에서 DTO 타입 확인 |

---

## Anti-Patterns & Guardrails

- ❌ **`whitelist: true`, `forbidNonWhitelisted: true` 없이 ValidationPipe 사용 금지** — 공격자가 추가 필드를 주입할 수 있음
- ❌ **Controller에서 직접 검증 로직 작성 금지** — DTO 데코레이터로 처리하세요
- ❌ **중첩 객체에 `@Type(() => NestedDto)` 누락 금지** — 없으면 중첩 객체가 plain object로 남아 검증이 안 됨
- ❌ **`any` 타입 DTO 사용 금지** — 명확한 타입 정의가 검증의 전제 조건입니다
- ⚠️ **`transform: true`는 성능에 약간의 오버헤드가 있음** — 하지만 안전성을 위해 항상 활성화 권장

## Best Practices

1. `main.ts`에서 ValidationPipe 글로벌 설정 필수 (`whitelist` + `forbidNonWhitelisted` + `transform`)
2. 모든 요청 데이터는 DTO를 통해 검증
3. 중첩 객체/배열은 반드시 `@Type(() => ChildDto)` 사용
4. 민감한 필드(비밀번호 등)는 DTO에서 제외하거나 `@Exclude()` 적용
5. 커스텀 검증 로직은 재사용 가능한 데코레이터로 캡슐화

## References

- [NestJS Validation Docs](https://docs.nestjs.com/techniques/validation)
- [class-validator Documentation](https://github.com/typestack/class-validator)
- [class-transformer Documentation](https://github.com/typestack/class-transformer)
