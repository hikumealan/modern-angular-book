# Chapter 47: Dependency Injection Deep Dive

> **Extends:** [Chapter 5](ch05-state-management.md) and [Chapter 12](ch12-initialization-routes.md) used DI in passing. This chapter covers the machinery -- tokens, multi-providers, hierarchical injection, and `EnvironmentInjector`.

Angular's dependency injection system is hierarchical, tree-shakable, and resolved at runtime. For most of what you write, none of that matters. You decorate a class with `@Injectable({ providedIn: 'root' })`, call `inject(AccountService)` from a component, and the framework hands you the singleton. The fast paths are so ergonomic that a developer can ship features for months without touching the rest of the machinery.

This chapter covers the rest -- what happens when you need to inject a configuration value that isn't a class, aggregate contributions from multiple feature modules, scope a store to a route, create a component dynamically, or carve out an isolated injector for a micro frontend. Every example uses the **FinancialApp** we have been evolving since [Chapter 8](ch08-architecture.md).

> **Companion code:** Widget-registration example in `financial-app/libs/shared/data-access/`; dynamic component creation uses `EnvironmentInjector` in the shared UI dialog service.

---

## Why Tokens Exist

The injector is, at its simplest, a map. It takes a key, walks up a tree of parent injectors looking for a provider, and returns the associated instance. The interesting design decision is what counts as a key.

For class providers, the class itself is the key. `inject(AccountService)` asks the injector for whatever is registered under that constructor reference -- type-safe, unique, and tree-shakable for free.

But plenty of values are not classes. An API base URL is a string. A feature-flag object is a plain record. An error-handler callback is a function. For those, there is no class identity to key on, and a raw string key is fragile: nothing stops two libraries from both picking `'API_URL'` and colliding, and the compiler has no type to check against. Angular's answer is the `InjectionToken<T>`.

```typescript
// libs/shared/config/src/tokens.ts
import { InjectionToken } from '@angular/core';

export interface AppConfig {
  readonly apiUrl: string;
  readonly featureFlags: Record<string, boolean>;
  readonly environment: 'development' | 'staging' | 'production';
}

export const APP_CONFIG = new InjectionToken<AppConfig>('APP_CONFIG');
export const API_URL = new InjectionToken<string>('API_URL');
```

The string passed to the constructor is a debug label, not an identifier. The token's identity is the object itself -- two tokens created with the same label are still different keys. That uniqueness is what makes tokens safe to export from a library.

Providing a token looks identical to providing a class, except you spell out the `provide`/`useValue` pair:

```typescript
// apps/financial-app/src/app/app.config.ts
import { ApplicationConfig } from '@angular/core';
import { APP_CONFIG, API_URL } from '@financial-app/shared/config';
import { environment } from '../environments/environment';

export const appConfig: ApplicationConfig = {
  providers: [
    { provide: API_URL, useValue: environment.apiUrl },
    { provide: APP_CONFIG, useValue: environment.config },
  ],
};
```

Injecting is the mirror image:

```typescript
// libs/shared/data-access/src/account.service.ts
@Injectable({ providedIn: 'root' })
export class AccountService {
  private http = inject(HttpClient);
  private apiUrl = inject(API_URL);

  list() {
    return this.http.get<Account[]>(`${this.apiUrl}/accounts`);
  }
}
```

`inject(API_URL)` returns `string`, not `unknown`. The generic parameter on the token carries all the way through.

---

## The Four Provider Types

Every entry in a `providers` array ultimately reduces to one of four provider shapes. Knowing all four saves you from reaching for service locators when a factory would do.

**`useClass`** is the default for class providers. `AccountService` on its own is shorthand for `{ provide: AccountService, useClass: AccountService }`. The interesting case is swapping implementations:

```typescript
// apps/financial-app/src/app/app.config.ts
providers: [
  { provide: AnalyticsService, useClass: environment.production ? GoogleAnalyticsService : NoopAnalyticsService },
]
```

Every `inject(AnalyticsService)` call now returns the environment-appropriate implementation.

