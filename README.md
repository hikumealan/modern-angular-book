# Modern Angular -- Learning Companion

Original educational content structured around the table of contents from
*Modern Angular: Architecture, Concepts, Implementation* by **Manfred Steyer** (March 2026) and
*ng-book 2* by **Fullstack.io** (https://www.newline.co/ng-book/2/).

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

The book is organized into 12 parts plus appendices. All files are prefixed `chNN-` so they sort in reading order.

### Part 0 -- Orientation

| # | Chapter | File |
|---|---------|------|
| 0 | The Angular Ecosystem | [ch00-angular-ecosystem.md](ch00-angular-ecosystem.md) |

### Part I -- Core Angular

| # | Chapter | File |
|---|---------|------|
| 1 | Getting Started with Angular | [ch01-getting-started.md](ch01-getting-started.md) |
| 2 | Signal-Based Components | [ch02-signal-components.md](ch02-signal-components.md) |
| 3 | Reactive Design with Signals | [ch03-reactive-signals.md](ch03-reactive-signals.md) |
| 4 | Router & Lazy Loading | [ch04-router.md](ch04-router.md) |
| 5 | Services, DI & Shared State with Signals | [ch05-state-management.md](ch05-state-management.md) |
| 6 | Signal Forms | [ch06-signal-forms.md](ch06-signal-forms.md) |
| 7 | Testing with Vitest & Component Harnesses | [ch07-testing-vitest.md](ch07-testing-vitest.md) |

### Part II -- Reactivity at Depth

| # | Chapter | File |
|---|---------|------|
| 8 | RxJS Deep Dive | [ch08-rxjs-deep-dive.md](ch08-rxjs-deep-dive.md) |

### Part III -- Architecture, State & Structure

| # | Chapter | File |
|---|---------|------|
| 9 | Sustainable Architectures | [ch09-architecture.md](ch09-architecture.md) |
| 10 | NgRx Signal Store | [ch10-ngrx-signal-store.md](ch10-ngrx-signal-store.md) |
| 11 | Signal Queries & Component Communication | [ch11-signal-queries.md](ch11-signal-queries.md) |
| 12 | Directives, Templates & Containers | [ch12-directives-templates.md](ch12-directives-templates.md) |
| 13 | Initialization & Route Lifecycle | [ch13-initialization-routes.md](ch13-initialization-routes.md) |

### Part IV -- Monorepo & Repo Governance

| # | Chapter | File |
|---|---------|------|
| 14 | Monorepos & Nx Fundamentals | [ch14-monorepos-libraries.md](ch14-monorepos-libraries.md) |
| 15 | Advanced Nx: Generators, Release & CI | [ch15-advanced-nx.md](ch15-advanced-nx.md) |
| 16 | Style Guide & Architecture Boundaries | [ch16-style-guide-governance.md](ch16-style-guide-governance.md) |

### Part V -- Cross-Cutting Concerns

| # | Chapter | File |
|---|---------|------|
| 17 | Authentication & Authorization | [ch17-auth-patterns.md](ch17-auth-patterns.md) |
| 18 | Security & OWASP | [ch18-security-owasp.md](ch18-security-owasp.md) |
| 19 | Error Handling | [ch19-error-handling.md](ch19-error-handling.md) |
| 20 | Observability & Production Monitoring | [ch20-observability.md](ch20-observability.md) |
| 21 | Feature Flags & Runtime Config | [ch21-feature-flags.md](ch21-feature-flags.md) |

### Part VI -- UI, Design System & Accessibility

| # | Chapter | File |
|---|---------|------|
| 22 | Angular Material & Design System | [ch22-material-design-system.md](ch22-material-design-system.md) |
| 23 | Motion & Animations | [ch23-animations.md](ch23-animations.md) |
| 24 | Accessibility Full-Stack | [ch24-accessibility.md](ch24-accessibility.md) |
| 25 | Internationalization at Scale | [ch25-internationalization.md](ch25-internationalization.md) |
| 26 | Storybook for Component Libraries | [ch26-storybook.md](ch26-storybook.md) |

### Part VII -- Data, Contracts & Real-Time

| # | Chapter | File |
|---|---------|------|
| 27 | Advanced TypeScript & API Contracts | [ch27-advanced-typescript-openapi.md](ch27-advanced-typescript-openapi.md) |
| 28 | WebSockets & Real-Time Data | [ch28-websockets-realtime.md](ch28-websockets-realtime.md) |
| 29 | File Handling | [ch29-file-handling.md](ch29-file-handling.md) |
| 30 | Charts & Data Visualization | [ch30-charts.md](ch30-charts.md) |

### Part VIII -- Delivery & Runtime

| # | Chapter | File |
|---|---------|------|
| 31 | Defer, SSR & Hydration | [ch31-defer-ssr-hydration.md](ch31-defer-ssr-hydration.md) |
| 32 | Performance Optimization | [ch32-performance.md](ch32-performance.md) |
| 33 | PWA & Service Workers | [ch33-pwa-service-workers.md](ch33-pwa-service-workers.md) |
| 34 | Optimistic UI & Offline Sync | [ch34-optimistic-ui.md](ch34-optimistic-ui.md) |
| 35 | Web Workers | [ch35-web-workers.md](ch35-web-workers.md) |
| 36 | CI/CD & Deployment | [ch36-cicd-deployment.md](ch36-cicd-deployment.md) |
| 37 | E2E Testing with Playwright | [ch37-e2e-playwright.md](ch37-e2e-playwright.md) |

### Part IX -- Scaling, Platform & Enterprise

| # | Chapter | File |
|---|---------|------|
| 38 | Micro Frontends | [ch38-micro-frontends.md](ch38-micro-frontends.md) |
| 39 | Forensic Architecture | [ch39-forensic-architecture.md](ch39-forensic-architecture.md) |
| 40 | Angular Elements | [ch40-angular-elements.md](ch40-angular-elements.md) |
| 41 | Capacitor & Mobile | [ch41-capacitor-mobile.md](ch41-capacitor-mobile.md) |
| 42 | Compliance for Financial Apps | [ch42-compliance.md](ch42-compliance.md) |
| 43 | Enterprise Governance & Supply Chain | [ch43-enterprise-supply-chain.md](ch43-enterprise-supply-chain.md) |

### Part X -- Interaction, Forms & Polish

| # | Chapter | File |
|---|---------|------|
| 44 | Drag & Drop | [ch44-drag-and-drop.md](ch44-drag-and-drop.md) |
| 45 | Multi-Step Forms & Wizards | [ch45-multi-step-forms.md](ch45-multi-step-forms.md) |
| 46 | Command Palette & Keyboard Shortcuts | [ch46-command-palette.md](ch46-command-palette.md) |

### Part XI -- AI in Angular

| # | Chapter | File |
|---|---------|------|
| 47 | AI in Angular -- Runtime & Engineering | [ch47-ai-in-angular.md](ch47-ai-in-angular.md) |

### Part XII -- Advanced Mechanics & Upgrades

| # | Chapter | File |
|---|---------|------|
| 48 | Dependency Injection Deep Dive | [ch48-di-deep-dive.md](ch48-di-deep-dive.md) |
| 49 | Debugging Angular Applications | [ch49-debugging.md](ch49-debugging.md) |
| 50 | Migrations & Upgrades | [ch50-migrations.md](ch50-migrations.md) |

### Appendices

| # | Appendix | File |
|---|----------|------|
| 51 | Appendix A -- TypeScript Essentials for Angular | [ch51-ch51-ch51-appendix-a-typescript-primer.md](ch51-ch51-ch51-appendix-a-typescript-primer.md) |
| 52 | Appendix B0 -- Backend Adapter: FastAPI + Pydantic (canonical contract-driven workflow) | [ch52-appendix-b0-backend-fastapi.md](ch52-appendix-b0-backend-fastapi.md) |
| 53 | Appendix B1 -- Backend Adapter: Node + Express | [ch53-ch53-ch53-appendix-b1-backend-express.md](ch53-ch53-ch53-appendix-b1-backend-express.md) |
| 54 | Appendix B2 -- Backend Adapter: Spring Boot | [ch54-ch54-ch54-appendix-b2-backend-spring.md](ch54-ch54-ch54-appendix-b2-backend-spring.md) |
| 55 | Appendix B3 -- Backend Adapter: Go | [ch55-ch55-ch55-appendix-b3-backend-go.md](ch55-ch55-ch55-appendix-b3-backend-go.md) |
| 56 | Appendix C -- Reactive Forms Compatibility | [ch56-appendix-c-reactive-forms.md](ch56-appendix-c-reactive-forms.md) |

---

## Chapter Renumbering -- Redirect Table

If you arrived here from an older bookmark, use this table to find the new location. For the full mapping see [chapter-mapping.md](chapter-mapping.md).

| Old location | New location |
|---|---|
| ch08 Architecture | ch09 |
| ch09 NgRx Signal Store | ch10 |
| ch10 Signal Queries | ch11 |
| ch11 Directives & Templates | ch12 |
| ch12 Initialization & Routes | ch13 |
| ch13 Agentic UI (Hashbrown) | ch47 (merged into AI in Angular) |
| ch15 Internationalization | ch25 |
| ch16 Auth Patterns | ch17 |
| ch17 Defer, SSR & Hydration | ch31 |
| ch18 Micro Frontends | ch38 |
| ch19 Forensic Architecture | ch39 |
| ch20 Style Guide & Structure | ch16 (expanded with Governance) |
| ch21 Security & OWASP | ch18 |
| ch22 Accessibility & ARIA | ch24 (merged with screen-reader testing) |
| ch23 Error Handling | ch19 |
| ch24 Performance | ch32 |
| ch25 E2E Playwright | ch37 |
| ch26 PWA & Service Workers | ch33 |
| ch27 Material & Design System | ch22 (animation section moved to ch23) |
| ch28 Storybook | ch26 |
| ch29 AI Tooling & MCP | ch47 (merged into AI in Angular) |
| ch30 Advanced Nx | ch15 |
| ch31 Advanced TypeScript & OpenAPI | ch27 |
| ch32 RxJS Deep Dive | ch08 |
| ch33 WebSockets | ch28 |
| ch34 File Handling | ch29 |
| ch35 Charts | ch30 |
| ch36 Drag & Drop | ch44 |
| ch37 Animations Deep Dive | ch23 (merged with Motion from design system) |
| ch38 Web Workers | ch35 |
| ch39 Angular Elements | ch40 |
| ch40 Capacitor & Mobile | ch41 |
| ch41 CI/CD & Deployment | ch36 |
| ch42 Observability | ch20 |
| ch43 Feature Flags | ch21 |
| ch44 Optimistic UI | ch34 |
| ch46 Compliance | ch42 |
| ch47 Dependency Injection Deep Dive | ch48 |
| ch48 Debugging | ch49 |
| ch49 Screen Reader Testing | ch24 (merged with Accessibility) |
| ch50 Command Palette | ch46 |
| ch51 Migrations | ch50 |
| Appendix A TypeScript Essentials | ch51 / Appendix A (file renamed) |
| Appendix B0 Contract-Driven Workflow | ch52 / Appendix B0 (file + label renamed) |
| Appendix B1 Node + Express | ch53 / Appendix B1 (file renamed) |
| Appendix B2 Spring Boot | ch54 / Appendix B2 (file renamed) |
| Appendix B3 Go | ch55 / Appendix B3 (file renamed) |

Chapters whose number did NOT change: 0, 1, 2, 3, 4, 5, 6, 7, 14, 45.

---

## What This Book Does Not Cover

A few topics are intentionally out of scope. Pointers are provided so you know where to go when you need them:

- **NativeScript as a mobile runtime.** [Chapter 44](ch41-capacitor-mobile.md) covers Capacitor for iOS/Android deployment of Angular apps; Ionic and NativeScript are mentioned briefly. If you need NativeScript-specific guidance, see the [NativeScript documentation](https://docs.nativescript.org/).
- **Deep-dive coverage of classic Reactive Forms.** Signal Forms is the v21 default and receives full treatment in [Chapter 6](ch06-signal-forms.md). The `FormControl`/`FormGroup` compatibility material lives in [Appendix C](ch56-appendix-c-reactive-forms.md). For the complete classic API, see the [Angular Forms guide](https://angular.dev/guide/forms).
- **Plain Redux as a library.** The Redux pattern is covered through [NgRx Signal Store](ch10-ngrx-signal-store.md), which is the idiomatic choice in the signal era.
- **Backend service implementation details.** [Appendix B0](ch52-appendix-b0-backend-fastapi.md) and the sibling adapter appendices ([B1 Node + Express](ch53-ch53-ch53-appendix-b1-backend-express.md), [B2 Spring Boot](ch54-ch54-ch54-appendix-b2-backend-spring.md), [B3 Go](ch55-ch55-ch55-appendix-b3-backend-go.md)) cover the contract layer between the backend and Angular. [Chapter 34](ch36-cicd-deployment.md) covers deployment of the Angular app itself.

---

## Companion Codebase

The [`financial-app/`](financial-app/) directory contains a complete, runnable Angular v21
Nx monorepo. See its own [README](financial-app/README.md) for setup instructions.

### Key conventions in the codebase

- **Angular v21** -- zoneless by default, signal-first, standalone-only, Vitest
- **Plain SCSS** -- minimal styling, no CSS framework
- **Nx monorepo** -- `apps/financial-app` + `libs/shared/{models,ui,data-access}` + domain libs
- **Module boundaries** -- enforced by `@softarc/sheriff-core` (see [Chapter 31](ch16-style-guide-governance.md))
- **Mock API** -- json-server on port 3000 (`npm run mock-api`)
- **Canonical backend** -- FastAPI + Pydantic (see [Appendix B0](ch52-appendix-b0-backend-fastapi.md))
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
