---
name: nestjs-middleware
description: NestJS request processing pipeline — middleware, guards, interceptors, exception filters, pipes, and custom decorators. Use when implementing cross-cutting concerns like logging, auth, validation, error handling in NestJS.
license: MIT
metadata:
  author: snowmerak
  version: '1.0'
  category: nestjs
  tags: [middleware, guards, interceptors, pipes, filters, decorators]
---

# NestJS Request Pipeline - Cross-Cutting Concerns

## Overview

NestJS는 요청을 처리하는 명확한 파이프라인을 제공합니다:

```
Request → Middleware → Guards → Interceptors (Before) → Pipes → Controller
         ↓                           ↓                      ↓
      Logging, CORS           Auth, Roles          Transform/Validate
                                                       ↓
Response ← Exception Filters ← Interceptors (After) ← Controller Return
```

각 단계의 역할과 사용 시기를 명확히 이해해야 합니다.

---

## SOP: Step-by-Step Procedures

### SOP-1: Middleware — 로깅 & 전처리

**언제 쓰나요?** 요청 공통 처리(로깅, CORS, Body Parser)

```typescript
// 1. NestMiddleware 인터페이스 구현
@Injectable()
export class LoggerMiddleware implements NestMiddleware {
  use(req: Request, res: Response, next: NextFunction) {
    console.log(`${req.method} ${req.url}`);
    next();                    // ← 반드시 호출! 안 하면 응답이 멈춤
  }
}

// 2. Module에 적용 (NestModule 구현)
@Module({})
export class AppModule implements NestModule {
  configure(consumer: MiddlewareConsumer) {
    consumer.apply(LoggerMiddleware).forRoutes('*');        // 모든 라우트
    consumer.apply(LoggerMiddleware).forRoutes('cats');     // 특정 라우트
    consumer.apply(LoggerMiddleware).forRoutes({            // 조건부 적용
      path: 'cats', method: RequestMethod.GET,
    });
  }
}

// 3. 글로벌 설정 (main.ts) — Express 미들웨어도 가능
app.enableCors({ origin: '*' });
app.use(express.json());
```

**⚠️ Middleware는 DI를 사용하지 못합니다.** (`@Inject` 불가) → 복잡한 의존성이 필요하면 Guard/Interceptor 사용하세요.

### SOP-2: Guards — 인증 & 인가

**언제 쓰나요?** 요청이 Controller에 도달하기 전 허가 여부 결정

```typescript
// 1. CanActivate 구현
@Injectable()
export class AuthGuard implements CanActivate {
  canActivate(context: ExecutionContext): boolean {
    const request = context.switchToHttp().getRequest();
    return validateToken(request.headers.authorization);  // true면 통과, false면 403
  }
}

// 2. 적용 범위 (작은 순부터)
@UseGuards(AuthGuard)                       // 메서드 레벨
@Get() findAdmins() {}

@UseGuards(RolesGuard(['admin']))           // 컨트롤러 레벨 (모든 라우트 공통)
@Controller('cats') class CatsController {}

app.useGlobalGuards(new AuthGuard());       // 글로벌 (모든 컨트롤러)
```

**ExecutionContext로 데이터 꺼내기:**
- `context.switchToHttp().getRequest()` — HTTP 요청 객체
- `context.switchToRpc().getContext()` — RPC/Microservice
- `context.getHandler()` — 핸들러 메서드
- `context.getClass()` — 컨트롤러 클래스

### SOP-3: Interceptors — Before/After 처리

**언제 쓰나요?** 응답 변환, 로깅, 캐싱, 타임아웃

```typescript
// 1. NestInterceptor 구현 — next.handle()이 핵심
@Injectable()
export class TransformInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    return next.handle().pipe(
      map(data => ({ statusCode: 200, timestamp: new Date(), data })),
    );
  }
}

// 2. 요청 시간 측정 예시
@Injectable()
export class TimingInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const start = Date.now();
    return next.handle().pipe(
      tap(() => console.log(`${context.getHandler().name}: ${Date.now() - start}ms`)),
    );
  }
}

// 3. 적용
@UseInterceptors(TransformInterceptor)     // 메서드/컨트롤러 레벨
app.useGlobalInterceptors(new TimingInterceptor());  // 글로벌
```

**⚠️ Interceptor는 rxjs Observable을 반환합니다.** `next.handle()`의 반환값을 반드시 `.pipe()`로 가공하세요.

### SOP-4: Exception Filters — 에러 처리 표준화

**언제 쓰나요?** 일관된 에러 응답 포맷 제공