**`useValue`** registers a fixed value -- configuration, constants, anything you have in hand at bootstrap. Angular stores it as-is and never constructs or re-evaluates it:

```typescript
{ provide: APP_CONFIG, useValue: { apiUrl: 'https://api.example.com', featureFlags: {}, environment: 'production' } }
```

**`useFactory`** defers creation to a function that Angular calls the first time the token is requested. Factories run inside an injection context, so the modern form just uses `inject()`:

```typescript
// libs/shared/data-access/src/logger.provider.ts
export const LOGGER = new InjectionToken<Logger>('LOGGER');

export const loggerProvider = {
  provide: LOGGER,
  useFactory: (): Logger => {
    const config = inject(APP_CONFIG);
    return config.environment === 'production' ? new RemoteLogger(config.apiUrl) : new ConsoleLogger();
  },
};
```

The legacy form takes a `deps: [APP_CONFIG]` array and receives dependencies as positional arguments. Both are equivalent; the `inject()` version is harder to misconfigure because the dependencies do not have to be kept in sync with a separate list.

**`useExisting`** aliases one token to another's instance. It is what you use when a concrete service implements an interface-typed token:

```typescript
// libs/shared/auth/src/auth.provider.ts
export const AUTH_CLIENT = new InjectionToken<AuthClient>('AUTH_CLIENT');

providers: [
  OAuthAuthService,
  { provide: AUTH_CLIENT, useExisting: OAuthAuthService },
]
```

Both `inject(OAuthAuthService)` and `inject(AUTH_CLIENT)` now return the *same* instance. `useClass` would create a second instance under the aliased key, which is almost never what you want.

---

## Multi-Providers

The default provider behavior is last-wins: registering two providers under the same token means the later replaces the earlier. Sometimes you want contributions to *accumulate* instead -- many registrants, one consumer that receives all of them as an array. That is what `multi: true` gives you.

The classic example is HTTP interceptors. Each interceptor registered itself under `HTTP_INTERCEPTORS`, the framework collected them, and the HTTP client ran requests through the pipeline. Modern code uses `withInterceptors(...)` from `provideHttpClient`, which hides the multi-provider mechanism, but the underlying pattern is unchanged.

The FinancialApp uses the pattern for dashboard widget registration. The dashboard is owned by the shell, but individual widgets (account summary, portfolio performance, recent transactions) live in their feature libraries. Each feature registers its widget metadata under a shared token, and the dashboard renders whatever shows up.

```typescript
// libs/shared/data-access/src/widget-registry.ts
import { InjectionToken, Type } from '@angular/core';

export interface WidgetRegistration {
  readonly id: string;
  readonly title: string;
  readonly component: Type<unknown>;
  readonly defaultGridSpan: 1 | 2 | 3 | 4;
  readonly permission?: string;
}

export const WIDGET_REGISTRATIONS = new InjectionToken<WidgetRegistration[]>(
  'WIDGET_REGISTRATIONS',
);
```

Each feature contributes its widgets as a multi-provider:

```typescript
// libs/accounts/feature/src/accounts.providers.ts
export const accountsWidgetProviders: Provider[] = [
  {
    provide: WIDGET_REGISTRATIONS,
    useValue: { id: 'accounts-summary', title: 'Account Summary', component: AccountSummaryWidgetComponent, defaultGridSpan: 2 },
    multi: true,
  },
];

// libs/portfolio/feature/src/portfolio.providers.ts
export const portfolioWidgetProviders: Provider[] = [
  {
    provide: WIDGET_REGISTRATIONS,
    useValue: { id: 'portfolio-performance', title: 'Portfolio Performance', component: PerformanceWidgetComponent, defaultGridSpan: 3 },
    multi: true,
  },
  {
    provide: WIDGET_REGISTRATIONS,
    useValue: { id: 'portfolio-allocation', title: 'Asset Allocation', component: AllocationWidgetComponent, defaultGridSpan: 2, permission: 'portfolio:view' },
    multi: true,
  },
];
```

