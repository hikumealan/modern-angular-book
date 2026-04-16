# Initialization & Route Changes

Before the first component renders, Angular runs through initialization steps: loading configuration, establishing connections, verifying tokens. After bootstrap, every route change triggers further decisions -- should this navigation be allowed? Does the target route need pre-fetched data? Should outgoing requests carry credentials? This chapter covers the mechanisms Angular provides for hooking into these moments. We will wire up initializers, guards, resolvers, and interceptors for the FinancialApp, whose portfolio routes require authentication, whose transaction entry form needs unsaved-changes protection, and whose API calls must carry an auth token.

> **Prerequisites:** Familiarity with Angular's router configuration ([Chapter 4](ch04-router.md)). Authentication concepts referenced here are explored in depth in [Chapter 16](ch16-auth-patterns.md).

---

## Initializers

Initializers run before the application becomes interactive -- loading remote configuration, warming a cache, or verifying that an auth token is still valid.

### Application Initializers

An application initializer is a function registered with `provideAppInitializer()`. Angular calls every registered initializer during bootstrap and waits for any returned `Promise` or `Observable` to settle before rendering the root component. In the FinancialApp, we load environment-specific configuration from a JSON endpoint:

```typescript
// financial-app/.../shared/initializers/config.initializer.ts
export function initializeConfig(): () => Promise<void> {
  const http = inject(HttpClient);
  const config = inject(ConfigService);

  return () =>
    firstValueFrom(http.get('/assets/config.json')).then((data) =>
      config.load(data)
    );
}
```

Register it in the application's provider array:

```typescript
export const appConfig: ApplicationConfig = {
  providers: [
    provideAppInitializer(initializeConfig()),
    // ...other providers
  ],
};
```

Angular resolves `inject()` calls against the application's injector. If the initializer's promise rejects, Angular aborts bootstrap entirely -- a useful guarantee when the application cannot function without configuration.

### Environment Initializers

Environment initializers run in the context of a specific environment injector rather than the root injector. They are useful when a lazy-loaded route needs setup work scoped to its own injector. The FinancialApp's portfolios feature loads market-data WebSocket configuration only when the user navigates to a portfolio route:

```typescript
// financial-app/.../features/portfolios/portfolios.routes.ts
export const PORTFOLIO_ROUTES: Routes = [
  {
    path: '',
    providers: [
      {
        provide: ENVIRONMENT_INITIALIZER,
        multi: true,
        useValue: () => inject(MarketDataService).connect(),
      },
    ],
    children: [
      { path: '', component: PortfolioListComponent },
      { path: ':id', component: PortfolioDetailComponent },
    ],
  },
];
```

The initializer runs once when Angular creates the environment injector for this route -- not on every navigation to a child. This distinction matters for expensive setup like WebSocket connections.

### Platform Initializers

Platform initializers run even earlier -- during platform creation, before any application bootstraps. They are rarely needed, but serve a purpose in multi-application setups:

```typescript
import { PLATFORM_INITIALIZER } from '@angular/core';

export const platformProviders = [
  {
    provide: PLATFORM_INITIALIZER,
    multi: true,
    useValue: () => performance.mark('platform-init'),
  },
];
```

For the FinancialApp -- a single-application deployment -- application initializers cover every practical need. Platform initializers become relevant when you embed multiple Angular applications on the same page (see [Chapter 18](ch18-micro-frontends.md)).

---

## Guards

Guards intercept navigation and decide whether it should proceed. Angular's router calls them at specific moments during the navigation lifecycle, giving you the chance to redirect unauthorized users or protect unsaved data.

### Preventing Route Activation

A `CanActivateFn` runs before the router activates a route. If it returns `false` or a `UrlTree`, the navigation is cancelled or redirected. The FinancialApp protects portfolio routes behind an authentication check:

```typescript
// financial-app/.../shared/guards/auth.guard.ts
export const authGuard: CanActivateFn = () => {
  const auth = inject(AuthService);
  const router = inject(Router);

  if (auth.isAuthenticated()) {
    return true;
  }

  return router.createUrlTree(['/login'], {
    queryParams: { returnUrl: router.getCurrentNavigation()?.extractedUrl.toString() },
  });
};
```

The guard returns a `UrlTree` for redirection rather than calling `router.navigate()`. This is preferred because the redirect participates in the same navigation cycle, and other guards see a consistent state. Apply it in the route configuration:

```typescript
{
  path: 'portfolios',
  canActivate: [authGuard],
  loadChildren: () => import('./features/portfolios/portfolios.routes')
    .then(m => m.PORTFOLIO_ROUTES),
}
```

Multiple guards on the same route run concurrently. If any guard returns `false` or a redirect, the navigation fails. Design your guards to be independent of each other -- do not rely on execution order.

### Preventing Route Deactivation

A `CanDeactivateFn` runs when the user navigates *away* from a route. It receives the component instance being deactivated, which lets you inspect component state -- like whether a form has unsaved changes.

The FinancialApp uses this to protect the transaction entry form:

```typescript
// financial-app/.../shared/guards/unsaved-changes.guard.ts
export interface HasUnsavedChanges {
  hasUnsavedChanges(): boolean;
}

export const unsavedChangesGuard: CanDeactivateFn<HasUnsavedChanges> = (
  component
) => {
  if (component.hasUnsavedChanges()) {
    return window.confirm(
      'You have unsaved changes. Are you sure you want to leave?'
    );
  }
  return true;
};
```

Any component implementing `HasUnsavedChanges` can use this guard. The transaction entry component implements the interface and wires the guard to its route:

```typescript
@Component({ /* ... */ })
export class TransactionEntryComponent implements HasUnsavedChanges {
  private form = inject(FormBuilder).group({ /* ... */ });
  hasUnsavedChanges(): boolean { return this.form.dirty; }
}

// Route configuration
{ path: 'new', component: TransactionEntryComponent, canDeactivate: [unsavedChangesGuard] }
```

For production, replace `window.confirm` with a Material dialog. The guard can return an `Observable<boolean>`, so asynchronous confirmation flows work naturally.

---

## Router Events

The router emits a stream of events describing every phase of navigation. Subscribe to this stream for analytics, loading indicators, or debugging:

```typescript
// financial-app/apps/web/src/app/core/router-analytics.init.ts
import { inject } from '@angular/core';
import { Router, NavigationEnd } from '@angular/router';
import { filter } from 'rxjs';

export function initRouterAnalytics(): void {
  inject(Router)
    .events.pipe(filter((e) => e instanceof NavigationEnd))
    .subscribe((e) => analytics.trackPageView(e.urlAfterRedirects));
}
```

The most commonly observed events, in order:

| Event               | When it fires                              |
| ------------------- | ------------------------------------------ |
| `NavigationStart`   | A new navigation begins.                   |
| `GuardsCheckStart`  | The router begins evaluating guards.       |
| `GuardsCheckEnd`    | All guards have resolved.                  |
| `ResolveStart`      | The router begins running resolvers.       |
| `ResolveEnd`        | All resolvers have completed.              |
| `NavigationEnd`     | The navigation finishes successfully.      |
| `NavigationCancel`  | A guard rejected the navigation.           |
| `NavigationError`   | An unrecoverable error occurred.           |

A practical use case: show a spinner on `NavigationStart`, hide it on `NavigationEnd`, `NavigationCancel`, or `NavigationError`. Avoid coupling business logic to router events -- guards and resolvers are better tools for controlling navigation behavior.

---

## Resolver

A `ResolveFn` fetches data before the router activates a route. The resolved data is available through `ActivatedRoute.data`, so the component never handles a "loading" state for its initial render. The FinancialApp resolves account details before displaying the account page:

```typescript
// financial-app/.../shared/resolvers/account.resolver.ts
export const accountResolver: ResolveFn<Account> = (route) => {
  const accountService = inject(AccountService);
  return accountService.getById(route.paramMap.get('id')!);
};

// Route configuration
{ path: 'accounts/:id', component: AccountDetailComponent, resolve: { account: accountResolver } }
```

The component reads the resolved data via `ActivatedRoute`:

```typescript
@Component({ /* ... */ })
export class AccountDetailComponent {
  private route = inject(ActivatedRoute);
  account = toSignal(this.route.data.pipe(map((d) => d['account'] as Account)));
}
```

Resolvers delay navigation until the data is ready. For slow endpoints, this means the user stares at the previous page with no feedback. Consider pairing resolvers with a global loading indicator driven by `ResolveStart` and `ResolveEnd` router events, or fetching data inside the component with explicit loading state when responsiveness matters more than data-readiness guarantees.

---

## HttpInterceptors

Interceptors sit in the HTTP pipeline and transform outgoing requests or incoming responses. Every `HttpClient` call passes through the interceptor chain, making interceptors the right place for cross-cutting concerns like authentication headers, error normalization, or request logging. An interceptor is a plain function matching `HttpInterceptorFn`:

```typescript
// financial-app/.../shared/interceptors/auth-token.interceptor.ts
export const authTokenInterceptor: HttpInterceptorFn = (req, next) => {
  const auth = inject(AuthService);
  const token = auth.getToken();

  if (token) {
    const cloned = req.clone({
      setHeaders: { Authorization: `Bearer ${token}` },
    });
    return next(cloned);
  }

  return next(req);
};
```

Register interceptors with `provideHttpClient()` using `withInterceptors()`:

```typescript
export const appConfig: ApplicationConfig = {
  providers: [
    provideHttpClient(withInterceptors([authTokenInterceptor])),
  ],
};
```

Interceptors run in the order they appear in the array. Place authentication first so that subsequent interceptors (logging, retry logic) see the fully formed request. Each interceptor calls `next(req)` to pass control to the next interceptor in the chain, and receives the response `Observable` flowing back. For a deeper treatment of how interceptors coordinate with OAuth flows and BFF proxies, see [Chapter 16](ch16-auth-patterns.md).

---

## Summary

Angular provides a layered set of hooks that let you control what happens before, during, and after key application events:

- **Initializers** run setup logic before the application renders. Use `provideAppInitializer()` for bootstrap-time work, `ENVIRONMENT_INITIALIZER` for lazy-loaded scope, and `PLATFORM_INITIALIZER` for the rare multi-app scenario.
- **Guards** control navigation. `CanActivateFn` prevents unauthorized access; `CanDeactivateFn` protects against data loss. Both return booleans, `UrlTree` redirects, or observables for async decisions.
- **Router events** provide an observable stream of navigation lifecycle phases, useful for analytics and loading indicators.
- **Resolvers** pre-fetch data so components render with everything they need, at the cost of delaying navigation.
- **Interceptors** transform every HTTP request and response in the pipeline. Use `withInterceptors()` to register them with `provideHttpClient()`.

All of these APIs are fully functional in modern Angular -- no classes, no interfaces to implement for DI registration, no `NgModule` ceremony. A guard is a function. An interceptor is a function. A resolver is a function. They use `inject()` for dependencies and compose naturally with the rest of Angular's standalone architecture.
