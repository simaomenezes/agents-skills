---
name: ddd-tactical-nestjs
description: Use when modeling Entities, Value Objects, Aggregates, Repositories, Domain Events in NestJS TypeScript.
---

# DDD Tactical Patterns for NestJS

Scope: tactical modeling inside the Clean Architecture Entities circle. No framework code in `src/domain/`.

## 1. Ubiquitous Language + Bounded Contexts

- Name classes/methods from business talk. Prefer `Order.confirm()` over `OrderService.updateOrderStatus(1)`.
- One NestJS feature module per bounded context or aggregate (`OrderModule`, `BillingModule`). Do not share entities across contexts; use anti-corruption mappers.

## 2. Entities vs Value Objects

Entity: stable identity (`id`), mutable lifecycle, rich methods.
Value Object: immutable, compared by value, self-validating, no ID.

```ts
export class Money {
  private constructor(readonly cents: number, readonly currency: string) {}
  static of(cents: number, currency = 'BRL'): Money {
    if (cents < 0) throw new DomainError('Negative amount');
    return new Money(cents, currency);
  }
  add(other: Money): Money {
    if (other.currency !== this.currency) throw new DomainError('Currency mismatch');
    return new Money(this.cents + other.cents, this.currency);
  }
}
```

## 3. Aggregates

- One Aggregate Root guards invariants. Reference other aggregates by ID, not object graph.
- Factory method validates: `Order.create(id, items)` rejects empty items.
- Mutators enforce transitions: `confirm()` only from `DRAFT`.
- Keep aggregate small; large graphs -> eventual consistency via domain events.

```
src/domain/
  orders/order.ts (root)
  orders/order-item.ts
  orders/order.repository.ts (interface)
  orders/events/order-confirmed.event.ts
```

## 4. Repositories (ports)

Interface in domain, implementation in infrastructure:

```ts
// src/domain/orders/order.repository.ts
export interface OrderRepository {
  findById(id: string): Promise<Order | null>;
  save(order: Order): Promise<void>;
}

// src/infrastructure/persistence/order-orm.entity.ts -> TypeORM/Prisma model
// src/infrastructure/persistence/typeorm-order.repository.ts implements OrderRepository + mapper
```

Never return ORM entities from domain. Mapper converts both ways.

## 5. Domain Events

Plain classes, dispatched by application layer after `save()`:

```ts
export class OrderConfirmed { constructor(readonly orderId: string) {} }
```

Infrastructure subscribes (EventEmitter, message bus). Domain does not import bus.

## 6. Checklist

- [ ] Entities have factory + behavior methods, not public setters everywhere.
- [ ] VOs immutable with `static create/factory` validation.
- [ ] Aggregate references by ID across boundaries.
- [ ] Repository interface in `domain/`, concrete in `infrastructure/` + mapper.
- [ ] No `@Entity()`, `@Injectable()`, `class-validator` in `domain/`.