The dashboard component asks for the accumulated array and renders it:

```typescript
// libs/dashboard/feature/src/dashboard.component.ts
@Component({
  selector: 'fin-dashboard',
  template: `
    <section class="dashboard-grid">
      @for (widget of visibleWidgets(); track widget.id) {
        <fin-widget-host [registration]="widget" [span]="widget.defaultGridSpan" />
      }
    </section>
  `,
})
export class DashboardComponent {
  private widgets = inject(WIDGET_REGISTRATIONS);
  private permissions = inject(PermissionService);

  readonly visibleWidgets = computed(() =>
    this.widgets.filter((w) => !w.permission || this.permissions.has(w.permission)),
  );
}
```

The dashboard never imports the feature libraries' widget components directly; the injector is the glue. A new feature team ships a widget by exporting a provider array without editing the dashboard at all.

One caveat: multi-providers and non-multi-providers cannot mix on the same token. The first provider Angular sees fixes the mode, and registering the opposite form later throws at bootstrap. If a token is meant for aggregation, enforce it by convention -- export only a helper that returns multi-providers.

---

## Hierarchical Injection

Injectors form a tree, not a flat map. Every application has at least these layers:

1. The **platform injector** -- one per browser tab. You never interact with it directly.
2. The **root environment injector** -- built from `appConfig.providers` and everything `providedIn: 'root'` contributes.
3. **Route environment injectors** -- created for any route that has a `providers` array (covered in [Chapter 12](ch12-initialization-routes.md)) or that is lazy-loaded.
4. **Component injectors** -- one per component instance, created as the component tree is constructed.

When you call `inject(SomeToken)`, Angular starts at the current injector and walks *up* the tree until it finds a provider. The first match wins. Siblings are invisible: a component in one branch cannot see providers declared in another branch.

The practical consequence is that *scope follows placement*. A service provided at `providedIn: 'root'` is a singleton for the whole application. A service provided in a route's `providers` array is scoped to that route and its children -- navigating away and back creates a fresh instance. A service provided in a component's `providers` array is scoped to that component instance and its descendants.

The FinancialApp uses this for the transaction entry form. The form needs a draft store that must be fresh every time the user opens the form -- carrying state across navigations would re-populate the inputs with a stale draft. Rather than manually resetting state on `ngOnInit`, the feature scopes the store to the route:

```typescript
// libs/transactions/feature-entry/src/transaction-entry.routes.ts
export const transactionEntryRoutes: Routes = [
  {
    path: 'new',
    providers: [TransactionDraftStore],
    loadComponent: () => import('./transaction-entry.component').then((m) => m.TransactionEntryComponent),
  },
];
```

`TransactionDraftStore` has no `@Injectable({ providedIn })` decoration at all. The route is its only provider location, so every navigation to `/transactions/new` spawns a new store, and leaving the route destroys it. Components inside the form call `inject(TransactionDraftStore)` and get the route-scoped instance. This is how [Chapter 5](ch05-state-management.md)'s "per-feature store" pattern actually works.

Rule of thumb: default to `providedIn: 'root'`. Scope to a route when state must not outlive the route. Scope to a component only when state must not outlive a specific component instance.

---

## `EnvironmentInjector` and Injection Contexts

Two kinds of injectors live side by side: `ElementInjector` (one per component instance, tied to the DOM) and `EnvironmentInjector` (the modern replacement for the `NgModule` injector). `EnvironmentInjector` holds providers that are not tied to a particular component -- the root injector, route injectors, and lazy-loaded library injectors are all `EnvironmentInjector` instances.

You rarely create one directly, but two situations require it: dynamic component creation, and running `inject()` outside the call stack of a constructor or factory.

### Dynamic Component Creation

The shared `DialogService` from [Chapter 11](ch11-directives-templates.md) creates dialog components at runtime via `createComponent()`. That API needs an `EnvironmentInjector` so the created component can see the application's global providers:

