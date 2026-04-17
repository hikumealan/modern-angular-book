# Chapter 51: Migration Guides

Most Angular work is not greenfield. You inherit an AngularJS 1.x application that has been paying for itself since 2015. You inherit an Angular v14 codebase with modules, class guards, and Karma tests that was "going to be migrated eventually." You inherit a React product marketing sold as "the new platform" and that now needs to interop with the Angular shell the rest of the company builds. Migrations are where architecture meets reality -- the place where the elegant patterns from the previous fifty chapters collide with payroll deadlines, browser support matrices, and code that predates the engineers maintaining it.

This chapter is a field guide covering the three migration categories you are most likely to encounter -- legacy AngularJS to modern Angular, older Angular to v21, and React/Vue to Angular -- and the cross-cutting concerns that apply to all of them: data-layer upgrades, test migration, long-running staged strategies, dependency bumps, CI tooling, and customer communication. Every pattern is grounded in a FinancialApp example. No new code lives in this chapter; each diff shows an older version of a file already in the repo next to its modernized form.

> **Companion code:** No new code. The chapter references before/after diffs of existing FinancialApp migrations (reactive forms to signal forms, NgModules to standalone, Karma configs to Vitest).

---

## Migrating AngularJS (1.x) to Angular v21

AngularJS reached end-of-life in January 2022, but the code it powers has not. Banks, insurers, and internal enterprise portals still run AngularJS 1.7 or 1.8 daily. A rewrite-from-scratch rarely survives its first budget review -- what does survive is a **hybrid bootstrap**: AngularJS and Angular running inside the same page, with components crossing the boundary in both directions while you migrate the application one leaf at a time.

### The Hybrid Bootstrap

The `@angular/upgrade/static` package lets both frameworks share a single DOM tree. Bootstrapping looks unlike anything else in this book:

```typescript
// legacy/src/main.ts
import { platformBrowser } from '@angular/platform-browser';
import { UpgradeModule } from '@angular/upgrade/static';
import { AppModule } from './app/app.module';
import * as angular from 'angular';
import './legacy/app.module';

platformBrowser()
  .bootstrapModule(AppModule)
  .then((ref) => {
    const upgrade = ref.injector.get(UpgradeModule);
    upgrade.bootstrap(document.body, ['financialAppLegacy'], { strictDi: true });
  });
```

Angular bootstraps first. The resulting `UpgradeModule` manually bootstraps the AngularJS module. Both frameworks share the DOM, share change detection via Zone.js (non-negotiable during the hybrid phase -- this is where you pay for the privilege of migrating incrementally), and share a single digest cycle.

### `downgradeComponent` and `upgradeComponent`

Two shims cross the boundary. `downgradeComponent` wraps an Angular component so AngularJS templates can consume it:

```typescript
// apps/legacy-shell/src/app/transaction-card.bridge.ts
import { downgradeComponent } from '@angular/upgrade/static';
import { TransactionCardComponent } from '@financial-app/shared/ui';

angular
  .module('financialAppLegacy')
  .directive(
    'finTransactionCard',
    downgradeComponent({ component: TransactionCardComponent }) as angular.IDirectiveFactory,
  );
```

Any `.html` file in the AngularJS app can now use `<fin-transaction-card transaction="$ctrl.tx"></fin-transaction-card>`. The reverse -- consuming an AngularJS directive inside an Angular template -- uses `upgradeComponent`:

```typescript
// apps/legacy-shell/src/app/bridges/legacy-advisor.directive.ts
@Directive({ selector: 'legacy-advisor-widget' })
export class LegacyAdvisorDirective extends UpgradeComponent {
  constructor(ref: ElementRef, injector: Injector) {
    super('legacyAdvisorWidget', ref, injector);
  }
}
```

The Angular template then uses `<legacy-advisor-widget [client]="client()"></legacy-advisor-widget>` while the underlying implementation remains AngularJS.

### Migration Order

There is exactly one order that works:

1. **Leaf components first.** A transaction row, a currency badge, an input wrapper. Downgrade each so AngularJS callers keep working. Every conversion is small, testable, and independently revertable.
2. **Containers next.** Once every leaf a container uses is available as an Angular component, migrate the container. Cross-tree communication collapses as you go.
3. **Services.** `$http` becomes `HttpClient`. `$q.defer()` becomes `firstValueFrom()`. Services converted early sit behind a thin AngularJS wrapper (`.factory('advisorService', (...) => angularService)`) until every caller is migrated.
4. **Routing last.** `ui-router` and Angular's `Router` cannot share the URL without pain. Flip routing only when every route destination is Angular. Until then, keep AngularJS as the routing authority.
5. **Remove AngularJS.** When the last `.html` template that references AngularJS expressions is gone, delete the `UpgradeModule` bootstrap, drop the `angular` dependency, and enjoy.

Timeline for a production app with 200-300k lines of AngularJS: **6-18 months**. The first three months feel like no progress; the last three feel like freefall.

---

## Migrating Older Angular (v14-v20) to v21

Most teams face the other migration: a working Angular codebase stuck three or four major versions behind. The framework has shipped good migration tooling for years, and the Nx workflow from [Chapter 30](ch30-advanced-nx.md) makes even multi-major jumps tractable.

### The Baseline: `nx migrate latest`

Regardless of which Angular version you start from, the workflow is identical:

```bash
nx migrate latest
# review package.json and migrations.json
npm install --legacy-peer-deps
nx migrate --run-migrations
# verify
nx run-many -t build test lint
git add -A && git commit -m "chore: migrate to Angular v21"
```

The `migrations.json` file is the single most important artifact. Read every entry before running migrations. Most are mechanical (renaming `HttpClientModule` to `provideHttpClient()`, for instance). A handful will touch behavior -- those deserve a commit of their own and a manual smoke test. Resist the temptation to run every migration in one batch when crossing more than two majors; prefer a migration commit per Angular major, because a bisect target that says "upgrade from v18 to v19" is infinitely more useful than "upgrade from v14 to v21".

### NgModule to Standalone

The single biggest structural change in modern Angular history. The Angular CLI ships a schematic that handles it:

```bash
ng generate @angular/core:standalone
```

The schematic runs in three phases -- convert components, remove modules, clean up imports. Commit after each phase.

A before/after for the FinancialApp's transaction card from [Chapter 2](ch02-signal-components.md):

```typescript
// BEFORE
@NgModule({
  declarations: [TransactionCardComponent, AccountCardComponent],
  imports: [CommonModule],
  exports: [TransactionCardComponent, AccountCardComponent],
})
export class SharedUiModule {}

@Component({
  selector: 'fin-transaction-card',
  templateUrl: './transaction-card.component.html',
})
export class TransactionCardComponent {
  @Input() transaction!: Transaction;
}

// AFTER
@Component({
  selector: 'fin-transaction-card',
  imports: [CurrencyPipe, DatePipe],
  templateUrl: './transaction-card.component.html',
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class TransactionCardComponent {
  readonly transaction = input.required<Transaction>();
}
```

The `SharedUiModule` file disappears entirely. The barrel export from [Chapter 14](ch14-monorepos-libraries.md) now exports components directly, not a module.

### Zone.js to Zoneless

`provideZonelessChangeDetection()` is the only bootstrap flag that changes, but the runtime implications reach into every component that called `ChangeDetectorRef.markForCheck()`, depended on `setTimeout` triggering a tick, or used the `async` pipe with `Observable`s that did not originate from `HttpClient`. The `onpush_zoneless_migration` MCP tool from [Chapter 29](ch29-ai-tooling-mcp-skills.md) analyzes the codebase and produces a migration plan:

```
> Please analyze my workspace for zoneless readiness.
Agent calls: onpush_zoneless_migration
Returns: 47 components still on default change detection,
         12 files use setTimeout for change-detection side effects,
         3 services subscribe to Observable streams outside Zone.
```

