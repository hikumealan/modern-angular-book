<!-- REWRITER-SKIP-THIS-FILE: chapter-mapping.md must preserve both old and new filenames as literals. Do not run the link-rewriter on it. -->

# Chapter Mapping -- Old -> New

This document is the single source of truth for the book reorganization. It lists every
chapter and appendix rename, so programmatic rewriters (link updaters, prose-ordinal
updaters, source-comment rewriters) can mechanically consume it.

Structure:
- Part I lists each file rename (old path -> new path) and the associated title/number change.
- Part II lists prose ordinal changes (`Chapter N` -> `Chapter M`) derived from Part I.
- Part III lists appendix-label prose changes (`Appendix B` -> `Appendix B0`).
- Part IV lists merge and rebalance operations that dissolve source chapters.
- Part V is the reader-facing redirect table (published in the README).

**Important for tooling**: the "Old file" column contains strings that no longer exist on disk. Do not run automatic file-link rewriters on this document.

---

## Part I -- File renames

Total: 52 existing chapters (ch00-ch51 + 4 appendices) -> 51 new chapters (ch00-ch50) + 6 appendices (ch51-ch56).

### Chapters

| Old filename                          | New filename                        | Old # | New # | Part | Title (new) | Notes |
|---|---|---|---|---|---|---|
| ch00-angular-ecosystem&#46;md | ch00-angular-ecosystem&#46;md | 0 | 0 | 0 Orientation | The Angular Ecosystem | unchanged |
| ch01-getting-started&#46;md | ch01-getting-started&#46;md | 1 | 1 | I Core Angular | Getting Started | unchanged |
| ch02-signal-components&#46;md | ch02-signal-components&#46;md | 2 | 2 | I | Signal-Based Components | unchanged |
| ch03-reactive-signals&#46;md | ch03-reactive-signals&#46;md | 3 | 3 | I | Reactive Design with Signals | unchanged |
| ch04-router&#46;md | ch04-router&#46;md | 4 | 4 | I | Router & Lazy Loading | unchanged |
| ch05-state-management&#46;md | ch05-state-management&#46;md | 5 | 5 | I | Services, DI & Shared State with Signals | unchanged |
| ch06-signal-forms&#46;md | ch06-signal-forms&#46;md | 6 | 6 | I | Signal Forms | Reactive Forms compat extracted to ch56-appendix-c-reactive-forms&#46;md |
| ch07-testing-vitest&#46;md | ch07-testing-vitest&#46;md | 7 | 7 | I | Testing with Vitest & Component Harnesses | CDK harness material added |
| ch32-rxjs-deep-dive&#46;md | ch08-rxjs-deep-dive&#46;md | 32 | 8 | II Reactivity at Depth | RxJS Deep Dive | moved forward |
| ch08-architecture&#46;md | ch09-architecture&#46;md | 8 | 9 | III Architecture | Sustainable Architectures | +1 |
| ch09-ngrx-signal-store&#46;md | ch10-ngrx-signal-store&#46;md | 9 | 10 | III | NgRx Signal Store | +1; absorbs short server-cache section |
| ch10-signal-queries&#46;md | ch11-signal-queries&#46;md | 10 | 11 | III | Signal Queries & Component Communication | +1 |
| ch11-directives-templates&#46;md | ch12-directives-templates&#46;md | 11 | 12 | III | Directives, Templates & Containers | +1 |
| ch12-initialization-routes&#46;md | ch13-initialization-routes&#46;md | 12 | 13 | III | Initialization & Route Lifecycle | +1 |
| ch14-monorepos-libraries&#46;md | ch14-monorepos-libraries&#46;md | 14 | 14 | IV Monorepo & Governance | Monorepos & Nx Fundamentals | unchanged |
| ch30-advanced-nx&#46;md | ch15-advanced-nx&#46;md | 30 | 15 | IV | Advanced Nx: Generators, Release & CI | moved adjacent to ch14 |
| ch20-style-guide-structure&#46;md | ch16-style-guide-governance&#46;md | 20 | 16 | IV | Style Guide & Architecture Boundaries | EXPANDED with governance/Sheriff/eslint/AGENTS&#46;md; file slug changed |
| ch16-auth-patterns&#46;md | ch17-auth-patterns&#46;md | 16 | 17 | V Cross-Cutting | Authentication & Authorization | +1; EXPANDED with SSO/SAML/WS-Fed |
| ch21-security-owasp&#46;md | ch18-security-owasp&#46;md | 21 | 18 | V | Security & OWASP | -3 |
| ch23-error-handling&#46;md | ch19-error-handling&#46;md | 23 | 19 | V | Error Handling | -4 |
| ch42-observability&#46;md | ch20-observability&#46;md | 42 | 20 | V | Observability & Production Monitoring | moved adjacent to error handling |
| ch43-feature-flags&#46;md | ch21-feature-flags&#46;md | 43 | 21 | V | Feature Flags & Runtime Config | -22 |
| ch27-material-design-system&#46;md | ch22-material-design-system&#46;md | 27 | 22 | VI UI, Design System & A11y | Angular Material & Design System | -5; animation section moved to ch23 |
| ch37-animations-deep-dive&#46;md | ch23-animations&#46;md | 37 | 23 | VI | Motion & Animations | absorbs old ch27's animation section |
| ch22-accessibility-aria&#46;md | ch24-accessibility&#46;md | 22 | 24 | VI | Accessibility Full-Stack | MERGE target (merges with ch49 content); adds VPAT section |
| ch49-screen-reader-testing&#46;md | *(merged into ch24)* | 49 | 24 | VI | *(merged)* | content absorbed into ch24 Accessibility Full-Stack |
| ch15-internationalization&#46;md | ch25-internationalization&#46;md | 15 | 25 | VI | Internationalization at Scale | EXPANDED with runtime vs build-per-locale, CI pipeline, financial formatting |
| ch28-storybook&#46;md | ch26-storybook&#46;md | 28 | 26 | VI | Storybook for Component Libraries | -2 |
| ch31-advanced-typescript-openapi&#46;md | ch27-advanced-typescript-openapi&#46;md | 31 | 27 | VII Data & Contracts | Advanced TypeScript & API Contracts | -4 |
| ch33-websockets-realtime&#46;md | ch28-websockets-realtime&#46;md | 33 | 28 | VII | WebSockets & Real-Time Data | -5 |
| ch34-file-handling&#46;md | ch29-file-handling&#46;md | 34 | 29 | VII | File Handling | -5 |
| ch35-charts&#46;md | ch30-charts&#46;md | 35 | 30 | VII | Charts & Data Visualization | -5 |
| ch17-defer-ssr-hydration&#46;md | ch31-defer-ssr-hydration&#46;md | 17 | 31 | VIII Delivery & Runtime | Defer, SSR & Hydration | +14; delivery-bundle content deduped vs new ch32 |
| ch24-performance&#46;md | ch32-performance&#46;md | 24 | 32 | VIII | Performance Optimization | +8 |
| ch26-pwa-service-workers&#46;md | ch33-pwa-service-workers&#46;md | 26 | 33 | VIII | PWA & Service Workers | +7 |
| ch44-optimistic-ui&#46;md | ch34-optimistic-ui&#46;md | 44 | 34 | VIII | Optimistic UI & Offline Sync | -10 |
| ch38-web-workers&#46;md | ch35-web-workers&#46;md | 38 | 35 | VIII | Web Workers | -3 |
| ch41-cicd-deployment&#46;md | ch36-cicd-deployment&#46;md | 41 | 36 | VIII | CI/CD & Deployment | -5 |
| ch25-e2e-playwright&#46;md | ch37-e2e-playwright&#46;md | 25 | 37 | VIII | E2E Testing with Playwright | moved adjacent to CI/CD |
| ch18-micro-frontends&#46;md | ch38-micro-frontends&#46;md | 18 | 38 | IX Scaling & Enterprise | Micro Frontends | +20 |
| ch19-forensic-architecture&#46;md | ch39-forensic-architecture&#46;md | 19 | 39 | IX | Forensic Architecture | +20 |
| ch39-angular-elements&#46;md | ch40-angular-elements&#46;md | 39 | 40 | IX | Angular Elements | +1 |
| ch40-capacitor-mobile&#46;md | ch41-capacitor-mobile&#46;md | 40 | 41 | IX | Capacitor & Mobile | +1 |
| ch46-compliance&#46;md | ch42-compliance&#46;md | 46 | 42 | IX | Compliance for Financial Apps | -4 |
| *(new)* | ch43-enterprise-supply-chain&#46;md | -- | 43 | IX | Enterprise Governance & Supply Chain | NEW chapter (SBOM, dep review, secrets, VPAT) |
| ch36-drag-and-drop&#46;md | ch44-drag-and-drop&#46;md | 36 | 44 | X Interaction & Polish | Drag & Drop | +8 |
| ch45-multi-step-forms&#46;md | ch45-multi-step-forms&#46;md | 45 | 45 | X | Multi-Step Forms & Wizards | unchanged # |
| ch50-command-palette&#46;md | ch46-command-palette&#46;md | 50 | 46 | X | Command Palette & Keyboard Shortcuts | -4 |
| ch13-agentic-ui-hashbrown&#46;md | ch47-ai-in-angular&#46;md | 13 | 47 | XI AI | AI in Angular: Runtime & Engineering | MERGE target (merges with ch29 AI tooling) |
| ch29-ai-tooling-mcp-skills&#46;md | *(merged into ch47)* | 29 | 47 | XI | *(merged)* | content absorbed into ch47 |
| ch47-di-deep-dive&#46;md | ch48-di-deep-dive&#46;md | 47 | 48 | XII Advanced & Upgrades | Dependency Injection Deep Dive | +1 |
| ch48-debugging&#46;md | ch49-debugging&#46;md | 48 | 49 | XII | Debugging Angular Applications | +1 |
| ch51-migrations&#46;md | ch50-migrations&#46;md | 51 | 50 | XII | Migrations & Upgrades | -1; final chapter |