```typescript
// libs/shared/ui/src/dialog/dialog.service.ts
@Injectable({ providedIn: 'root' })
export class DialogService {
  private appRef = inject(ApplicationRef);
  private environmentInjector = inject(EnvironmentInjector);

  open<T>(component: Type<T>, inputs?: Partial<T>): ComponentRef<T> {
    const host = document.createElement('div');
    document.body.appendChild(host);
    const ref = createComponent(component, { hostElement: host, environmentInjector: this.environmentInjector });
    if (inputs) {
      for (const [key, value] of Object.entries(inputs)) ref.setInput(key, value);
    }
    this.appRef.attachView(ref.hostView);
    return ref;
  }
}
```

When a dialog needs its own scoped providers -- say, a form-specific store it should not share with sibling dialogs -- `createEnvironmentInjector()` builds a child injector on the fly:

```typescript
// libs/shared/ui/src/dialog/dialog.service.ts
openWithProviders<T>(component: Type<T>, providers: Provider[]): ComponentRef<T> {
  const scoped = createEnvironmentInjector(providers, this.environmentInjector);
  const host = document.createElement('div');
  document.body.appendChild(host);
  const ref = createComponent(component, { hostElement: host, environmentInjector: scoped });
  this.appRef.attachView(ref.hostView);
  return ref;
}
```

The dialog and its children can now `inject(DraftStore)` and receive an instance scoped to that specific dialog.

### `runInInjectionContext`

`inject()` only works inside an *injection context*: the body of a class constructor, a field initializer, a factory, a route guard, or `runInInjectionContext`. Calling it from an arbitrary function -- a `setTimeout` callback, a method on a plain class -- throws `NG0203: inject() must be called from an injection context`.

`runInInjectionContext` is the escape hatch. Given an injector, it runs a callback with that injector temporarily installed as the current context:

```typescript
// libs/shared/data-access/src/deferred-setup.ts
export function scheduleBackgroundRefresh(): void {
  const injector = inject(EnvironmentInjector);
  setTimeout(() => {
    runInInjectionContext(injector, () => {
      inject(AccountService).refresh().subscribe();
    });
  }, 5000);
}
```

---

## When Injection Fails: The Null Injector and `@Optional()`

The root injector's parent is the **null injector**, which has no providers and throws `NG0201: No provider for <Token>` when asked for anything. Every failing injection eventually reaches it.

Sometimes that failure is correct behavior. Sometimes you want to inject something if it exists and fall back if it does not. The `optional: true` flag on `inject()` suppresses the error and returns `null`:

```typescript
// libs/shared/data-access/src/feature-flag.ts
@Injectable({ providedIn: 'root' })
export class FeatureFlagService {
  private overrides = inject(FEATURE_FLAG_OVERRIDES, { optional: true });

  isEnabled(flag: string): boolean {
    if (this.overrides && flag in this.overrides) return this.overrides[flag];
    return this.defaultFlags[flag] ?? false;
  }
}
```

The inferred type of `overrides` is `Record<string, boolean> | null`, so TypeScript forces the null check. Tests can override flags by providing the token; production skips the provider entirely.

Other useful `inject()` options:

- **`{ self: true }`** -- look only at the current element injector; do not walk up the tree.
- **`{ skipSelf: true }`** -- skip the current injector and start at the parent. The classic use is a recursive directive wanting the next ancestor up.
- **`{ host: true }`** -- stop at the nearest component host. Rare outside directive authoring.

These correspond to the older `@Optional()`, `@Self()`, `@SkipSelf()`, and `@Host()` decorators that constructor injection still uses.

---

## `inject()` in Reactive Contexts

Angular tracks "the current injection context" as internal state. It is set during class construction, factory calls, route guards and resolvers, and inside `runInInjectionContext`. It is *not* set inside methods called later, RxJS callback chains whose emissions are produced long after the stream was assembled, or anything past a `setTimeout` / `Promise.then` / `await`.

The failure mode is subtle because the rule isn't quite "inject only works in a class." This works:

```typescript
readonly accounts = this.fetchAccounts();

private fetchAccounts() {
  const http = inject(HttpClient);
  return http.get<Account[]>('/api/accounts');
}
```

The field initializer runs during construction, which is an injection context; calling a helper method from there preserves it. But move the `inject()` call into a later callback and it breaks:

```typescript
ngOnInit() {
  setTimeout(() => {
    const http = inject(HttpClient);
    http.get('...').subscribe();
  }, 1000);
}
```

By the time the `setTimeout` fires, the original injection context is gone.

The idiomatic pattern is to inject at the top of the class and reference the field later, not to reach for `inject()` deep inside callbacks. When you genuinely need injection inside an async flow -- typically when the token depends on runtime input -- capture the injector up front and use `runInInjectionContext`:

```typescript
@Component({ /* ... */ })
export class ConfigurableLoaderComponent {
  private injector = inject(Injector);

  loadLater() {
    setTimeout(() => {
      runInInjectionContext(this.injector, () => {
        inject(HttpClient).get('...').subscribe();
      });
    }, 1000);
  }
}
```

---

## Custom Injectors for Micro Frontends

[Chapter 18](ch18-micro-frontends.md) introduced the FinancialApp's micro-frontend topology: a shell application loading Accounts, Portfolio, and Onboarding MFEs at runtime via Native Federation. Each MFE bootstraps its own Angular application, which means each MFE has its own root `EnvironmentInjector`.

This creates an important property: **each MFE's DI tree is isolated**. A service provided at the root of the Accounts MFE is invisible to the Portfolio MFE, even though both run in the same browser tab. That isolation is usually what you want -- it is what makes independent deployment possible.

But some state is genuinely shared. Authentication is the archetype: all three MFEs must see the same logged-in user, the same token refresh cycle, the same sign-out event. Creating three independent `AuthService` instances would be a disaster.

The pattern is to elect a single owner (typically the shell), publish the shared instance through an out-of-band mechanism, and have each MFE provide a token whose factory reads from that registry:

```typescript
// libs/shared/auth/src/shared-auth.provider.ts
export interface SharedAuthApi {
  readonly user: Signal<User | null>;
  readonly token: Signal<string | null>;
  signIn(credentials: Credentials): Promise<void>;
  signOut(): Promise<void>;
}

export const SHARED_AUTH = new InjectionToken<SharedAuthApi>('SHARED_AUTH');

declare global {
  interface Window { __financialAppAuth?: SharedAuthApi; }
}

export function provideSharedAuth(): Provider {
  return {
    provide: SHARED_AUTH,
    useFactory: (): SharedAuthApi => {
      const shared = window.__financialAppAuth;
      if (!shared) throw new Error('Shared auth not installed. Shell must bootstrap first.');
      return shared;
    },
  };
}
```

The shell bootstraps first, constructs the real `AuthService`, and publishes it. Each MFE includes `provideSharedAuth()` in its application config, and components call `inject(SHARED_AUTH)` to receive the shell-owned instance. The DI trees remain isolated; only this one token bridges them. The token makes the contract explicit; the factory makes the lookup lazy.

---

## Tree-Shakable Providers

`providedIn: 'root'` is not just ergonomic shorthand -- it is the *tree-shakable* form of provider registration. The class carries its own registration metadata, and the bundler can statically see that if nothing imports `AccountService`, neither the class nor its provider reaches the bundle. Compare that to manually listing the service in `appConfig.providers`: the array imports the class unconditionally, so it ships whether or not anything injects it.

For services, always prefer `providedIn: 'root'`. For `InjectionToken` values, you can get the same benefit by passing a factory in the token definition:

```typescript
// libs/shared/data-access/src/clock.token.ts
export const CLOCK = new InjectionToken<() => Date>('CLOCK', {
  providedIn: 'root',
  factory: () => () => new Date(),
});
```

No `appConfig.providers` entry needed. `inject(CLOCK)` just works, tests can override via `TestBed`, and if nothing ever calls `inject(CLOCK)`, neither the token nor its default factory ships.