Address the `OnPush` conversions first -- zoneless tolerates `Default` strategy in v21 but at a performance cost. Then eliminate the `setTimeout` tricks: most resolve into `effect()` or `afterNextRender()` from [Chapter 3](ch03-reactive-signals.md). Finally, convert any manual subscriptions that relied on Zone tracking into `toSignal()` or explicit `ChangeDetectorRef.markForCheck()` calls.

### Karma to Vitest

Angular v21 ships Vitest as the default test runner. Existing Karma projects migrate through the `@nx/vite` plugin:

```bash
nx generate @nx/vite:convert-to-inferred
nx generate @nx/vite:configuration --project=financial-app --testEnvironment=jsdom
```

The mechanical conversion is a small config change:

```typescript
// BEFORE: karma.conf.js
module.exports = (config) => config.set({
  frameworks: ['jasmine', '@angular-devkit/build-angular'],
  plugins: [require('karma-jasmine'), require('karma-chrome-launcher')],
  browsers: ['ChromeHeadless'],
});

// AFTER: vitest.config.ts
import { defineConfig } from 'vitest/config';
import angular from '@analogjs/vite-plugin-angular';

export default defineConfig({
  plugins: [angular()],
  test: { environment: 'jsdom', globals: true, setupFiles: ['./src/test-setup.ts'] },
});
```

The test APIs are nearly identical -- `describe`, `it`, `expect` all come from Vitest. The subtle differences matter: `jasmine.createSpy()` becomes `vi.fn()`, `spyOn(...).and.returnValue(x)` becomes `vi.spyOn(...).mockReturnValue(x)`, and `done` callbacks give way to async functions. See [Chapter 7](ch07-testing-vitest.md) for the canonical FinancialApp test patterns.

### Reactive Forms to Signal Forms

Signal Forms ([Chapter 6](ch06-signal-forms.md)) shipped as developer preview in v21. Migrating is optional -- reactive forms still work, and the two APIs coexist in the same application. Migrate when:

- The form already has reactivity around validation or dependent-field logic.
- The team is comfortable with the rest of the signal-first stack.
- The form has tests you can port.

A before/after for the FinancialApp's transaction entry form:

```typescript
// BEFORE: domains/transactions/transaction-entry.component.ts
@Component({ /* ... */ imports: [ReactiveFormsModule] })
export class TransactionEntryComponent {
  private fb = inject(FormBuilder);
  form = this.fb.group({
    description: ['', Validators.required],
    amount: [0, [Validators.required, Validators.min(0.01)]],
    date: ['', Validators.required],
  });
}

// AFTER: domains/transactions/transaction-entry.component.ts
@Component({ /* ... */ imports: [SignalFormDirective] })
export class TransactionEntryComponent {
  readonly form = withSchema(transactionDraftSchema, {
    description: formField(''),
    amount: formField(0),
    date: formField(new Date().toISOString().slice(0, 10)),
  });
}
```

Coexistence is fine. Migrate one form at a time and keep `ReactiveFormsModule` in the providers list for as long as any form still uses it.

### Class-Based Guards to Functional `CanActivateFn`

Angular v14.2 deprecated class-based guards. In v21 they still compile but produce warnings. The conversion is one-to-one:

```typescript
// BEFORE
@Injectable({ providedIn: 'root' })
export class AuthGuard implements CanActivate {
  private auth = inject(AuthService);
  private router = inject(Router);
  canActivate(): boolean {
    if (this.auth.isAuthenticated()) return true;
    this.router.navigate(['/login']);
    return false;
  }
}

// AFTER
export const authGuard: CanActivateFn = () => {
  const auth = inject(AuthService);
  const router = inject(Router);
  if (auth.isAuthenticated()) return true;
  router.navigate(['/login']);
  return false;
};
```

Route configs change from `canActivate: [AuthGuard]` to `canActivate: [authGuard]`. The same transformation applies to `CanDeactivate`, `Resolve`, `CanLoad` (now `CanMatch`), and HTTP interceptors.

