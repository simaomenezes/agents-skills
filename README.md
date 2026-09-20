# agents-skills

Reusable **opencode agents + skills** for **NestJS** and **React (Vite + TS)** following **Clean Architecture (Uncle Bob)**, **SOLID**, **DDD tactical patterns**, and **JWT auth** (`jwt.io` / RFC 7519).

## References

- Clean Architecture diagram: https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html
- JWT Debugger / concepts (`header.payload.signature`, claims `sub/iat/exp/iss/aud/jti`): https://www.jwt.io
- Chakra UI component system (`Provider`, tokens, recipes): https://chakra-ui.com/

## Structure

```
.opencode/
  agents/                          # subagents (mode: subagent)
    nestjs-clean-architect.md      # scaffolds/refactors NestJS modules into 4 layers
    nestjs-domain-modeler.md       # Entities, Value Objects, Aggregates (pure TS)
    nestjs-use-case-designer.md    # Interactors + Input/Output Ports
    nestjs-solid-reviewer.md       # read-only SOLID audit (edit: deny)
    nestjs-jwt-auth-architect.md   # JWT access+refresh scaffold (Clean + SOLID checks)
    nestjs-jwt-security-reviewer.md# read-only JWT security audit (edit: deny)
    react-clean-architect.md       # scaffolds/refactors React features into 4 layers
    react-presenter-designer.md    # controller hooks, presenters, gateways, view models
    react-solid-reviewer.md        # read-only React SOLID + boundary audit (edit: deny)
  skills/                          # SKILL.md, name == folder
    clean-architecture-nestjs/     # layer map, allowed/forbidden imports, module example
    solid-principles-nestjs/       # SRP/OCP/LSP/ISP/DIP with NestJS snippets
    ddd-tactical-nestjs/           # aggregates, repos (interface in domain), events
    nestjs-jwt-auth/               # @nestjs/jwt + passport, guards, refresh rotation
    jwt-security-hardening/        # secrets, expiry, revocation, RS256/JWKS, audit greps
    clean-architecture-react/      # React layer map (Vite+TS, Query, Hook Form+Zod, Chakra at edge)
    solid-principles-react/        # SRP/OCP/LSP/ISP/DIP with React + Chakra snippets
opencode.json                      # $schema + skills.paths: [".opencode/skills"]
```

## Layer model enforced

| Circle | NestJS location |
|--------|-----------------|
| Entities (yellow) | `src/domain/` — pure TS, no `@nestjs/*`, no ORM |
| Use Cases (red) | `src/application/` — Interactors + `ports/in`, `ports/out` |
| Interface Adapters (green) | `src/interface/` — Controllers, Presenters, Gateways |
| Frameworks & Drivers (blue) | `src/infrastructure/` — NestJS modules, TypeORM/Prisma, JWT, DB |

Flow: `Controller -> InputPort -> Interactor -> OutputPort -> Presenter`
Rule: inner layers know nothing of outer; cross-boundary only via Port interfaces + DTOs + DI tokens (`provide/useClass`).

JWT binding: `domain` holds `Session`/`UserId` only; `application` holds `Login/Refresh/Logout UseCases + TokenIssuer/Verifier/SessionStore ports`; `infrastructure` is the only place with `@nestjs/jwt`; refresh tokens stored hashed with `jti` rotation + reuse detection.

React binding (Vite + TS, TanStack Query, Hook Form + Zod, Chakra UI): `domain` pure TS; `application` framework-free use cases + gateway ports; `interface` controller hooks (`useXController`) + presenters (`toXViewModel`); `infrastructure` HTTP gateways + Zod schemas + Query wrappers; `ui` dumb Chakra components/pages + `ui/theme` (`Provider`, `defineTokens`, `defineRecipe`). Flow: `View -> Controller hook -> InputPort -> Interactor -> Gateway -> Presenter -> View`. Chakra renders ViewModels only (`Stack/Field/Input/Button/Spinner/Alert/EmptyState`), no business logic in handlers/props.

## Usage

1. Clone this repo or copy `.opencode/` + `opencode.json` into your NestJS project.
2. Ensure `opencode.json` keeps `"$schema": "https://opencode.ai/config.json"` and `skills.paths`.
3. **Quit and restart opencode** (config is loaded once at startup).
4. Invoke e.g. `nestjs-clean-architect` to scaffold a module, `nestjs-jwt-auth-architect` for `AuthModule`, `react-clean-architect` for a React feature, or `nestjs-solid-reviewer` / `react-solid-reviewer` / `nestjs-jwt-security-reviewer` for audits. Skills trigger automatically on matching prompts (`login`, `refresh`, `guard`, `Scaffold module`, `review SOLID`, `React hook`, `presenter`, etc.).

## Verification

- Skill `name` matches folder, agents use `mode: subagent` (reviewers add `permission: edit: deny`).
- `domain/` must have zero `typeorm/prisma/@nestjs/passport/@nestjs/jwt` imports; `application/` must not import `JwtService` concrete or HTTP types.
- React: `domain/` zero `react/@tanstack/axios/zod`; `application/` zero `react/@tanstack/axios/fetch`; no `fetch` outside `infrastructure/`, no business `if` in `.tsx`/controller hooks.
- Validate sample JWT in the jwt.io Debugger (signature verified, `exp/iss/aud` correct).
