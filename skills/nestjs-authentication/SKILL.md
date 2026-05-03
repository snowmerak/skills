---
name: nestjs-authentication
description: NestJS authentication strategies including JWT, local login, passport integration, and role-based authorization. Use when implementing user login, token management, or access control in NestJS applications.
license: MIT
metadata:
  author: snowmerak
  version: '1.0'
  category: nestjs
  tags: [authentication, jwt, passport, guards, authorization]
---

# NestJS Authentication - JWT, Passport & Authorization Patterns

## Overview

NestJS는 `@nestjs/passport`를 통해 Passport.js 전략을 쉽게 통합합니다. 이 스킬은 JWT 기반 인증과 로컬 로그인, 그리고 역할(Role) 기반 인가 패턴을 다룹니다.

---

## SOP: Step-by-Step Procedures

### SOP-1: 패키지 설치 및 Auth Module 설정

```bash
npm install @nestjs/jwt @nestjs/passport passport passport-jwt bcryptjs
npm install -D @types/bcryptjs
```

**AuthModule 구성:**
```typescript
@Module({
  imports: [
    UsersModule,                             // ← User 서비스 의존성
    PassportModule.register({ defaultStrategy: 'jwt' }),  // 전역 JWT 전략 설정
    JwtModule.registerAsync({
      useFactory: (config: ConfigService) => ({
        secret: config.get('JWT_SECRET'),
        signOptions: { expiresIn: config.get('JWT_EXPIRES_IN', '1d') },
      }),
      inject: [ConfigService],
      imports: [ConfigModule],
    }),
  ],
  providers: [AuthService, JwtStrategy, LocalStrategy],
  exports: [AuthService, JwtStrategy, PassportModule], // ← 다른 모듈에서 사용 가능
})
export class AuthModule {}
```

### SOP-2: JWT Strategy 구현

```typescript
import { Injectable } from '@nestjs/common';
import { PassportStrategy } from '@nestjs/passport';
import { ExtractJwt, Strategy } from 'passport-jwt';
import { ConfigService } from '@nestjs/config';

@Injectable()
export class JwtStrategy extends PassportStrategy(Strategy) {
  constructor(private config: ConfigService) {
    super({
      jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(), // ← Authorization: Bearer <token>
      ignoreExpiration: false,
      secretOrKey: config.get('JWT_SECRET'),
    });
  }

  async validate(payload: any) {
    return { userId: payload.sub, username: payload.username, role: payload.role };
  }
}
```

**⚠️ `validate()`의 반환값이 `req.user`로 자동 주입됩니다.** Payload에 필요한 모든 필드를 포함하세요.

### SOP-3: Auth Guard 생성 및 적용

```typescript
// jwt-auth.guard.ts — JWT 인증 Guard
import { Injectable } from '@nestjs/common';
import { AuthGuard } from '@nestjs/passport';

@Injectable()
export class JwtAuthGuard extends AuthGuard('jwt') {}

// roles.guard.ts — 역할 기반 인가 Guard
@Injectable()
export class RolesGuard implements CanActivate {
  canActivate(context: ExecutionContext): boolean {
    const requiredRoles = Reflector.get<string[]>('roles', context.getHandler());
    if (!requiredRoles) return true; // 데코레이터 없으면 통과

    const user = context.switchToHttp().getRequest().user;
    return requiredRoles.includes(user.role);
  }
}
```

**적용:**
```typescript
// 메서드 레벨
@UseGuards(JwtAuthGuard, RolesGuard)
@Roles('admin')                        // ← 커스텀 데코레이터로 역할 정의
@Get() findAllAdmins() {}

// 컨트롤러 레벨 (모든 라우트 공통)
@UseGuards(JwtAuthGuard)
@Controller('cats') class CatsController {}

// 글로벌 — main.ts에서 설정 시 login/register 등 공개 엔드포인트는 별도 처리 필요
app.useGlobalGuards(new JwtAuthGuard());
```

### SOP-4: Public 데코레이터 (공개 엔드포인트용)

전역 Guard를 적용했을 때 로그인/회원가입 라우트를 제외하려면 `SKIP_AUTH` 메타데이터 패턴을 사용하세요.

