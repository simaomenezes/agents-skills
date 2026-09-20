---
name: clean-architecture-react
description: Use when scaffolding React features, hooks, use-cases, gateways, Chakra UI views, or checking Clean Architecture boundaries in Vite+TS.
---

# Clean Architecture for React (Vite + TS + Chakra UI)

Reference: Uncle Bob's Clean Architecture (Entities -> Use Cases -> Interface Adapters -> Frameworks & Drivers). Dependency rule points inward. UI kit: Chakra UI (https://chakra-ui.com, `npm i @chakra-ui/react`, `Provider`, `defineTokens`/`defineRecipe`/`defineTextStyles`).

## 1. Layer map

| Circle | Name | Location | Contains |
|--------|------|----------|----------|
| Yellow | Enterprise Business Rules | `src/domain/` | Entities, Value Objects, domain errors. No React/fetch/Query/Zod |
| Red | Application Business Rules | `src/application/` | Use Cases, Input/Output Ports, Gateway ports, DTOs. No React/Query/axios |
| Green | Interface Adapters | `src/interface/` | Controller hooks (`useXController`), Presenters (`toXViewModel`), mappers |
| Blue | Frameworks & Drivers | `src/infrastructure/` + `src/ui/` | `fetch/axios` gateways, Zod schemas, TanStack Query wrappers, Router, Chakra UI components/pages/theme (`Provider`, tokens, recipes) |

Flow: `View -> Controller hook -> InputPort -> Interactor -> Gateway port -> Presenter -> View`.

## 2. Allowed imports

```
ui/components, pages -> interface/controllers, interface/presenters
interface/controllers -> application/ports/in
application/use-cases -> domain/*, application/ports/out
infrastructure/gateways -> application/ports/*, domain/*
domain/* -> stdlib only
```

Forbidden: `domain` importing `react|@tanstack|axios|zod`; `application` importing `react|@tanstack|axios|fetch`; `fetch(` outside `infrastructure/`.

## 3. Canonical example (TanStack Query + Hook Form + Zod at edge)

```ts
// src/domain/orders/order.ts — pure, no decorators
export class Order {
  static create(items: OrderItem[]): Order { /* invariant: non-empty */ }
  total(): Money { /* business math here, not in component */ }
}

// src/application/orders/create-order.use-case.ts — no React
export class CreateOrderUseCase implements CreateOrderInputPort {
  constructor(private readonly gateway: OrderGateway) {}
  async execute(input: CreateOrderInput): Promise<OrderViewModel> {
    const order = Order.create(input.items);
    const saved = await this.gateway.create({ items: order.items });
    return toOrderViewModel(saved);
  }
}

// src/interface/orders/use-create-order.controller.ts
export function useCreateOrderController(port: CreateOrderInputPort) {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: (form: OrderFormData) => port.execute(toInput(form)),
    onSuccess: () => { qc.invalidateQueries({ queryKey: ['orders'] }); },
  });
}

// src/infrastructure/orders/order.schema.ts — Zod at edge
export const orderSchema = z.object({ items: z.array(orderItemSchema).min(1) });
export type OrderFormData = z.infer<typeof orderSchema>;

// src/ui/orders/OrderForm.tsx — dumb Chakra view, props only
import { Stack, Field, Input, Button } from '@chakra-ui/react';
export function OrderForm({ onSubmit, isPending, error }: { onSubmit: (d: OrderFormData) => void; isPending: boolean; error: string | null }) {
  const { register, handleSubmit, formState } = useForm<OrderFormData>({ resolver: zodResolver(orderSchema) });
  return (
    <Stack as="form" gap="4" onSubmit={handleSubmit(onSubmit)}>
      <Field.Root invalid={!!formState.errors.items}>
        <Field.Label>Items</Field.Label>
        <Input {...register('items')} placeholder="SKU x qty" />
        <Field.ErrorText>{formState.errors.items?.message}</Field.ErrorText>
      </Field.Root>
      {error ? <Alert.Root status="error"><Alert.Title>{error}</Alert.Title></Alert.Root> : null}
      <Button type="submit" loading={isPending}>Create order</Button>
    </Stack>
  );
}

// src/ui/orders/OrderListState.tsx — Query states with Chakra primitives
// isPending -> <Spinner />; isError -> <Alert status="error" />; empty -> <EmptyState />; data -> map ViewModel, no math here.
```

Chakra rules: primitives render ViewModels only; variants via `defineRecipe`, brand colors/fonts via `defineTokens`, type via `defineTextStyles` in `src/ui/theme/`; `Provider` wired once at app root; handlers call `onSubmit`/controller `mutate` only.

Wiring: factory/provider creates `HttpOrderGateway` -> `CreateOrderUseCase` -> controller hook receives port via props/context. Swap gateway (REST -> mock) without touching use case or component.

## 4. Anti-patterns

- `fetch`/`axios` inside `.tsx` or `useEffect` doing list+filter+total.
- Pricing/discount/status `if`s in component or hook.
- Zod schema imported by `domain/` or use case validating raw HTTP shape.
- `useQuery` with business math inline; query keys/URLs scattered.
- God `OrderPage` doing everything.
- Chakra as logic holder: price/status `if`s in Chakra props/handlers, `fetch` in `onClick`, hardcoded brand hex/px instead of tokens/recipes.

## 5. Checklist

- [ ] `grep -r "from 'react'|@tanstack|axios|zod" src/domain` empty; `grep -r "from 'react'|@tanstack|axios|fetch(" src/application` empty.
- [ ] Each feature has InputPort + Gateway port + presenter.
- [ ] Controller hook has no business `if`; component has no `fetch`.
- [ ] Form schema at edge, mapped to domain input before use case.
- [ ] Use case testable with mocked gateway (no QueryClient).
- [ ] Chakra only in `ui/` + `ui/theme/`; `Provider` present; brand styling via tokens/recipes; views handle pending/error/empty via `Spinner`/`Alert`/`EmptyState` with ViewModel flags.
