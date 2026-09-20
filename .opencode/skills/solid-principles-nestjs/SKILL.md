---
name: solid-principles-nestjs
description: Use when reviewing or writing NestJS providers, services, controllers for SRP OCP LSP ISP DIP compliance.
---

# SOLID for NestJS

Apply per principle with minimal NestJS examples. Prefer ports + DI tokens.

## S - Single Responsibility

One reason to change per class.
Bad: `UserService` with `register() + chargeCard() + sendEmail() + generateReport()`.
Good: `RegisterUserUseCase`, `BillingService` (port), `MailerPort`, `UserReportPresenter` split across `application/` and `infrastructure/`.
Rule: Controller = HTTP only. UseCase = orchestration only. Entity = invariants only.

## O - Open / Closed

Open for extension, closed for modification.
Bad:
```ts
if (type === 'stripe') { /* ... */ } else if (type === 'pix') { /* ... */ } // edit core each time
```
Good:
```ts
export interface PaymentGateway { charge(amount: number): Promise<void>; }
@Injectable() export class StripeGateway implements PaymentGateway {}
@Injectable() export class PixGateway implements PaymentGateway {}
// Module picks implementation, UseCase unchanged
{ provide: PAYMENT_GATEWAY, useClass: StripeGateway }
```

## L - Liskov Substitution

Any port implementation must be substitutable.
Bad: `class InMemoryRepo implements OrderRepository { findById() { throw new Error('todo'); } }`.
Good: full contract honored, same input/output types, no narrowed preconditions. Test by swapping `useClass` in module without changing callers.

## I - Interface Segregation

Small client-specific ports.
Bad: `interface UserRepo { findById(); save(); findByEmail(); listAdmins(); exportCsv(); /* ... 20 methods */ }` forced on every caller.
Good:
```ts
export interface FindUserByIdPort { findById(id: string): Promise<User | null>; }
export interface SaveUserPort { save(user: User): Promise<void>; }
```
UseCases inject only what they need.

## D - Dependency Inversion

High-level modules depend on abstractions.
Bad:
```ts
import { PrismaClient } from '@prisma/client';
export class CreateOrderUseCase {
  private prisma = new PrismaClient(); // concrete infra in application
}
```
Good:
```ts
export const ORDER_REPOSITORY = Symbol('ORDER_REPOSITORY');
export interface OrderRepository { save(order: Order): Promise<void>; }

@Injectable()
export class CreateOrderUseCase {
  constructor(@Inject(ORDER_REPOSITORY) private readonly orders: OrderRepository) {}
}
```

## Review checklist

- [ ] No `new ConcreteInfra()` inside `domain/` or `application/`.
- [ ] No `typeorm` / `prisma` imports outside `infrastructure/`.
- [ ] Ports have <= 4 methods; split if larger.
- [ ] `if/switch` on type in UseCase replaced by polymorphism + DI.
- [ ] Swapping `useClass` does not break callers (LSP).