The exception is any value that depends on runtime input -- configuration loaded from a JSON file, factory-created singletons tied to the environment. Those must go in an explicit `providers` array because the factory cannot be known at token definition time. `providedIn` also accepts a component or environment type for niche scoping, but in practice, route `providers` or component `providers` arrays express that intent more clearly.

---

## Testing DI

Tests are where provider overrides earn their keep. `TestBed.configureTestingModule` takes a providers array just like the application config, and every FinancialApp spec uses it to replace collaborators with fakes:

```typescript
// libs/shared/data-access/src/account.service.spec.ts
describe('AccountService', () => {
  let service: AccountService;
  let http: HttpTestingController;

  beforeEach(() => {
    TestBed.configureTestingModule({
      providers: [
        provideHttpClient(),
        provideHttpClientTesting(),
        { provide: API_URL, useValue: 'https://test.example.com' },
      ],
    });
    service = TestBed.inject(AccountService);
    http = TestBed.inject(HttpTestingController);
  });

  it('prefixes requests with the configured API URL', () => {
    service.list().subscribe();
    http.expectOne('https://test.example.com/accounts').flush([]);
  });
});
```

The `{ provide: API_URL, useValue: ... }` entry overrides whatever the root injector would otherwise have provided. `TestBed.inject()` is the test-time equivalent of `inject()`. For per-test overrides, `TestBed.overrideProvider(API_URL, { useValue: '' })` replaces a provider without rebuilding the module; `TestBed.overrideComponent` does the same for providers declared on a component. Designing for injection is what makes testing straightforward.

---

## Anti-Patterns

A few patterns routinely cause pain.

**Service locator.** Injecting `Injector` and calling `injector.get(SomeToken)` on demand, instead of listing dependencies up front. It defeats the compiler's ability to verify dependencies and makes the class harder to test. Use `inject()` at the top of the class; reach for the `Injector` object only for `runInInjectionContext` or when the token itself is chosen at runtime.

**Injecting inside template expressions.** Templates are not injection contexts, and the framework does not re-evaluate `inject()` calls between change detection passes. Expose a signal or method on the component and bind to that instead.

**Circular dependencies.** When `AccountService` depends on `TransactionService`, and `TransactionService` depends on `AccountService`, Angular throws `NG0200: Circular dependency in DI detected`. Extract the shared logic into a third service, or pass data as arguments instead of holding references.

**Over-scoping services to fix bugs.** If state is leaking between navigations, scope the store to the route. But moving services off `providedIn: 'root'` "to be safe" creates duplicate instances, defeats tree-shaking, and makes cross-feature sharing harder. Scope for correctness, not superstition.

**Strings as keys.** Wherever you are tempted to pass a raw string to `@Inject` or `useExisting`, stop and create an `InjectionToken<T>`. One line of code buys type safety and a guaranteed-unique key.

---

## Summary

Angular's dependency injection is a small surface area with a lot of depth. For day-to-day work, the fast paths -- `providedIn: 'root'`, `inject()` at the top of a component, route `providers` -- cover almost everything. The machinery in this chapter is what you reach for when those fast paths stop being enough: `InjectionToken<T>` for non-class values, the four provider types for describing how an instance is built, multi-providers for aggregating contributions across features, and the hierarchical injector tree for scoping state to a route, component, or micro frontend.

`EnvironmentInjector`, `createEnvironmentInjector`, and `runInInjectionContext` are the tools for dynamic component creation and for keeping `inject()` working inside async callbacks. `optional`, `self`, `skipSelf`, and `host` cover the cases where the default tree walk is wrong. `providedIn: 'root'` and the `{ providedIn, factory }` token form let the bundler drop unused services. `TestBed.configureTestingModule` and `overrideProvider` close the loop -- design for injection, test by swapping providers.

The recurring theme is that DI is not just a convenience for wiring constructors -- it is the composition system. Every decision about where a provider lives is a decision about scope, lifetime, and coupling. When Angular's defaults don't quite fit, the tools in this chapter are how you take over.