```typescript
// 1. @Catch() + ExceptionFilter 구현
@Catch(HttpException)
export class HttpErrorFilter implements ExceptionFilter {
  catch(exception: HttpException, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const res = ctx.getResponse<Response>();
    const status = exception.getStatus();

    res.status(status).json({
      statusCode: status,
      timestamp: new Date().toISOString(),
      path: ctx.getRequest<Request>().url,
      message: exception.message,
    });
  }
}

// 2. 여러 예외 클래스 캐치
@Catch(UnauthorizedException, ForbiddenException)
export class AuthErrorFilter implements ExceptionFilter { /* ... */ }

// 3. 적용 (글로벌 권장 — 에러 포맷은 일관되어야 함)
app.useGlobalFilters(new HttpErrorFilter());
```

**예외 계층:** `HttpException` ← `BadRequestException`, `UnauthorizedException`, `NotFoundException`, 등

### SOP-5: Pipes — 데이터 변환 & 검증

**언제 쓰나요?** 요청 데이터 검증, 타입 변환

```typescript
// 1. 전역 ValidationPipe 설정 (가장 중요!)
app.useGlobalPipes(new ValidationPipe({
  whitelist: true,                // ← DTO에 없는 필드 자동 삭제
  forbidNonWhitelisted: true,     // ← 있는 경우 400 에러
  transform: true,                // ← plain object → DTO 인스턴스 변환
}));

// 2. 빌트인 파이프 — 파라미터 레벨 적용
@Get(':id')
findOne(@Param('id', ParseIntPipe) id: number) {}        // 문자열 → 숫자

@Get()
findAll(@Query('page', new DefaultValuePipe(1), ParseIntPipe) page: number) {}  // 기본값 + 변환

// 3. 커스텀 파이프 — 복잡한 검증 로직 필요 시
@Injectable()
export class UuidPipe implements PipeTransform<string, string> {
  transform(value: string): string {
    if (!isUUID(value)) throw new BadRequestException('Invalid UUID');
    return value;
  }
}

@Get(':id')
findOne(@Param('id', UuidPipe) id: string) {}
```

**⚠️ `whitelist` + `forbidNonWhitelisted`는 항상 함께 사용하세요.** 하나만 켜면 보안 우회 가능.

### SOP-6: Custom Decorators — 파라미터 추출

```typescript
// 1. createParamDecorator 사용 (권장)
export const GetUser = createParamDecorator(
  (_data: unknown, ctx: ExecutionContext) => {
    return ctx.switchToHttp().getRequest().user;
  },
);

@Get('profile')
getProfile(@GetUser() user: UserEntity) { return user; }

// 2. 메타데이터 기반 데코레이터 (Guard와 연동 시)
export const Roles = (...roles: string[]) => SetMetadata('roles', roles);

@Roles('admin', 'moderator')           // Guard에서 Reflect.getMetadata('roles')로 읽음
@Get() findAll() {}
```

---

## Tool Integration

| 작업 | 도구 | 예시 |
|------|------|------|
| 파이프라인 요소 검색 | `search_files` | `search_files("implements CanActivate", "*.ts")` |
| 글로벌 설정 확인 | `read_file` | `main.ts`에서 `useGlobalPipes`, `useGlobalGuards` 등 확인 |
| 모듈 내 미들웨어 확인 | `read_file` | `AppModule`의 `configure()` 메서드 읽기 |

---

## Anti-Patterns & Guardrails

- ❌ **Middleware에 비즈니스 로직 넣지 마세요** — DI를 못 써서 테스트 불가능합니다. Guard/Interceptor 사용
- ❌ **Middleware에서 `next()` 호출하지 않으면 응답이 멈춥니다** — 항상 `next()` 호출
- ❌ **ValidationPipe 옵션 누락 금지** — 최소한 `whitelist: true`, `forbidNonWhitelisted: true` 활성화
- ❌ **Interceptor에서 Observable을 반환하지 않으면 에러 발생** — `return next.handle().pipe(...)` 패턴 필수
- ⚠️ **Exception Filter는 HttpException만 캐치하지 않으면 일반 JS Error가 누수됨** — 모든 예외를 캐치하는 필터를 가장 마지막에 배치

## Best Practices

1. Middleware → 로깅, CORS 등 경량 전처리만
2. Guards → 인증/인가 (허가 여부 판단)
3. Interceptors → 응답 변환, 시간 측정, 캐싱
4. Exception Filters → 일관된 에러 포맷 (글로벌 권장)
5. Pipes → 데이터 검증, 타입 변환 (`ValidationPipe` 글로벌 설정 필수)
6. Custom Decorators — 반복되는 파라미터 추출 로직 캡슐화

## References

- [NestJS Middleware](https://docs.nestjs.com/middleware)
- [NestJS Guards](https://docs.nestjs.com/guards)
- [NestJS Interceptors](https://docs.nestjs.com/interceptors)
- [NestJS Exception Filters](https://docs.nestjs.com/exception-filters)
- [NestJS Pipes](https://docs.nestjs.com/pipes)
- [NestJS Custom Decorators](https://docs.nestjs.com/custom-decorators)