### Appendices

| Old filename | New filename | New # | Label (new) | Notes |
|---|---|---|---|---|
| appendix-a-typescript-primer&#46;md | ch51-appendix-a-typescript-primer&#46;md | 51 | Appendix A -- TypeScript Essentials | file-rename only |
| appendix-b-e2e-workflow&#46;md | ch52-appendix-b0-backend-fastapi&#46;md | 52 | Appendix B0 -- Backend Adapter: FastAPI + Pydantic | file + prose rename ("Appendix B" -> "Appendix B0"); content is the canonical contract-driven workflow recipe |
| appendix-b1-backend-express&#46;md | ch53-appendix-b1-backend-express&#46;md | 53 | Appendix B1 -- Backend Adapter: Node + Express | file-rename only |
| appendix-b2-backend-spring&#46;md | ch54-appendix-b2-backend-spring&#46;md | 54 | Appendix B2 -- Backend Adapter: Spring Boot | file-rename only |
| appendix-b3-backend-go&#46;md | ch55-appendix-b3-backend-go&#46;md | 55 | Appendix B3 -- Backend Adapter: Go | file-rename only |
| *(new)* | ch56-appendix-c-reactive-forms&#46;md | 56 | Appendix C -- Reactive Forms Compatibility | NEW appendix, extracted from old ch06 |

