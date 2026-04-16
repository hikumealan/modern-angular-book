# Modern Angular -- Learning Companion

Original educational content structured around the table of contents of
*Modern Angular: Architecture, Concepts, Implementation* by **Manfred Steyer** (March 2026).

> **Disclaimer**: This is independently written learning material -- it is **not** a reproduction
> of the book. The chapter structure follows the published table of contents, but the prose,
> code examples, and companion codebase are original work. For the authoritative treatment of
> these topics, purchase the book directly from the author at
> [angulararchitects.io](https://www.angulararchitects.io/).

---

## Example Application -- FinancialApp

Every chapter uses the same running example: a personal finance management application built
with Angular v21. The companion codebase lives in [`financial-app/`](financial-app/).

### Domain Entities

```typescript
interface Account {
  id: number;
  name: string;
  type: 'checking' | 'savings' | 'investment';
  balance: number;
  currency: string;
  ownerId: number;
}

interface Transaction {
  id: number;
  accountId: number;
  amount: number;
  type: 'credit' | 'debit';
  category: string;
  date: string;
  description: string;
  pending: boolean;
}

interface Client {
  id: number;
  firstName: string;
  lastName: string;
  email: string;
  riskProfile: 'conservative' | 'moderate' | 'aggressive';
}

interface Portfolio {
  id: number;
  clientId: number;
  name: string;
  holdings: number[]; // Holding IDs
  totalValue: number;
}

interface Holding {
  id: number;
  portfolioId: number;
  ticker: string;
  shares: number;
  purchasePrice: number;
  currentPrice: number;
}
```

### What the Domain Covers

| Domain area | Angular concepts demonstrated |
|---|---|
| Transaction entry & search | Signal forms, data binding, pipes, `httpResource`, `@for`/`@if` |
| Account management | Services, signal stores, routing, parameterized routes |
| Portfolio & holdings | Child routes, entity management, `@defer`, NgRx SignalStore |
| Client onboarding | Nested forms, validation, content projection |
| Advisor chat | Agentic UI (Hashbrown), tool calling, generative UI |

---

## Chapter Index

### Part I -- Fundamentals

| # | Chapter | File |
|---|---------|------|
| 1 | Getting Started with Angular | [ch01-getting-started.md](ch01-getting-started.md) |
| 2 | Signal-Based Components | [ch02-signal-components.md](ch02-signal-components.md) |
| 3 | Reactive Design with Signals | [ch03-reactive-signals.md](ch03-reactive-signals.md) |
| 4 | Navigation & Lazy Loading with the Router | [ch04-router.md](ch04-router.md) |
| 5 | State Management with Services & Signals | [ch05-state-management.md](ch05-state-management.md) |
| 6 | Signal Forms | [ch06-signal-forms.md](ch06-signal-forms.md) |
| 7 | Modern Testing with Vitest | [ch07-testing-vitest.md](ch07-testing-vitest.md) |

### Part II -- Architecture & Advanced

| # | Chapter | File |
|---|---------|------|
| 8 | Sustainable Architectures for Modern Angular | [ch08-architecture.md](ch08-architecture.md) |
| 9 | State Management with NgRx Signal Store | [ch09-ngrx-signal-store.md](ch09-ngrx-signal-store.md) |
| 10 | Signal Queries & Component Communication | [ch10-signal-queries.md](ch10-signal-queries.md) |
| 11 | Directives, Templates, and Containers | [ch11-directives-templates.md](ch11-directives-templates.md) |
| 12 | Initialization & Route Changes | [ch12-initialization-routes.md](ch12-initialization-routes.md) |
| 13 | Agentic UI & AI Assistants with Hashbrown | [ch13-agentic-ui-hashbrown.md](ch13-agentic-ui-hashbrown.md) |

### Part III -- Scaling & Deployment

| # | Chapter | File |
|---|---------|------|
| 14 | Monorepos & Reusable Libraries | [ch14-monorepos-libraries.md](ch14-monorepos-libraries.md) |
| 15 | Internationalization | [ch15-internationalization.md](ch15-internationalization.md) |
| 16 | Modern Patterns for Authentication & Authorization | [ch16-auth-patterns.md](ch16-auth-patterns.md) |
| 17 | Defer, SSR & Hydration | [ch17-defer-ssr-hydration.md](ch17-defer-ssr-hydration.md) |
| 18 | Micro Frontends: Scaling Across Multiple Teams | [ch18-micro-frontends.md](ch18-micro-frontends.md) |
| 19 | Analyzing Your Architecture with Forensic Techniques | [ch19-forensic-architecture.md](ch19-forensic-architecture.md) |

---

## Companion Codebase

The [`financial-app/`](financial-app/) directory contains a complete, runnable Angular v21
Nx monorepo. See its own [README](financial-app/README.md) for setup instructions.

### Key conventions in the codebase

- **Angular v21** -- zoneless by default, signal-first, standalone-only, Vitest
- **Plain SCSS** -- minimal styling, no CSS framework
- **Nx monorepo** -- `apps/financial-app` + `libs/shared/{models,ui,data-access}`
- **Mock API** -- json-server on port 3000 (`npm run mock-api`)
- **Chapter traceability** -- every source file references its chapter(s) in a comment

### Angular v21 Conventions Used Throughout

| Pattern | Modern (used here) | Legacy (avoided) |
|---|---|---|
| Change detection | Zoneless (signals) | zone.js, OnPush |
| Component I/O | `input()`, `output()`, `model()` | `@Input`, `@Output` |
| Queries | `viewChild()`, `contentChild()` | `@ViewChild`, `@ContentChild` |
| Control flow | `@if`, `@for`, `@switch` | `*ngIf`, `*ngFor`, `*ngSwitch` |
| DI | `inject()` function | Constructor injection |
| Guards/resolvers | `CanActivateFn`, `ResolveFn` | Class-based guards |
| Interceptors | `HttpInterceptorFn` | Class-based interceptors |
| Testing | Vitest (`vi.fn()`, `vi.spyOn()`) | Karma/Jasmine |
| Forms | Signal Forms (`formField`) | Reactive Forms |
| Async data | `httpResource()`, `resource()` | Manual `subscribe()` |
| Animations | `provideAnimationsAsync()` | `provideAnimations()` |
