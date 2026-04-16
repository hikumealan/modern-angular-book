# Chapter 4: Navigation & Lazy Loading with the Router

## Overview

A single-page application renders a single HTML document. Every "page" the user visits is actually a swap of components inside that document, orchestrated by the router. Angular's router maps URL segments to components, manages browser history, and -- crucially -- controls *when* code is loaded. In a financial application where the accounts dashboard, transaction search, and portfolio drill-down each carry significant weight, loading everything up front punishes users who only came to check a balance.

This chapter walks through the router from first principles: configuring routes, linking between them, reading parameters, nesting child routes, and lazy loading entire feature areas so the browser only downloads what the user actually needs. All examples build on the FinancialApp introduced in [Chapter 1](ch01-getting-started.md). By the end, you will have a routing setup where each domain -- accounts, transactions, portfolios, and clients -- loads on demand, with deep-linkable URLs and programmatic navigation for workflow transitions.

> **Companion code:** The route configuration discussed here lives in `financial-app/apps/financial-app/src/app/app.routes.ts`, with feature-specific routes in each domain directory (e.g., `domains/portfolios/portfolio.routes.ts`).

---

## Getting Started with the Router

### Setting up Routing Configuration

Angular v21 applications configure routing through `provideRouter()`, registered in the application's `appConfig`. There is no `RouterModule.forRoot()` -- standalone applications wire the router as a provider function.

```typescript
// app.config.ts
import { ApplicationConfig } from '@angular/core';
import { provideRouter } from '@angular/router';
import { routes } from './app.routes';

export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(routes),
  ],
};
```

The `routes` array is a flat list of `Route` objects, each mapping a URL path to a component:

```typescript
// app.routes.ts
import { Routes } from '@angular/router';
import { DashboardComponent } from './domains/dashboard/dashboard.component';
import { AccountListComponent } from './domains/accounts/feature/account-list.component';
import { TransactionSearchComponent } from './domains/transactions/feature/transaction-search.component';

export const routes: Routes = [
  { path: '', redirectTo: 'dashboard', pathMatch: 'full' },
  { path: 'dashboard', component: DashboardComponent },
  { path: 'accounts', component: AccountListComponent },
  { path: 'transactions', component: TransactionSearchComponent },
];
```

The first route redirects the empty path to the dashboard. `pathMatch: 'full'` ensures the redirect only fires when the *entire* URL is empty, not when `/accounts` happens to start with an empty string.

### Adding a Placeholder in the App

Routes do not render themselves. You need a `<router-outlet>` in the host component's template to tell Angular where to insert the routed component:

```typescript
// app.component.ts
import { Component } from '@angular/core';
import { RouterOutlet } from '@angular/router';

@Component({
  selector: 'app-root',
  imports: [RouterOutlet],
  template: `
    <header>
      <h1>FinancialApp</h1>
      <nav><!-- links go here --></nav>
    </header>
    <main>
      <router-outlet />
    </main>
  `,
})
export class AppComponent {}
```

Everything outside the `<router-outlet>` -- the header, navigation, footer -- stays on screen as the user navigates. Only the content inside `<main>` changes.

### Setting up Hyperlinks to Activate Routes

Plain `<a href="/accounts">` would trigger a full-page reload, defeating the purpose of an SPA. Angular's `routerLink` directive intercepts the click and performs an in-app navigation instead:

```html
<nav>
  <a routerLink="/dashboard" routerLinkActive="active">Dashboard</a>
  <a routerLink="/accounts" routerLinkActive="active">Accounts</a>
  <a routerLink="/transactions" routerLinkActive="active">Transactions</a>
</nav>
```

The `routerLinkActive` directive adds the CSS class `active` to whichever link matches the current URL, making it straightforward to highlight the current section in the navigation bar. Both `RouterLink` and `RouterLinkActive` must be included in the component's `imports` array.

### Trying out Your Setup

With the configuration above, navigating to `/dashboard` renders the `DashboardComponent` inside the outlet. Clicking the "Accounts" link swaps it for `AccountListComponent` without a page reload. The browser's address bar updates, and the back button works as expected -- Angular pushes each navigation onto the browser's history stack.