Note: the `&#46;` HTML entity is used for the dot in `.md` so this document's filenames survive programmatic rewriters that match `.md` suffixes.

---

## Part II -- Prose ordinal map (`Chapter N` -> `Chapter M`)

Derived mechanically from Part I. Generator scripts should use this as a dictionary. Mentions like "Chapter 3", "Ch 3", "in Chapter 3", "chapters 3-5" all map through the same table.

| Old # | New # |
|---|---|
| 0 | 0 |
| 1 | 1 |
| 2 | 2 |
| 3 | 3 |
| 4 | 4 |
| 5 | 5 |
| 6 | 6 |
| 7 | 7 |
| 8 | 9 |
| 9 | 10 |
| 10 | 11 |
| 11 | 12 |
| 12 | 13 |
| 13 | 47 |
| 14 | 14 |
| 15 | 25 |
| 16 | 17 |
| 17 | 31 |
| 18 | 38 |
| 19 | 39 |
| 20 | 16 |
| 21 | 18 |
| 22 | 24 |
| 23 | 19 |
| 24 | 32 |
| 25 | 37 |
| 26 | 33 |
| 27 | 22 |
| 28 | 26 |
| 29 | 47 |
| 30 | 15 |
| 31 | 27 |
| 32 | 8 |
| 33 | 28 |
| 34 | 29 |
| 35 | 30 |
| 36 | 44 |
| 37 | 23 |
| 38 | 35 |
| 39 | 40 |
| 40 | 41 |
| 41 | 36 |
| 42 | 20 |
| 43 | 21 |
| 44 | 34 |
| 45 | 45 |
| 46 | 42 |
| 47 | 48 |
| 48 | 49 |
| 49 | 24 |
| 50 | 46 |
| 51 | 50 |

