---
name: nestjs-jwt-auth
description: Use when implementing login, refresh, guards, JwtStrategy with @nestjs/jwt and passport in NestJS.
---

# NestJS JWT Auth (@nestjs/jwt + passport)

JWT = `base64url(header).base64url(payload).signature` (see https://www.jwt.io Debugger: decode -> verify signature -> check exp/iss/aud). Claims: `sub (userId), sid/jti (sessionId), iat, exp, iss, aud`.

Strictly follows `clean-architecture-nestjs` and `solid-principles-nestjs` skills. Do not put framework types in `domain/` or `application/`.

## 1. Ports first (application)

```ts
export const TOKEN_ISSUER = Symbol('TOKEN_ISSUER');
export interface TokenIssuerPort {
  issueAccess(userId: string, sessionId: string): Promise<string>;
  issueRefresh(userId: string, sessionId: string, jti: string): Promise<string>;
}
export const TOKEN_VERIFIER = Symbol('TOKEN_VERIFIER');
export interface TokenVerifierPort {
  verifyAccess(token: string): Promise<{ sub: string; sid: string }>;
}
```

UseCases (`LoginUseCase`, `RefreshUseCase`) depend only on these + `SessionStorePort`. Testable with mocks.

## 2. Infrastructure adapter (only place with @nestjs/jwt)

```ts
@Injectable()
export class NestJwtIssuerAdapter implements TokenIssuerPort {
  constructor(private readonly jwt: JwtService, private readonly cfg: ConfigService) {}
  issueAccess(userId: string, sessionId: string) {
    return this.jwt.signAsync({ sub: userId, sid: sessionId },
      { secret: this.cfg.getOrThrow('JWT_ACCESS_SECRET'), expiresIn: '15m', issuer: 'my-api', audience: 'my-app', algorithm: 'HS256' });
  }
  issueRefresh(userId: string, sessionId: string, jti: string) {
    return this.jwt.signAsync({ sub: userId, sid: sessionId, jti },
      { secret: this.cfg.getOrThrow('JWT_REFRESH_SECRET'), expiresIn: '7d', issuer: 'my-api', audience: 'my-app', algorithm: 'HS256' });
  }
}
```

## 3. Strategy + Guard + Controller (interface)

```ts
@Injectable()
export class JwtStrategy extends PassportStrategy(Strategy) {
  constructor(cfg: ConfigService) {
    super({ jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(), secretOrKey: cfg.getOrThrow('JWT_ACCESS_SECRET'), issuer: 'my-api', audience: 'my-app', algorithms: ['HS256'], ignoreExpiration: false });
  }
  validate(payload: { sub: string; sid: string }) { return { userId: payload.sub, sessionId: payload.sid }; }
}

@Controller('auth')
export class AuthController {
  constructor(@Inject(LoginUseCase) private readonly login: LoginInputPort) {}
  @Public() @Post('login') loginRoute(@Body() dto: LoginDto) { return this.login.execute(dto); }
}
```

Support `@Public()` metadata + `JwtAuthGuard`, `@CurrentUser()` decorator returning `validate()` result. No business `if` here.

## 4. Refresh rotation (application rule)

1. Verify refresh, load session by `jti`, check reuse (if `jti` already rotated -> revoke whole session chain).
2. Generate new `jti`, save hashed (`sha256`), issue new pair, return.
3. Store refresh hashed only; compare with `timingSafeEqual`/hash compare.

## Anti-patterns

- `LoginUseCase` importing `JwtService` directly (DIP violation) -> use `TokenIssuerPort`.
- `@Entity()` session with JWT logic, or `Session` importing `JwtService`.
- Access `expiresIn: '7d'`, refresh without `jti`, secret `'secret'` hardcoded.
- Tokens logged or returned in URL query.

## Checklist

- [ ] `grep -r "JwtService|@nestjs/jwt" src/domain src/application` empty (infra only).
- [ ] `JwtModule.registerAsync` with env secrets, `issuer/audience`, `expiresIn 15m/7d`.
- [ ] Strategy validates `issuer/audience/algorithm`, `ignoreExpiration: false`.
- [ ] Refresh rotation + hashed storage + revocation on logout/reuse.
- [ ] SOLID table + Clean Compliance section filled.
- [ ] Validated sample token in jwt.io Debugger (signature verified, exp correct).
