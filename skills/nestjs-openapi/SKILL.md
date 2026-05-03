---
name: nestjs-openapi
description: NestJS OpenAPI (Swagger) integration for automatic API documentation generation. Use when setting up Swagger UI, documenting controllers/DTOs, or creating interactive API specs in NestJS.
license: MIT
metadata:
  author: snowmerak
  version: '1.0'
  category: nestjs
  tags: [openapi, swagger, api-docs, documentation]
---

# NestJS OpenAPI - Swagger Documentation Generation

## Overview

NestJS는 `@nestjs/swagger` 패키지를 통해 코드 데코레이터로 자동으로 OpenAPI 3.0 문서를 생성합니다. Swagger UI를 설정하면 브라우저에서 대화형 API 테스트가 가능합니다.

---

## SOP: Step-by-Step Procedures

### SOP-1: 설치 및 기본 설정

```bash
npm install @nestjs/swagger swagger-ui-express
```

**main.ts에 통합 (권장):**
```typescript
import { NestFactory } from '@nestjs/core';
import { SwaggerModule, DocumentBuilder } from '@nestjs/swagger';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  // 1. 문서 설정 정의
  const config = new DocumentBuilder()
    .setTitle('My API')
    .setDescription('API Documentation')
    .setVersion('1.0')
    .addTag('cats', 'Cat management endpoints')
    .addBearerAuth({                           // ← JWT 인증 데모용
      type: 'http', scheme: 'bearer', bearerFormat: 'JWT',
    })
    .build();

  // 2. 문서 생성 및 Swagger UI 설정
  const document = SwaggerModule.createDocument(app, config);
  SwaggerModule.setup('api-docs', app, document, {                   // ← /api-docs 경로에서 접근
    swaggerOptions: { persistAuthorization: true },
  });

  await app.listen(3000);
}
bootstrap();
```

**⚠️ `SwaggerModule.setup()`은 반드시 `app.listen()` 전에 호출해야 합니다.**

### SOP-2: Controller & DTO 문서화

```typescript
// 1. Controller — @ApiTags, @ApiOperation, @ApiResponse 사용
import { ApiTags, ApiOperation, ApiResponse } from '@nestjs/swagger';
import { CreateCatDto } from './dto/create-cat.dto';
import { CatResponseDto } from './dto/cat-response.dto';

@ApiTags('cats')                           // ← Swagger UI에서 그룹화
@Controller('cats')
export class CatsController {
  constructor(private catsService: CatsService) {}

  @Post()
  @ApiOperation({ summary: '새 고양이 추가' })          // ← 한국어/영어 요약
  @ApiResponse({ status: 201, description: '성공', type: CatResponseDto })
  @ApiResponse({ status: 400, description: '검증 실패' })
  create(@Body() dto: CreateCatDto) { return this.catsService.create(dto); }

  @Get(':id')
  @ApiOperation({ summary: '고양이 조회' })
  @ApiResponse({ status: 200, type: CatResponseDto })
  @ApiResponse({ status: 404, description: '존재하지 않음' })
  findOne(@Param('id') id: string) { return this.catsService.findOne(id); }
}
```

### SOP-3: DTO 필드 문서화 (`@ApiProperty`)

```typescript
import { IsString, IsInt, Min, Max, IsOptional } from 'class-validator';
import { ApiProperty, ApiPropertyOptional } from '@nestjs/swagger';

export class CreateCatDto {
  @ApiProperty({
    description: '고양이 이름',
    example: 'Kitty',
    minLength: 2,
  })
  @IsString() name: string;

  @ApiProperty({
    description: '나이 (0~30)',
    example: 3,
    minimum: 0, maximum: 30,
  })
  @IsInt() age: number;

  @ApiPropertyOptional({                   // ← ? 필드는 Optional 사용
    description: '품종 (선택사항)',
    example: 'Persian',
  })
  @IsOptional() breed?: string;
}
```

**⚠️ `@ApiProperty`는 반드시 `class-validator` 데코레이터 위에 배치하세요.** 순서가 중요합니다.

### SOP-4: nest-cli.json에 Swagger 플러그인 등록 (자동 생성)

데코레이터를 최소화하고 주석에서 자동으로 OpenAPI 스키마를 생성하려면 `nest-cli.json`을 수정합니다.

```json
{
  "compilerOptions": {
    "plugins": [
      {
        "name": "@nestjs/swagger",
        "options": {
          "classValidatorShim": true,       // class-validator 데코레이터를 Swagger로 자동 매핑
          "introspectComments": true         // JSDoc 주석에서 description 추출
        }
      }
    ]
  }
}
```

**이 설정 후엔 이보다 더 간단해집니다:**
```typescript
export class CreateCatDto {
  /** 고양이 이름 */                    // ← JSDoc 주석이 자동으로 description이 됨
  @IsString() name: string;

  /** 나이 (0~30) */                    // ← @ApiProperty 필요 없음!
  @IsInt() age: number;
}
```

### SOP-5: Enum & 중첩 응답 문서화

```typescript
// Enum 자동 스키마 생성
export enum CatStatus { ACTIVE = 'active', INACTIVE = 'inactive' }

@ApiProperty({ enum: CatStatus, example: CatStatus.ACTIVE })
status: CatStatus;

// 배열 응답 (중첩)
@ApiResponse({ status: 200, type: [CatResponseDto] }) // ← []로 배열 표시
findAll() { return this.catsService.findAll(); }
```

### SOP-6: 환경별 Swagger UI 토글

```typescript
if (process.env.NODE_ENV !== 'production') {          // ← 개발/스테이징에서만 활성화
  const config = new DocumentBuilder().setTitle('API').build();
  const document = SwaggerModule.createDocument(app, config);
  SwaggerModule.setup('api-docs', app, document);
}
```

---

## Tool Integration

| 작업 | 도구 | 예시 |
|------|------|------|
| 데코레이터 사용 확인 | `search_files` | `search_files("@ApiProperty", "*.dto.ts")` |
| Swagger 설정 읽기 | `read_file` | `main.ts`, `nest-cli.json` 플러그인 설정 |
| Swagger UI 접근 | 브라우저 | `http://localhost:3000/api-docs` 확인 |

---

## Anti-Patterns & Guardrails

- ❌ **프로덕션에서 Swagger UI 노출 금지** — 환경 변수 체크로 비활성화 필수
- ❌ **`@ApiResponse` 누락 시 클라이언트가 예상할 수 없는 응답 타입 노출** — 최소한 200/400/401/500은 명시
- ❌ **JWT secret을 Swagger 예시에 포함 금지** — `example` 필드에 실제 토큰값 넣지 마세요
- ⚠️ **`classValidatorShim: true` 설정 시에도 복잡한 검증 로직은 수동 `@ApiProperty` 추가 필요**

## Best Practices

1. `nest-cli.json`에 Swagger 플러그인 등록 → 데코레이터 최소화, JSDoc 활용
2. 전역 인증(`addBearerAuth`) 설정으로 모든 엔드포인트에서 테스트 가능
3. DTO 응답 타입 명시 (`type: ResponseDto`) — 클라이언트 코드 생성 용이
4. 환경변수로 프로덕션 Swagger UI 비활성화

## References

- [NestJS Swagger Docs](https://docs.nestjs.com/openapi/introduction)
- [@nestjs/swagger GitHub](https://github.com/nestjs/swagger)
- [OpenAPI Specification](https://swagger.io/specification/)