```typescript
// public.decorator.ts
import { SetMetadata } from '@nestjs/common';
export const IS_PUBLIC_KEY = 'isPublic';
export const Public = () => SetMetadata(IS_PUBLIC_KEY, true);

// public.guard.ts — AuthGuard 상속 + 예외 처리
@Injectable()
export class AuthGuard extends PassportAuthGuard('jwt') {
  constructor(private reflector: Reflector) { super(); }

  canActivate(context: ExecutionContext): boolean {
    const isPublic = this.reflector.getAllAndOverride<boolean>(IS_PUBLIC_KEY, [
      context.getHandler(),
      context.getClass(),
    ]);
    if (isPublic) return true; // 공개 라우트면 인증 스킵
    return super.canActivate(context);
  }
}

// 사용: @Public() 데코레이터 붙인 라우트는 JWT 토큰 불필요
@Public() @Post('login') login(@Body() dto) { /* ... */ }
```

### SOP-5: AuthService 구현 (로그인 + 회원가입)

```typescript
@Injectable()
export class AuthService {
  constructor(
    private usersService: UsersService,
    private jwtService: JwtService,
  ) {}

  async validateUser(username: string, password: string): Promise<User | null> {
    const user = await this.usersService.findByUsername(username);
    if (user && await bcrypt.compare(password, user.password)) {
      const { password: _, ...result } = user; // ← 비밀번호 제외!
      return result;
    }
    return null;
  }

  async login(dto: LoginDto): Promise<{ access_token: string }> {
    const user = await this.validateUser(dto.username, dto.password);
    if (!user) throw new UnauthorizedException('Invalid credentials');

    const payload = { username: user.username, sub: user.id, role: user.role };
    return { access_token: this.jwtService.sign(payload) };
  }

  async register(dto: RegisterDto): Promise<{ access_token: string }> {
    const user = await this.usersService.create({
      ...dto,
      password: await bcrypt.hash(dto.password, 10), // ← 해싱 필수!
    });
    const payload = { username: user.username, sub: user.id, role: user.role };
    return { access_token: this.jwtService.sign(payload) };
  }
}
```

---

## Tool Integration

| 작업 | 도구 | 예시 |
|------|------|------|
| Guard 파일 탐색 | `search_files` | `search_files("CanActivate", "*.guard.ts")` |
| JWT 설정 확인 | `read_file` | `.env`, `auth.module.ts`의 secret/expiresIn |
| 전략 파일 확인 | `list_dir` | `src/auth/strategies/` 폴더 구조 |

---

## Anti-Patterns & Guardrails

- ❌ **평문 비밀번호 저장 절대 금지** — 항상 `bcrypt.hash()` 사용. 비교 시 `bcrypt.compare()` 필수
- ❌ **JWT Payload에 민감 정보 포함 금지** — ID/Role 정도만 넣으세요. 전체 User 객체 넣지 마세요
- ❌ **`validateUser`에서 비밀번호 리턴 금지** — `const { password, ...result } = user;`로 반드시 제거
- ❌ **Access Token 만료기간 너무 길게 설정 금지** — 15분~1시간 권장. Refresh Token 패턴 사용
- ❌ **Global Guard 적용 시 Public 엔드포인트 처리 안 함** — `@Public()` 데코레이터 + `IS_PUBLIC_KEY` 체크 필수

## Best Practices

1. JWT Secret은 환경 변수에서 관리 (`JWT_SECRET`)
2. 비밀번호 항상 bcrypt 해싱 (라운드 10~12 권장)
3. Role 기반 인가는 `RolesGuard` + `@Roles('admin')` 패턴
4. Refresh Token으로 Access Token 갱신 (리프레시 토큰 별도 저장소 관리)
5. 공개 엔드포인트는 `@Public()` 데코레이터로 명시

## References

- [NestJS Authentication Docs](https://docs.nestjs.com/security/authentication)
- [Passport.js Documentation](http://www.passportjs.org)
- [JWT Best Practices](https://auth0.com/docs/secure/tokens/json-web-tokens)