If a user enters a URL that does not match any configured route, Angular will throw an error in development and render nothing in production. Adding a wildcard route catches these cases:

```typescript
{ path: '**', redirectTo: 'dashboard' }
```

Place the wildcard route **last** in the array. The router evaluates routes in order and uses the first match.

### Programmatically Changing Routes

Not every navigation originates from a link click. After saving a new transaction, you might want to redirect the user back to the transaction list. Inject the `Router` service and call `navigate()`:

```typescript
import { Component, inject } from '@angular/core';
import { Router } from '@angular/router';
import { TransactionService } from '../data-access/transaction.service';

@Component({ /* ... */ })
export class TransactionFormComponent {
  private router = inject(Router);
  private transactionService = inject(TransactionService);

  async save(transaction: TransactionDraft): Promise<void> {
    await this.transactionService.create(transaction);
    this.router.navigate(['/transactions']);
  }
}
```

`navigate()` accepts an array of path segments and returns a `Promise<boolean>` that resolves to `true` if the navigation succeeds. You can also pass query parameters and fragments as the second argument:

```typescript
this.router.navigate(['/accounts'], {
  queryParams: { type: 'savings' },
  fragment: 'summary',
});
// Produces: /accounts?type=savings#summary
```

---

## Parameterized Routes

Static routes work fine for listing pages, but detail views need to identify *which* entity to display. A transaction detail page must know the transaction ID; a portfolio page must know which portfolio the user selected.

### Types of Routing Parameters

Angular supports three kinds of parameters:

| Type | Syntax | Example URL | Use case |
|---|---|---|---|
| **Path parameter** | `:id` | `/accounts/42` | Identifying a specific resource |
| **Query parameter** | `?key=val` | `/transactions?status=pending` | Filtering, optional state |
| **Fragment** | `#section` | `/accounts/42#activity` | Scrolling to a page section |

Path parameters are the right choice when the parameter is *required* and *defines* the resource. Query parameters suit optional, additive concerns like sorting and filtering.

### Reading Parameters with ActivatedRoute

The traditional approach uses `ActivatedRoute`, which exposes route data as observables:

```typescript
import { Component, inject } from '@angular/core';
import { ActivatedRoute } from '@angular/router';

@Component({ /* ... */ })
export class AccountDetailComponent {
  private route = inject(ActivatedRoute);

  ngOnInit(): void {
    this.route.paramMap.subscribe(params => {
      const accountId = Number(params.get('id'));
      this.loadAccount(accountId);
    });
  }
}
```

This works, but it introduces imperative subscription management and mixes lifecycle hooks with routing logic. Angular v21 offers a cleaner alternative.

### Using withComponentInputBinding() for Reading Parameters

The modern approach binds route parameters directly to component inputs. Enable it once in `appConfig`:

```typescript
// app.config.ts
export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(routes, withComponentInputBinding()),
  ],
};
```

Now any route parameter, query parameter, or resolved data can flow into the component through a standard `input()`:

```typescript
import { Component, input, inject } from '@angular/core';
import { AccountService } from '../data-access/account.service';

@Component({
  selector: 'app-account-detail',
  template: `
    @if (account(); as acct) {
      <h2>{{ acct.name }}</h2>
      <p>Balance: {{ acct.balance | currency:acct.currency }}</p>
    }
  `,
})
export class AccountDetailComponent {
  id = input.required<string>();

  private accountService = inject(AccountService);
  account = this.accountService.getAccount(this.id);
}
```

The input name `id` must match the route parameter name `:id`. Angular handles the binding automatically -- no subscriptions, no `ngOnInit`, no cleanup. This is the preferred approach throughout this book.

### Configuring Parameterized Routes

Add the parameter token to the route path:

```typescript
export const routes: Routes = [
  { path: 'accounts', component: AccountListComponent },
  { path: 'accounts/:id', component: AccountDetailComponent },
  { path: 'transactions', component: TransactionSearchComponent },
  { path: 'transactions/:transactionId', component: TransactionDetailComponent },
];
```

