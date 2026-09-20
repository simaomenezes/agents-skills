---
name: clean-architecture-nestjs
description: Use when scaffolding NestJS modules, controllers, use-cases, or checking Clean Architecture dependency direction.
---

# Clean Architecture for NestJS

Reference: Uncle Bob's Clean Architecture (Entities -> Use Cases -> Interface Adapters -> Frameworks & Drivers). Dependency rule points inward.

## 1. Layer map

| Circle | Name | NestJS location | Contains |
|--------|------|-----------------|----------|
| Yellow center | Enterprise Business Rules | `src/domain/` | Entities, Value Objects, Aggregates, Domain Events, Repository interfaces |
| Red | Application Business Rules | `src/application/` | Use Cases (Interactors), Input/Output Ports, Application DTOs |
| Green | Interface Adapters | `src/interface/`, `src/application/` | Controllers, Presenters, Gateways, Mappers |
| Blue | Frameworks & Drivers | `src/infrastructure/`, `src/main.ts` | NestJS modules, TypeORM/Prisma, HTTP server, DB, external APIs |

Flow of control: `Controller -> Input Port -> UseCase Interactor -> Output Port -> Presenter`.

## 2. Allowed imports

```
interface/controllers  -> application/ports/in, application/dtos
application/use-cases  -> domain/*, application/ports/out
infrastructure/*       -> application/ports/*, domain/*
domain/*               -> nothing outside domain (std lib + domain errors only)
```

Forbidden: `domain` importing `typeorm`, `@nestjs/*`, `prisma`; `application` importing `infrastructure`; controller containing business `if`.

## 3. Canonical NestJS example

```ts
// src/domain/entities/order.ts - no decorators
export class Order { /* invariants + methods */ }

// src/application/ports/in/create-order.input-port.ts
export interface CreateOrderInputPort { execute(input: CreateOrderInput): Promise<CreateOrderOutput>; }

// src/application/use-cases/create-order.use-case.ts
@Injectable()
export class CreateOrderUseCase implements CreateOrderInputPort {
  constructor(@Inject(ORDER_REPOSITORY) private readonly orders: OrderRepository) {}
  async execute(input: CreateOrderInput) { /* orchestrate */ }
}

// src/interface/controllers/order.controller.ts - thin
@Controller('orders')
export class OrderController {
  constructor(@Inject(CreateOrderUseCase) private readonly createOrder: CreateOrderInputPort) {}
  @Post() create(@Body() dto: CreateOrderDto) { return this.createOrder.execute(dto); }
}

// src/infrastructure/di/order.module.ts
@Module({
  controllers: [OrderController],
  providers: [
    CreateOrderUseCase,
    { provide: ORDER_REPOSITORY, useClass: TypeOrmOrderRepository },
  ],
})
export class OrderModule {}
```

## 4. Anti-patterns to fix

- Anemic TypeORM `@Entity()` used as domain entity -> split into `Order` (domain) + `OrderOrmEntity` (infra) + mapper.
- Business logic in `@Controller()` -> move to UseCase.
- UseCase doing `new PrismaClient()` or `injectRepository()` directly -> depend on `OrderRepository` port.
- Presenter returning ORM object -> map to view model DTO.

## 5. Checklist before done

- [ ] `domain/` has zero `@nestjs/*` / `typeorm` imports (`grep -r "@nestjs\|typeorm\|prisma" src/domain` empty).
- [ ] Each use case has explicit Input + Output Port.
- [ ] Controller depends on Input Port token, not concrete class.
- [ ] Module wires ports via `provide/useClass`.
- [ ] Use case unit-testable with mocked outbound ports.
