---
name: jwt-security-hardening
description: Use when hardening JWT config, secrets, expiry, revocation, RS256/JWKS in NestJS.
---

# JWT Security Hardening for NestJS

Reference Debugger workflow from https://www.jwt.io: paste token -> verify signature with secret/public key -> inspect `alg, exp, iss, aud, jti`.

Enforces `solid-principles-nestjs` (DIP via ports, OCP via key rotation) and `clean-architecture-nestjs` (crypto only in `infrastructure/`).

## 1. Algorithm decision

- **HS256:** single secret, simple microservice. Secret >=256 bits (`openssl rand -base64 32`), stored in env/vault, never committed. Good default for `@nestjs/jwt` monolith.
- **RS256:** private signs, public verifies. Use for multi-service / third-party verifiers, JWKS endpoint, `kid` header for rotation. Private key never in repo; public via JWKS with cache.
- Never accept `alg: none`. Pin `algorithms: ['HS256']` or `['RS256']` in Strategy + `verifyAsync`. Reject tokens with unexpected `kid`.

## 2. Claims matrix

| Claim | Required | Value |
|-------|----------|-------|
| `exp` | yes | access 5-15m, refresh 7d max with rotation |
| `iat` | yes | issued-at, check clock skew via `clockTolerance: 5` |
| `iss` | yes | e.g. `my-api`, validated on verify |
| `aud` | yes | e.g. `my-app`, validated on verify |
| `sub` | yes | userId (UUID, not email) |
| `jti/sid` | refresh yes | session binding, denylist on logout |

## 3. Secrets + config

```ts
// infrastructure/config/jwt.config.ts
export default registerAs('jwt', () => ({
  accessSecret: getOrThrow('JWT_ACCESS_SECRET'),
  refreshSecret: getOrThrow('JWT_REFRESH_SECRET'),
  issuer: 'my-api',
  audience: 'my-app',
}));
```

Validate startup: length >=32 chars, different access vs refresh, production `Secure; HttpOnly; SameSite` cookies for browser flows.

## 4. Revocation

- Stateless access (short TTL, no DB hit) + stateful refresh (DB/redis by `jti` hash).
- Logout: delete/revoke session row + optional access denylist by `jti` until `exp` (redis TTL = remaining exp).
- Reuse detection: if presented refresh `jti` already rotated -> revoke session chain + alert.

## 5. Audit greps (fail on hit unless justified)

- `algorithm.*none|algorithms.*\*` (wildcard alg)
- `secret:\s*['"]secret|123|test|changeme` (weak/hardcoded)
- `expiresIn:\s*['"][0-9]+d` with access >1d (check context; refresh allowed 7d)
- `ignoreExpiration:\s*true`
- `fromUrlQueryParameter` (token in URL)
- `private.*key|BEGIN PRIVATE` in repo (leaked key)

## 6. jwt.io validation steps

1. Copy `issueAccess()` output, paste in jwt.io Debugger.
2. Enter secret/public key, confirm `Signature Verified`.
3. Confirm header `alg/typ`, payload `sub/iss/aud/exp/jti` sane.
4. Confirm expired token rejected (`exp` in past -> `TokenExpiredError`).

## Checklist

- [ ] Secrets via env/vault, rotated, access != refresh.
- [ ] `issuer/audience/algorithms/clockTolerance` set on sign + verify.
- [ ] Refresh hashed, rotated, revokable; logout revokes.
- [ ] No crypto in `domain/application` (DIP); key swap via `useClass` without UseCase edit (OCP).
- [ ] Findings reported as `| Risk | File:line | Fix |` + Clean/SOLID tables.