When the URL is `/accounts/42`, the router matches the `accounts/:id` route and activates `AccountDetailComponent` with `id` bound to `"42"`. Note that the value arrives as a string -- numeric conversion is the component's responsibility.

### Linking to Parameterized Routes

Use `routerLink` with an array of segments:

```html
@for (account of accounts(); track account.id) {
  <a [routerLink]="['/accounts', account.id]">
    {{ account.name }}
  </a>
}
```

The array form `['/accounts', account.id]` concatenates into `/accounts/42`. This is safer than string interpolation because Angular encodes special characters in each segment.

---

## Hierarchical Routing with Child Routes

### Overview of Child Routes

Flat routes work until your UI requires nested layouts. The portfolios section of FinancialApp is a good example: a shell component displays a portfolio header and sidebar, while the content area switches between a holdings overview, performance charts, and rebalancing tools. These inner views should not replace the entire page -- they need their own `<router-outlet>` nested inside the portfolio shell.

Child routes solve this. A parent route renders a shell component with a `<router-outlet>`, and child routes fill that outlet with their own components.

### Implementing Child Components

The portfolio shell provides the persistent layout:

```typescript
// portfolio-shell.component.ts
import { Component, input } from '@angular/core';
import { RouterOutlet, RouterLink, RouterLinkActive } from '@angular/router';

@Component({
  selector: 'app-portfolio-shell',
  imports: [RouterOutlet, RouterLink, RouterLinkActive],
  template: `
    <section class="portfolio-shell">
      <nav class="portfolio-nav">
        <a [routerLink]="['holdings']" routerLinkActive="active">Holdings</a>
        <a [routerLink]="['performance']" routerLinkActive="active">Performance</a>
        <a [routerLink]="['rebalance']" routerLinkActive="active">Rebalance</a>
      </nav>
      <div class="portfolio-content">
        <router-outlet />
      </div>
    </section>
  `,
})
export class PortfolioShellComponent {
  portfolioId = input.required<string>();
}
```

Each child -- `HoldingsComponent`, `PerformanceComponent`, `RebalanceComponent` -- is a standalone component that renders inside the shell's `<router-outlet>`. The shell stays on screen as the user switches between tabs.

### Configuring Child Routes

Child routes are declared in the `children` array of the parent route:

```typescript
// domains/portfolios/portfolio.routes.ts
import { Routes } from '@angular/router';
import { PortfolioShellComponent } from './feature/portfolio-shell.component';
import { HoldingsComponent } from './feature/holdings.component';
import { PerformanceComponent } from './feature/performance.component';
import { RebalanceComponent } from './feature/rebalance.component';

export const portfolioRoutes: Routes = [
  {
    path: ':portfolioId',
    component: PortfolioShellComponent,
    children: [
      { path: '', redirectTo: 'holdings', pathMatch: 'full' },
      { path: 'holdings', component: HoldingsComponent },
      { path: 'performance', component: PerformanceComponent },
      { path: 'rebalance', component: RebalanceComponent },
    ],
  },
];
```

These are wired into the top-level routes under the `portfolios` prefix:

```typescript
// app.routes.ts (excerpt)
{ path: 'portfolios', children: portfolioRoutes },
```

Navigating to `/portfolios/7/holdings` activates `PortfolioShellComponent` with `portfolioId` bound to `"7"`, and renders `HoldingsComponent` inside the shell's outlet. The empty-path redirect ensures that `/portfolios/7` automatically lands on the holdings tab.

### Hyperlinks to Child Routes

Inside the shell, relative links are the cleanest approach. The `routerLink` values `['holdings']` and `['performance']` resolve relative to the shell's route, producing URLs like `/portfolios/7/holdings` without hard-coding the portfolio ID.

From *outside* the shell -- say, a portfolio list -- use absolute paths:

```html
@for (portfolio of portfolios(); track portfolio.id) {
  <a [routerLink]="['/portfolios', portfolio.id, 'holdings']">
    {{ portfolio.name }}
  </a>
}
```

---

## Lazy Loading of Routes

### Setting up Routes for Lazy Loading