Ambiguity note: chapters 13 + 29 both merge into new ch47, and 22 + 49 both merge into new ch24. Prose referring to either source chapter (e.g., "we saw in Chapter 13") rewrites to the merged target ("Chapter 47"); the rewriter should not worry about which of the two sources the sentence originally described.

---

## Part III -- Appendix-label prose map

Only one prose change; all other appendix labels stay as-is.

| Old prose | New prose |
|---|---|
| `Appendix B` | `Appendix B0` |

Do NOT rewrite `Appendix B1`, `Appendix B2`, or `Appendix B3` -- the rewriter's regex must require a word boundary after `B` to avoid corrupting those.

---

## Part IV -- Merge / rebalance operations

### Merge 1: Accessibility Full-Stack (new ch24)

- Source: old ch22 (Accessibility, Angular Aria & a11y Best Practices) + old ch49 (Screen Reader Testing & Advanced A11y)
- Target file: `ch24-accessibility.md`
- Strategy: ch22 body becomes the chapter opener (ARIA, Angular Aria, CDK a11y, axe). Append ch49 as a new "Screen reader verification" section. Add a new short "VPAT & audit governance" section. Drop ch49's duplicated WCAG 2.2 AA recap.
- Source files deleted: old ch22, old ch49

### Merge 2: AI in Angular (new ch47)

- Source: old ch13 (Agentic UI & AI Assistants with Hashbrown) + old ch29 (AI Tooling, MCP & Agent Skills)
- Target file: `ch47-ai-in-angular.md`
- Strategy: Split into two parts within the chapter. Part 1 "Runtime AI" is old ch13 (Hashbrown chat, tool calling, generative UI). Part 2 "Engineering AI" is old ch29 (Cursor rules, MCP, Agent Skills). Single "Stability notice" opens the chapter since both source chapters had one.
- Source files deleted: old ch13, old ch29

### Rebalance: Motion & Animations (new ch23)

- Source: old ch37 body + animation section from old ch27
- Target file: `ch23-animations.md`
- Strategy: ch37 body forms the chapter. Prepend ch27's animation section as the "From the design system" opening. Old ch27 (now ch22) loses its animation section and gains a forward pointer to ch23.
- Source file deleted: old ch37 (ch27 survives as ch22)

### Dedupe: Delivery bundles (ch31 Defer/SSR + ch32 Performance)

- No file merge. ch31 keeps the "splitting the bundle with @defer" narrative. ch32 (Performance) removes its duplicate bundle-splitting section and inserts a single pointer sentence back to ch31. Measurement, profiling, and budget material in ch32 stays.

### Absorb: Server cache section (new ch10 NgRx Signal Store)

- New ch10 (was ch09) gains a ~2-page "When to reach for a server cache (TanStack Query comparison)" section. No standalone "data layer" chapter; the comparison lives inside the store chapter where it makes sense.

---

## Part V -- Redirect table (reader-facing)

Paste this table into the root README under a "Chapter renumbering" heading so readers with old bookmarks can find the new location.

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
| Appendix B Contract-Driven Workflow | ch52 / Appendix B0 (file + label renamed) |
| Appendix B1 Node + Express | ch53 / Appendix B1 (file renamed) |
| Appendix B2 Spring Boot | ch54 / Appendix B2 (file renamed) |
| Appendix B3 Go | ch55 / Appendix B3 (file renamed) |

Chapters whose number did NOT change: 0, 1, 2, 3, 4, 5, 6, 7, 14, 45.
