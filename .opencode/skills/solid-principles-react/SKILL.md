---
name: solid-principles-react
description: Use when reviewing or writing React + Chakra UI components, hooks, gateways for SRP OCP LSP ISP DIP compliance.
---

# SOLID for React (Vite + TS + Chakra UI)

Prefer ports + injection + composition over concretions in components. Chakra UI (https://chakra-ui.com) primitives are render-only: they receive ViewModels, never compute business rules.

## S — Single Responsibility

One reason to change. Split god files.
Bad: `OrderPage.tsx` fetching orders, computing totals, formatting money, posting creates.
Good: `Order` (invariants) + `CreateOrderUseCase` (orchestration) + `useCreateOrderController` (glue/Query) + Chakra `OrderForm/OrderView` (render ViewModel) + `HttpOrderGateway` (HTTP).
Rule: Chakra component = render ViewModel with `Stack/Field/Input/Button/Alert/Spinner`; hook-controller = call port + Query state; use case = rules.

## O — Open / Closed

Extend by composition/new adapters, not by editing core per variant.
Bad: adding `if (variant === 'premium')` branches into `OrderView` + use case for each new type.
Good:
```tsx
// New behavior = new component/gateway/presenter, core untouched
< OrderView actions={<PremiumActions />} />
// New visual variant = new Chakra recipe variant, not a forked component
// cardRecipe variants: { primary: { bg: "teal.600" } } via defineRecipe
// New data source = new class implementing OrderGateway, injected via factory
const usecase = new CreateOrderUseCase(new GraphqlOrderGateway());
```

## L — Liskov Substitution

Substitutable gateways/components. A mock `InMemoryOrderGateway` or Storybook `OrderView` props must work without changing callers. Flag presenters/gateways that throw `NotImplemented` or narrow accepted shapes vs the port contract.

## I — Interface Segregation

Small props/ports. Prefer `OrderListView({ orders, onSelect })` over `DashboardProps` with 20 fields; `ListOrdersGateway { list() }` + `CreateOrderGateway { create() }` over `GodApi` with 30 methods. Flag components forced to receive/pass unused props.

## D — Dependency Inversion

High-level code depends on abstractions injected in.
Bad:
```tsx
import axios from 'axios';
export function OrderList() {
  const [orders, setOrders] = useState([]);
  useEffect(() => { axios.get('/api/orders').then(r => setOrders(r.data)); }, []);
}
```
Good:
```tsx
export function OrderList({ gateway }: { gateway: OrderGateway }) { /* ... */ }
// or controller hook receiving InputPort via context/factory:
export function useOrdersController(port: ListOrdersInputPort) {
  return useQuery({ queryKey: ['orders'], queryFn: () => port.execute() });
}
```
`fetch/axios/QueryClient` constructed in `infrastructure/` factories/providers, never `new`-ed inside use cases or components.

## Review checklist

- [ ] No `fetch/axios` outside `infrastructure/`; no business `if` in `.tsx`/controller hooks (including Chakra handlers/props).
- [ ] Ports/props ≤ ~4 members; split god types.
- [ ] Swapping gateway/presenter needs no caller change (LSP/OCP).
- [ ] Zod/Query/Router only at edge, mapped before domain.
- [ ] Chakra tokens/recipes for brand styling; `Field` a11y (`label`/`errorText`/`invalid`); `Button loading/disabled` bound to ViewModel flags.