Every `import` statement in your route configuration pulls the imported component and its dependencies into the main bundle. For FinancialApp, eagerly importing all four domains means the user downloads accounts, transactions, portfolios, and clients code before they see the dashboard.

Lazy loading defers this cost. Instead of importing the component directly, you point the route at a function that returns a dynamic `import()`. The router calls this function only when the user first navigates to that route, triggering a separate HTTP request for that chunk.

The two primary mechanisms are `loadChildren` (for a set of child routes) and `loadComponent` (for a single component).

### Lazy Loading Individual Components with the Router

For simple cases where a route maps to a single component with no children, use `loadComponent`:

```typescript
// app.routes.ts
export const routes: Routes = [
  { path: '', redirectTo: 'dashboard', pathMatch: 'full' },
  {
    path: 'dashboard',
    loadComponent: () =>
      import('./domains/dashboard/dashboard.component')
        .then(m => m.DashboardComponent),
  },
  {
    path: 'accounts',
    loadComponent: () =>
      import('./domains/accounts/feature/account-list.component')
        .then(m => m.AccountListComponent),
  },
  {
    path: 'accounts/:id',
    loadComponent: () =>
      import('./domains/accounts/feature/account-detail.component')
        .then(m => m.AccountDetailComponent),
  },
];
```

For feature areas with nested routing, `loadChildren` loads an entire route configuration file:

```typescript
{
  path: 'portfolios',
  loadChildren: () =>
    import('./domains/portfolios/portfolio.routes')
      .then(m => m.portfolioRoutes),
},
{
  path: 'transactions',
  loadChildren: () =>
    import('./domains/transactions/transaction.routes')
      .then(m => m.transactionRoutes),
},
```

The `portfolio.routes.ts` file exports the child route array directly -- there is no NgModule wrapping it. The router merges these routes under the `portfolios` prefix and handles the lazy chunk automatically.

When refactoring from eager to lazy loading, the key change is mechanical: remove the static `import` at the top of the file and replace the `component` property with `loadComponent` (or `children` with `loadChildren`). The component code itself does not change.

### Verify Lazy Loading in the Browser

Open DevTools, switch to the Network tab, and filter by JS. On initial load, you should see the main bundle and vendor chunk. Now click the "Portfolios" link -- a new chunk file appears in the network waterfall. That chunk contains `PortfolioShellComponent`, `HoldingsComponent`, and the rest of the portfolios domain. If you do not see a separate request, the route is still eagerly loaded; check for stray static imports.

The Sources panel in Chrome DevTools also shows lazy chunks under `webpack://` or `vite://` (depending on your build tool). You can set breakpoints inside lazy-loaded code to confirm it was not included in the initial bundle.

### Preloading

Lazy loading eliminates wasted bandwidth, but it introduces a navigation delay: the first visit to a lazy route must wait for the chunk to download. Preloading strategies let you reclaim that speed after the initial load completes.

Angular ships with `PreloadAllModules`, which begins fetching all lazy chunks in the background once the application is stable:

```typescript
import { provideRouter, withPreloadingStrategy, PreloadAllModules } from '@angular/router';

export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(
      routes,
      withComponentInputBinding(),
      withPreloadingStrategy(PreloadAllModules),
    ),
  ],
};
```

This is a reasonable default for applications with a handful of lazy routes. For larger applications where even background preloading is too aggressive, you can implement a custom `PreloadingStrategy` that selectively preloads routes based on a `data` flag:

```typescript
import { PreloadingStrategy, Route } from '@angular/router';
import { Observable, of, EMPTY } from 'rxjs';

export class SelectivePreloadingStrategy implements PreloadingStrategy {
  preload(route: Route, load: () => Observable<any>): Observable<any> {
    return route.data?.['preload'] ? load() : EMPTY;
  }
}
```

Then tag individual routes:

```typescript
{
  path: 'accounts',
  loadChildren: () => import('./domains/accounts/account.routes').then(m => m.accountRoutes),
  data: { preload: true },
},
```

Accounts is preloaded because users visit it frequently. The less-visited client management section loads only on demand.

---

## Working with Query Strings and Hash Fragments