### `@Input` / `@Output` / `@ViewChild` to Signal APIs

The v16.1+ schematics handle the bulk of the conversion:

```bash
ng generate @angular/core:signal-input-migration
ng generate @angular/core:output-migration
ng generate @angular/core:signal-queries-migration
```

```typescript
// BEFORE
export class AccountCardComponent {
  @Input() account!: Account;
  @Input({ required: true }) balance!: number;
  @Output() selected = new EventEmitter<number>();
  @ViewChild('menu') menuRef?: ElementRef;
}

// AFTER
export class AccountCardComponent {
  readonly account = input.required<Account>();
  readonly balance = input.required<number>();
  readonly selected = output<number>();
  readonly menuRef = viewChild<ElementRef>('menu');
}
```

The schematic gets 90% of it right. The remaining 10% is always the same thing: places where `this.account` was written imperatively (`this.account = newValue`). Those are bugs the old decorator API silently tolerated -- the schematic flags them with `// TODO: migrate manually` and you rewrite the callsite to use a signal.

### `*ngIf` / `*ngFor` to `@if` / `@for`

Automated, total, safe:

```bash
ng generate @angular/core:control-flow-migration
```

The schematic rewrites templates from the structural-directive form to the control-flow syntax. The only manual work is deciding on a `track` expression for each `@for`, which the schematic conservatively defaults to `$index`. Replace `track $index` with `track item.id` wherever a stable identifier exists -- without it, every list re-render throws away DOM nodes.

### `@HostBinding` / `@HostListener` to the Host Object

```typescript
// BEFORE
@Component({ selector: 'fin-button', template: '<ng-content />' })
export class ButtonComponent {
  @HostBinding('class.fin-button') classes = true;
  @HostListener('click', ['$event']) onClick(e: MouseEvent) { /* ... */ }
}

// AFTER
@Component({
  selector: 'fin-button',
  template: '<ng-content />',
  host: {
    class: 'fin-button',
    '(click)': 'onClick($event)',
  },
})
export class ButtonComponent { onClick(e: MouseEvent) { /* ... */ } }
```

A codemod ships with Angular's migration schematics; manual review catches computed host bindings, which must move to `host: { '[class.active]': 'isActive()' }`.

---

## Migrating from React or Vue

The hardest migration is not version-to-version but framework-to-framework. Your team has React expertise; the rest of the company runs Angular; leadership wants convergence. The frameworks rhyme but do not match.

### Conceptual Mapping

| React / Vue | Angular equivalent |
|---|---|
| `useState`, `useRef`, Vue `ref` | `signal()` from [Chapter 3](ch03-reactive-signals.md) |
| `useMemo`, Vue `computed` | `computed()` |
| `useEffect`, Vue `watch` | `effect()` |
| Redux / Zustand | NgRx Signal Store, [Chapter 9](ch09-ngrx-signal-store.md) |
| React Router | Angular Router |
| TanStack Query / React Query | `httpResource` / `resource` |
| React Context | Dependency injection + signal stores |
| Custom hooks | Services or injectable functions |
| JSX | Angular templates with `@if` / `@for` |
| CSS-in-JS | Scoped component SCSS |

The hardest conceptual distance is not syntax -- it is *DI*. React and Vue pass state through props and context; Angular's injector is a first-class dependency graph with scoping rules. Teams coming from React routinely over-inject (everything becomes a provider) or under-inject (they try to pass services through input bindings). Reserve a full sprint of architecture review for the first few feature migrations.

### Team Training Curve

Typical learning curves for a React developer moving to Angular: weeks 1-2 cover templates, `@if`/`@for`, `input()`/`output()`, and the component decorator (low friction); weeks 3-4 cover DI, services, and the router (medium friction); weeks 5-8 cover Signal Forms, RxJS (inevitable in HTTP interceptors), and change detection (high friction); from month 3 onward, the work is internalizing the signal model and writing architecture that uses `computed()` and `linkedSignal()` idiomatically.