Query parameters and hash fragments supplement the primary route without changing which component the router activates. They are ideal for filter state, pagination offsets, and in-page anchors.

Set query parameters through `routerLink`:

```html
<a [routerLink]="['/transactions']"
   [queryParams]="{ status: 'pending', page: 1 }">
  Pending Transactions
</a>
<!-- Produces: /transactions?status=pending&page=1 -->
```

Read them with `withComponentInputBinding()` -- the same mechanism used for path parameters:

```typescript
@Component({ /* ... */ })
export class TransactionSearchComponent {
  status = input<string>();
  page = input<string>();
}
```

To *preserve* query parameters during navigation (for instance, keeping the active filter when the user clicks into a transaction and then navigates back), use `queryParamsHandling`:

```typescript
this.router.navigate(['/transactions', txId], {
  queryParamsHandling: 'preserve',
});
```

Hash fragments scroll the viewport to an element with a matching `id`. Enable this behavior globally:

```typescript
provideRouter(
  routes,
  withComponentInputBinding(),
  withInMemoryScrolling({ anchorScrolling: 'enabled' }),
),
```

Now navigating to `/accounts/42#activity` scrolls to the element `<section id="activity">` inside the account detail view.

---

## Path Routing vs. Hash Routing

Angular supports two strategies for representing routes in the browser's URL.

### PathLocationStrategy

The default strategy uses the HTML5 History API. URLs look like normal paths: `/accounts/42`, `/portfolios/7/holdings`. The browser does not reload on navigation because `history.pushState()` updates the address bar without a server request.

This strategy requires server-side cooperation. When a user bookmarks `/portfolios/7/holdings` and later opens it directly, the server must return the Angular `index.html` for that path. Without a catch-all rule, the server returns a 404. Most hosting platforms support this through rewrite rules:

```
# nginx
location / {
  try_files $uri $uri/ /index.html;
}
```

PathLocationStrategy is the default and the recommended choice. It produces clean URLs, supports server-side rendering, and works with standard web infrastructure.

### HashLocationStrategy

Hash-based routing places the route after a `#` symbol: `http://localhost:4200/#/accounts/42`. The browser never sends the fragment to the server, so no rewrite rules are needed. The Angular application reads the fragment and resolves the route client-side.

Enable it with `withHashLocation()`:

```typescript
provideRouter(routes, withHashLocation())
```

Hash routing can be useful in constrained environments where you cannot control server configuration -- embedded apps, file-based deployments, or legacy infrastructure. The tradeoff is aesthetics (URLs are less readable), limited SSR compatibility, and the loss of the hash fragment for in-page anchors (since the router claims the `#` for routing).

For FinancialApp, PathLocationStrategy is the clear choice. The application is deployed to infrastructure that supports rewrite rules, and clean URLs matter for bookmarkability and sharing.

---

## Summary

The Angular router transforms a collection of standalone components into a navigable application with deep-linkable URLs and browser history integration. This chapter covered the essential patterns:

- **Route configuration** with `provideRouter()` and a flat `Routes` array, registered in `appConfig`.
- **Declarative navigation** with `routerLink` and `routerLinkActive` for template-driven links.
- **Programmatic navigation** with `inject(Router)` for workflow-driven redirects after form submissions.
- **Parameterized routes** using `:param` tokens in paths, read through `withComponentInputBinding()` and standard `input()` signals -- the preferred alternative to subscribing to `ActivatedRoute` observables.
- **Child routes** for nested layouts like the portfolio shell, where a parent component provides persistent chrome and a nested `<router-outlet>` for child views.
- **Lazy loading** with `loadComponent` and `loadChildren`, verified through the browser's Network tab, and optimized with preloading strategies.
- **Query strings and fragments** for optional state like filters and in-page anchors.
- **Path vs. hash routing** -- prefer `PathLocationStrategy` unless server-side rewrite rules are impossible.

What the router does *not* handle is controlling *access* to routes. Guards like `CanActivateFn` and resolvers like `ResolveFn` are covered in [Chapter 12](ch12-initialization-routes.md), where we use them to protect routes behind authentication checks and pre-fetch data before a component renders.