Plan for it. The first feature a React developer writes in Angular will contain anti-patterns. The second will contain fewer. Ship them both -- code review is the training.

### Parallel Development

Two patterns work for running React and Angular side by side during a migration:

- **Route-level split.** A reverse proxy routes `/transactions/*` to the Angular app and `/portfolios/*` to the React app. No runtime sharing -- each app is independent. Simple, ships fast, costs some cross-app navigation elegance.
- **Micro-frontend bridge via Angular Elements.** Expose each Angular feature as a custom element and mount it inside the React shell (or vice versa), covered in [Chapter 39](ch39-angular-elements.md). This works when the migration is slow and the shell cannot move. The price is payload duplication -- expect both framework runtimes in the browser for the duration. Choose the route-level split whenever politics allow; Elements are the fallback when they do not.

---

## Data Layer Migrations

Legacy apps tend to share a data-layer pattern: hand-rolled services wrapping `HttpClient` with manually written interfaces, type-cast responses, and no runtime validation. [Chapter 31](ch31-advanced-typescript-openapi.md) covers the destination -- OpenAPI-generated clients with Zod-validated responses. The path there is staged.

```typescript
// BEFORE: libs/shared/data-access/src/account.service.ts
@Injectable({ providedIn: 'root' })
export class AccountService {
  private http = inject(HttpClient);
  getAll(): Observable<Account[]> {
    return this.http.get<Account[]>('/api/accounts');  // cast, not validation
  }
}

// AFTER
@Injectable({ providedIn: 'root' })
export class AccountService {
  private client = createClient<paths>({ baseUrl: '' });
  async getAll(): Promise<Account[]> {
    const { data, error } = await this.client.GET('/api/accounts');
    if (error) throw new Error(error.message);
    return accountArraySchema.parse(data);  // runtime validation
  }
}
```

Stage the migration: generate an OpenAPI spec from the existing backend even if incomplete, write Zod schemas for the most critical response shapes, wrap the existing service so the new method delegates to the old one but adds parse-based validation (falling back to the legacy path if parsing fails until the spec catches up), flip callers to the new method incrementally, and remove the legacy method when the call count reaches zero.

API versioning during migration matters. Append `/v2/` to new endpoints while `/v1/` continues to serve the legacy frontend. Retire `/v1/` when metrics show no callers.

---

## Test Migrations

Migrating test suites is a migration of its own. The patterns mirror the main app: automated conversion where possible, manual review for behavioral edge cases.

### Jasmine/Karma to Vitest

The API differences are narrow but real:

| Jasmine | Vitest |
|---|---|
| `jasmine.createSpy()` | `vi.fn()` |
| `spyOn(obj, 'method').and.returnValue(x)` | `vi.spyOn(obj, 'method').mockReturnValue(x)` |
| `spyOn(obj, 'method').and.callThrough()` | `vi.spyOn(obj, 'method')` (default) |
| `jasmine.objectContaining({ id: 1 })` | `expect.objectContaining({ id: 1 })` |
| `done` callbacks | `async` / `await` |
| `fdescribe` / `fit` | `describe.only` / `it.only` |

A codemod handles the mechanical rewrite. Behavioral differences -- Vitest's module isolation is stricter than Karma's, and `vi.useFakeTimers()` differs from `jasmine.clock()` in subtle ways -- surface as test failures you triage individually.

### Protractor to Playwright

Protractor reached end-of-life in 2023. Tests that depended on Angular's Zone tracking (`browser.waitForAngular()`) must be reimagined; Playwright has no equivalent because it does not know or care about Angular. See [Chapter 25](ch25-e2e-playwright.md) for FinancialApp's Playwright patterns. The migration is a rewrite, not a codemod -- `element.all(by.css(...))` becomes `page.locator(...)` and assertions move from Jasmine matchers to Playwright's web-first expectations.

### Snapshot Tests

Snapshot tests age poorly across framework migrations because the DOM output changes for reasons unrelated to the behavior under test (`*ngIf` becomes `@if`, `ng-reflect-*` attributes disappear in production mode). Audit the snapshots: keep the ones covering business-critical output like invoice HTML, export CSV, or exported PDFs; rewrite snapshots of component markup as Testing Library queries that assert on roles, text, and attributes rather than structure.

---

## Long-Running Migration Strategy

Migrations longer than a sprint need discipline. Three patterns keep the application shippable throughout.

### Flag-Gated Migrations

Feature flags ([Chapter 43](ch43-feature-flags.md)) wrap the new and old code paths:

```typescript
@Injectable({ providedIn: 'root' })
export class TransactionEntryHost {
  private flags = inject(FeatureFlagService);
  readonly useSignalForms = computed(() =>
    this.flags.isEnabled('signal-forms.transaction-entry'),
  );
}
```

```html
@if (host.useSignalForms()) {
  <fin-transaction-entry-v2 />
} @else {
  <fin-transaction-entry />
}
```

Cut the flag over for a small percentage of users. Monitor. Expand. Remove the flag and the old component when metrics hold for two weeks.

### Parallel Change (Expand-Contract)

A three-step refactor that never leaves the repo in a broken state:

1. **Expand.** Add the new method or module next to the old one. Both compile; callers continue to use the old one.
2. **Migrate callers.** One commit per caller, each small and reviewable.
3. **Contract.** Once the call count on the old method reaches zero, delete it.

This is the pattern behind every migration schematic in the Angular CLI. It is also the pattern you should use for every hand-rolled migration larger than a single file.

### The Strangler Fig

Named after the banyan tree that grows around an existing tree until the host dies. In software: new features are only implemented in the new pattern. Old code is rewritten when a business change touches it, not proactively. After a year, the old pattern exists only in the parts of the app that have not shipped in a year -- which, for most production apps, is a surprisingly small fraction.

The strangler fig is the pattern for migrations with no committed deadline. Its risk is that the last 10% of the legacy code is the hardest 10% -- you inherit the bugs nobody wanted to touch.

---

## Non-Framework Dependency Migrations

Angular is the loudest dependency but rarely the riskiest. Three others deserve planning.

### Sass

Sass 1.x deprecations land continually. `@import` is deprecated in favor of `@use`. Division with `/` is deprecated in favor of `math.div()`. Plain `@mixin` calls to unresolved mixins will be errors in a future minor. The `sass --migrate-on-save` watcher handles the mechanical conversions; manual review catches cases where the deprecated behavior was load-bearing (deep selector trees that depend on `@import`'s transitive visibility).

### TypeScript Major Bumps

Each TypeScript major release tightens type checking. Plan for a few hundred new type errors per major:

- `strictNullChecks` edge cases appear around array indexing and `Object.keys` callbacks.
- `noUncheckedIndexedAccess` (off by default, worth turning on) turns every array and record access into `T | undefined`.
- New compiler flags may interact with transformers already in your build.

Bump TypeScript after Angular, never before. Angular's public types set the minimum supported TypeScript version, and race conditions between the two are painful.

### Node Major Bumps

Node 20, 22, 24 -- each ships with a new V8 and new standard-library removals. Audit native modules (`canvas`, `sharp`, any `node-gyp`-compiled dependency needs new binaries), watch for standard-library additions that replace userland shims (`crypto.randomUUID()` and friends), and keep the `.nvmrc` file from [Chapter 1](ch01-getting-started.md) in sync with the Dockerfile `FROM` line -- your CI runner can install anything, but your production runtime is whatever Dockerfile you shipped.

---

## CI Tooling for Migrations

Migration discipline lives in CI. Three integrations turn migration from a heroic effort into a default behavior.

**Monitor deprecation warnings.** Angular's compiler emits deprecation warnings to stdout. Pipe them into an artifact:

```yaml
- name: Capture deprecations
  run: nx run-many -t build 2>&1 | tee build.log
- name: Count deprecations
  run: echo "deprecations=$(grep -c DEPRECATED build.log)" >> "$GITHUB_ENV"
```

Post the count to the PR. Merges that increase the count are reviewed with scrutiny; merges that decrease it are celebrated.

**Block new legacy patterns.** Custom ESLint rules catch regressions:

```json
{
  "rules": {
    "@angular-eslint/prefer-standalone": "error",
    "@angular-eslint/prefer-signals": "error",
    "@angular-eslint/prefer-control-flow": "error",
    "@angular-eslint/no-ng-module": "error"
  }
}
```

Once a pattern is migrated, the lint rule prevents backsliding. A migration PR flips the rule from `warn` to `error` in the same commit that converts the last file.

**Automated upgrade PRs.** The `nx migrate` workflow from [Chapter 30](ch30-advanced-nx.md) runs weekly, creating a dependencies PR that is either green (merge) or red (investigate). Dependabot handles the non-Angular packages on the same cadence. Review one dependency PR per week and the framework stays current without a dedicated quarter of upgrade work.

---

## Working with Customers During Migration

Migrations affect customers even when they are purely internal. Three communications avoid surprises: release notes that explain behavior changes rather than code changes ("The login page now remembers your email" is a useful note; "Migrated AuthComponent to signal inputs" is not); a published migration calendar committing to a cadence (for example, "one major feature area migrates per month") that enterprise customers will ask for; and per-tenant feature-flag opt-outs so banks and compliance-heavy customers can stay on the legacy path until they can schedule their own regression testing.

---

## When Not to Migrate

Not every app deserves a migration. Stop and ask:

- **Is the app going to survive 18 more months?** If a sunset is scheduled, a migration is waste. Ship on the legacy stack and rewrite the replacement from scratch.
- **Is the affected feature end-of-life?** A module that only a handful of internal users touch weekly can stay on the old pattern forever, as long as the linter exemption is documented and scoped.
- **Is the team's bandwidth the binding constraint?** A partial migration that stalls is worse than no migration -- you now have two patterns and no authority on which to use. If the team cannot commit to finishing, do not start.

Migration is an investment. Like every investment, it can be the wrong call.

---

## Summary

Every chapter before this one described the destination. This one described the journey. The tools are remarkably good -- `nx migrate`, the Angular CLI schematics, the `onpush_zoneless_migration` MCP tool, the standalone and signal schematics -- but tools do not resolve ambiguity. Judgement does: about order, about scope, about when "good enough" is better than "perfect."

The patterns that repeat across every migration in this chapter:

- **Leaf-first order.** Whether migrating AngularJS components, decorators to signal APIs, or reactive forms to signal forms, start at the leaves and work toward the roots. The trunk moves only when the branches are ready.
- **Coexistence is fine.** Two patterns in the same app for six months is not failure. It is how migration works. Commit to finishing; do not commit to finishing tomorrow.
- **Flag-gate behavior changes, not code changes.** Wrap user-visible differences behind feature flags from [Chapter 43](ch43-feature-flags.md). Keep code-shape migrations (decorator to signal API) flag-free -- the compiler catches issues that flags would hide.
- **CI enforces the direction of travel.** Lint rules block new legacy code, deprecation counts track progress, and automated dependency PRs keep the framework current without heroics.
- **Schematics do 90%.** The other 10% is where the interesting bugs live. Budget review time per migration commit equal to at least the generation time, and lean on the AI tooling from [Chapter 29](ch29-ai-tooling-mcp-skills.md) -- Cursor rules, the Angular CLI MCP, and agent skills know about `@angular/upgrade`, know which schematic replaces which decorator, and can read your `migrations.json` and explain what each entry does.

Migrations are not glamorous. They do not demo well in release notes. But they are the work that distinguishes a codebase that will still be shipping in 2030 from one that will be frozen, abandoned, or rewritten from scratch. Done right, they are the quiet proof that the architecture from the first fifty chapters was correct -- and that the team maintaining it has the discipline to evolve.
