# Defer, SSR & Hydration

A user opens FinancialApp's portfolio overview. The page should render instantly -- or at least feel like it does. The account summary and navigation must appear immediately, but the full holdings table with its charting library and the transaction history with its virtualized scroll can wait until the user actually needs them. Meanwhile, search engines and social media crawlers need to see fully rendered account detail pages without executing a single line of JavaScript.

These two problems -- loading less code upfront and rendering HTML on the server -- converge in Angular v21 through deferrable views, server-side rendering, and incremental hydration. This chapter shows how to use all three, how they interact, and how hybrid routing lets you choose the right rendering strategy per route.

---

## Deferrable Views

Angular's `@defer` block is a template-level primitive for lazy loading. Unlike route-level code splitting (covered in [Chapter 4](ch04-router.md)), `@defer` works within a single component's template. It tells the compiler to extract the deferred content -- and all of its transitive dependencies -- into a separate JavaScript chunk that is loaded only when a trigger condition is met.

For FinancialApp's portfolio overview, this means the holdings table component, its charting library, and the transaction history component never appear in the main bundle. They load on demand, keeping the initial payload focused on the account summary and navigation shell.

### Using Defer Blocks

A `@defer` block wraps any section of a template. The compiler splits the wrapped content into its own chunk automatically -- no manual dynamic imports or `loadComponent` calls required.

```html
<!-- portfolio-overview.component.html -->
<app-account-summary [account]="account()" />

@defer (on viewport) {
  <app-holdings-table [holdings]="holdings()" />
} @placeholder {
  <div class="skeleton-table" aria-busy="true">Loading holdings...</div>
} @loading (minimum 300ms) {
  <mat-spinner diameter="32" />
} @error {
  <app-inline-error message="Could not load holdings." />
}
```

Four companion blocks control what the user sees at each stage of the loading lifecycle:

- **`@placeholder`** -- rendered immediately and remains visible until the trigger fires. This is where skeleton screens belong. The placeholder content is bundled with the parent component, so keep it lightweight.
- **`@loading`** -- replaces the placeholder once the chunk download begins. The optional `minimum` parameter prevents a flash of loading state on fast connections.
- **`@error`** -- displayed if the chunk fails to load (network error, CDN outage). Without this block, a failed load renders nothing.
- The main `@defer` body -- rendered after the chunk downloads and the deferred component initializes.

The `@placeholder` block accepts a `minimum` parameter as well, ensuring it stays visible for at least a specified duration. This avoids a jarring flicker when the deferred content loads near-instantly on fast networks:

```html
@defer (on viewport) {
  <app-transaction-history [accountId]="accountId()" />
} @placeholder (minimum 500ms) {
  <app-transaction-skeleton />
}
```

### Triggers

Triggers control *when* the deferred chunk begins loading. Angular provides several built-in triggers, and you can combine them.

**`on viewport`** -- loads when the placeholder element scrolls into the browser's viewport, using an `IntersectionObserver` internally. This is the most common trigger for below-the-fold content.

```html
@defer (on viewport) {
  <app-holdings-table [holdings]="holdings()" />
} @placeholder {
  <div style="height: 400px" class="skeleton-table"></div>
}
```

The placeholder's dimensions matter. An `IntersectionObserver` needs a measurable element to observe, so give the placeholder an explicit height that approximates the final content.

**`on interaction`** -- loads when the user interacts (click or keydown) with the placeholder or a referenced trigger element. Useful for content hidden behind a tab or expandable panel.

```html
<button #showHistory>Show Transaction History</button>

@defer (on interaction(showHistory)) {
  <app-transaction-history [accountId]="accountId()" />
} @placeholder {
  <p>Click above to load transaction history.</p>
}
```

**`on hover`** -- loads when the user hovers over the trigger element, giving the chunk a head start before a click. This works well paired with `on interaction` as a prefetch strategy:

```html
@defer (on hover(showHistory); on interaction(showHistory)) {
  <app-transaction-history [accountId]="accountId()" />
}
```

**`on timer`** -- loads after a fixed delay, regardless of user interaction. Use this for content that is almost always needed but can wait a beat after the critical path renders.

```html
@defer (on timer(2s)) {
  <app-market-news />
} @placeholder {
  <app-news-skeleton />
}
```

**`on idle`** -- loads when the browser is idle, using `requestIdleCallback`. This is the gentlest trigger -- it waits until the main thread has breathing room, making it ideal for non-critical content.

```html
@defer (on idle) {
  <app-portfolio-footer />
}
```

**`when` -- conditional trigger** -- loads when a boolean expression becomes `true`. This is the most flexible trigger and integrates naturally with signals.

```html
@defer (when showAdvancedAnalytics()) {
  <app-advanced-analytics [portfolio]="portfolio()" />
} @placeholder {
  <button (click)="showAdvancedAnalytics.set(true)">
    Load Advanced Analytics
  </button>
}
```

**Prefetching.** Every trigger type supports a separate `prefetch` condition that begins downloading the chunk before the main trigger fires, without rendering the content. This lets you overlap network time with user intent signals:

```html
@defer (on interaction(detailsBtn); prefetch on hover(detailsBtn)) {
  <app-account-details [account]="account()" />
}
```

Here the chunk starts downloading when the user hovers, but the component only renders when they click.

---

## SSR & Hydration

Server-side rendering generates the initial HTML on the server and sends it to the browser, where Angular **hydrates** the static markup -- attaching event listeners and initializing component state -- without destroying and re-creating the DOM. The user sees content before JavaScript finishes loading, and search engine crawlers see fully rendered pages.

For FinancialApp, SSR is essential for account detail pages. These pages are indexed by internal search systems, linked from email notifications, and must display meaningful content within the first paint -- even on slow mobile connections.

### Adding Server-Side Rendering

Angular v21 provides SSR through the `@angular/ssr` package. If you are starting a new project, the CLI includes SSR by default. For an existing project, add it with a schematic:

```bash
ng add @angular/ssr
```

This command generates or modifies several files:

- **`server.ts`** -- the Express server entry point that handles incoming requests and calls Angular's server rendering engine.
- **`app.config.server.ts`** -- server-specific providers, merged with the client-side `app.config.ts`.
- **`tsconfig.server.json`** -- TypeScript configuration for the server build.

The server-side application configuration provides the server rendering primitives:

```typescript
// app.config.server.ts
import { ApplicationConfig, mergeApplicationConfig } from '@angular/core';
import { provideServerRendering } from '@angular/platform-server';
import { provideServerRoutesConfig } from '@angular/ssr';
import { appConfig } from './app.config';
import { serverRoutes } from './app.routes.server';

const serverConfig: ApplicationConfig = {
  providers: [
    provideServerRendering(),
    provideServerRoutesConfig(serverRoutes),
  ],
};

export const config = mergeApplicationConfig(appConfig, serverConfig);
```

Angular v21 defaults to **zoneless change detection**, and this applies to the server as well. The server rendering engine uses signals and the framework's internal scheduler to determine when the application has stabilized -- no `zone.js` is loaded on the server. This simplifies the server runtime, reduces the server-side bundle size, and eliminates an entire class of `NgZone`-related SSR bugs where asynchronous operations (HTTP calls, timers) prevented the server from knowing when rendering was complete.

### Incremental Hydration

Standard hydration attaches Angular to all server-rendered DOM at once. For a large page like FinancialApp's portfolio overview, this means hydrating the account summary, the holdings table, the transaction history, and the footer in a single pass -- even though the user has not scrolled to the bottom of the page yet.

**Incremental hydration** combines `@defer` with a `hydrate` trigger. The server renders the full HTML for the deferred block (so crawlers and the first paint see complete content), but the client only hydrates it when the trigger fires. Until then, the static HTML remains visible but inert.

```html
<!-- portfolio-overview.component.html -->
<app-account-summary [account]="account()" />

@defer (on viewport; hydrate on viewport) {
  <app-holdings-table [holdings]="holdings()" />
} @placeholder {
  <div class="skeleton-table">Loading holdings...</div>
}

@defer (on interaction(showHistory); hydrate on interaction) {
  <app-transaction-history [accountId]="accountId()" />
} @placeholder {
  <p>Click to view transaction history.</p>
}
```

In this example, the server renders both the holdings table and the transaction history as full HTML. The client hydrates the holdings table only when it scrolls into the viewport, and the transaction history only when the user clicks on it. The JavaScript for each component is downloaded and executed on demand, not upfront.

The hydrate triggers mirror the standard `@defer` triggers: `hydrate on viewport`, `hydrate on interaction`, `hydrate on hover`, `hydrate on timer`, `hydrate on idle`, and `hydrate when condition`. The `hydrate never` option tells Angular to leave the server-rendered HTML permanently static -- useful for purely decorative content that has no interactive behavior.

```html
@defer (hydrate never) {
  <app-regulatory-disclosure />
}
```

### Event Replay

A problem emerges with incremental hydration: what happens when a user clicks a button before its component has hydrated? Without intervention, the click is lost. The user taps "Buy" on a holdings row, nothing happens, and they wonder if the application is broken.

Angular solves this with **event replay**. During server rendering, Angular injects a small inline script (the event dispatch script) that captures user interactions -- clicks, key presses, form inputs -- on elements that have not yet been hydrated. Once the target component hydrates, Angular replays the captured events against the now-interactive component. The user's intent is preserved.

Event replay is enabled by default when using `provideClientHydration()` with incremental hydration. You can configure it explicitly:

```typescript
// app.config.ts
import { provideClientHydration, withEventReplay } from '@angular/platform-browser';

export const appConfig: ApplicationConfig = {
  providers: [
    provideClientHydration(withEventReplay()),
  ],
};
```

The replay mechanism is lightweight: the event dispatch script is under 3 KB and uses event delegation at the document level. It captures the event type, the target element (identified by a stable attribute), and basic event properties. When hydration completes, Angular dispatches synthetic events that mimic the originals. For FinancialApp, this means a user can click "View Transactions" on a server-rendered account card before the transaction history component has hydrated, and the click will execute once hydration finishes.

### Different Implementations for Server and Client

Some code simply cannot run in both environments. The server has no `window`, no `document`, no `localStorage`. The client has no access to the filesystem or incoming HTTP headers. When a component depends on browser-specific APIs, you need a strategy for providing alternative behavior on the server.

#### Different Implementations via DI

The cleanest approach uses Angular's dependency injection to swap implementations based on the platform. Define an abstract interface, provide a browser implementation for the client, and a no-op or alternative implementation for the server.

```typescript
// storage.service.ts
export abstract class StorageService {
  abstract getItem(key: string): string | null;
  abstract setItem(key: string, value: string): void;
}

// browser-storage.service.ts
@Injectable()
export class BrowserStorageService extends StorageService {
  getItem(key: string): string | null {
    return localStorage.getItem(key);
  }
  setItem(key: string, value: string): void {
    localStorage.setItem(key, value);
  }
}

// server-storage.service.ts
@Injectable()
export class ServerStorageService extends StorageService {
  private store = new Map<string, string>();

  getItem(key: string): string | null {
    return this.store.get(key) ?? null;
  }
  setItem(key: string, value: string): void {
    this.store.set(key, value);
  }
}
```

Register the appropriate implementation in each configuration:

```typescript
// app.config.ts (client)
providers: [
  { provide: StorageService, useClass: BrowserStorageService },
]

// app.config.server.ts
providers: [
  { provide: StorageService, useClass: ServerStorageService },
]
```

Components inject `StorageService` without knowing which implementation they receive. This is the preferred approach because it keeps platform-specific code out of components entirely.

#### Checking the Platform at Runtime

When the DI approach is too heavyweight -- a single `if` check in one component, for example -- you can query the platform directly:

```typescript
import { inject, PLATFORM_ID } from '@angular/core';
import { isPlatformBrowser, isPlatformServer } from '@angular/common';

@Component({ ... })
export class ChartComponent {
  private platformId = inject(PLATFORM_ID);

  ngOnInit(): void {
    if (isPlatformBrowser(this.platformId)) {
      this.initializeChart();
    }
  }
}
```

Use this sparingly. If you find multiple `isPlatformBrowser` checks scattered across a codebase, it is a sign that the DI approach would be cleaner. In FinancialApp, the charting library used in the holdings table only initializes in the browser -- on the server, the chart area renders as a static placeholder via server-rendered HTML, and the chart library loads and initializes after hydration.

---

## Hybrid Routing

Not every route benefits from the same rendering strategy. FinancialApp's marketing landing page is static content that changes once a week -- it should be prerendered at build time. The portfolio overview is personalized and changes by the second -- it must be server-rendered on each request. The settings page is purely interactive with no SEO value -- it can be client-rendered.

Angular's `ServerRoute` configuration lets you declare the rendering mode per route:

```typescript
// app.routes.server.ts
import { RenderMode, ServerRoute } from '@angular/ssr';

export const serverRoutes: ServerRoute[] = [
  { path: '', renderMode: RenderMode.Prerender },
  { path: 'portfolio', renderMode: RenderMode.Server },
  { path: 'accounts/:id', renderMode: RenderMode.Server },
  { path: 'settings/**', renderMode: RenderMode.Client },
  { path: '**', renderMode: RenderMode.Server },
];
```

Three rendering modes are available:

- **`RenderMode.Prerender`** -- generates static HTML at build time. The HTML is served directly by the CDN with no server computation per request. Best for content that rarely changes.
- **`RenderMode.Server`** -- renders HTML on the server for each request. The response includes the user's personalized data. Best for SEO-critical pages with dynamic content.
- **`RenderMode.Client`** -- skips server rendering entirely. The server sends a minimal HTML shell and the client renders the page with JavaScript. Best for highly interactive pages behind authentication where SEO is irrelevant.

This configuration is provided alongside the server rendering setup in `app.config.server.ts` via `provideServerRoutesConfig(serverRoutes)`, as shown earlier in this chapter. The route matching follows the same rules as Angular's client-side router (see [Chapter 4](ch04-router.md)), including wildcard and parameterized paths.

### Prerendering Routes With Parameters

Static prerendering works straightforwardly for routes like `/about` or `/pricing`, but FinancialApp's account detail pages use parameterized routes: `/accounts/:id`. The build system cannot guess which account IDs exist. You must provide them explicitly using the `getPrerenderParams` callback:

```typescript
// app.routes.server.ts
export const serverRoutes: ServerRoute[] = [
  {
    path: 'accounts/:id',
    renderMode: RenderMode.Prerender,
    async getPrerenderParams() {
      const accountIds = await fetchAccountIdsForPrerender();
      return accountIds.map(id => ({ id }));
    },
  },
];
```

The `getPrerenderParams` function runs at build time. It returns an array of parameter objects, and Angular generates one HTML file per combination. For FinancialApp, this might prerender the top 100 most-accessed accounts while falling back to server rendering for the rest:

```typescript
async getPrerenderParams() {
  const topAccounts = await fetch('https://api.internal/accounts/top-100')
    .then(res => res.json());
  return topAccounts.map((a: { id: string }) => ({ id: a.id }));
}
```

Routes not covered by the prerender params fall through to the next matching `ServerRoute` entry. Structure your routes so that a `RenderMode.Server` fallback catches parameterized paths that were not prerendered:

```typescript
export const serverRoutes: ServerRoute[] = [
  {
    path: 'accounts/:id',
    renderMode: RenderMode.Prerender,
    async getPrerenderParams() { /* top 100 accounts */ },
  },
  { path: 'accounts/:id', renderMode: RenderMode.Server },
];
```

### Working with the HTTP Request and Response

Server-rendered routes often need access to the incoming HTTP request -- reading cookies for authentication, checking headers for feature flags, or setting response status codes for proper SEO behavior (returning 404 for missing accounts, 301 for redirected URLs).

Angular provides the request and response objects through dependency injection on the server:

```typescript
import { REQUEST, RESPONSE_INIT } from '@angular/ssr/tokens';

@Component({ ... })
export class AccountDetailComponent {
  private request = inject(REQUEST, { optional: true });
  private response = inject(RESPONSE_INIT, { optional: true });

  constructor() {
    if (this.request) {
      const authCookie = this.request.headers.get('cookie');
      // Parse auth state from cookies during server render
    }
  }

  setNotFound(): void {
    if (this.response) {
      this.response.status = 404;
    }
  }
}
```

Mark these injections as `optional` so the component works in both environments. On the client, `REQUEST` and `RESPONSE_INIT` are not provided, and the injection returns `null`. Guard all server-specific logic behind a null check.

For FinancialApp, the account detail page uses `REQUEST` to read the session cookie during server rendering, enabling personalized SSR without an additional authentication round-trip. If the account does not exist, it sets the response status to 404 so search crawlers do not index a broken page.

---

## Summary

Deferrable views, server-side rendering, and incremental hydration are complementary strategies that address different performance concerns. Used together, they let FinancialApp deliver fast initial renders, minimal JavaScript payloads, and full SEO coverage without sacrificing interactivity.

- **`@defer` blocks** split template sections into lazy-loaded chunks. Triggers (`on viewport`, `on interaction`, `on hover`, `on timer`, `on idle`, `when`) control when the chunk loads, and `prefetch` conditions let you overlap downloads with user intent. Placeholder, loading, and error blocks manage the loading lifecycle in the template itself.
- **SSR with `@angular/ssr`** renders HTML on the server for faster first paints and search engine visibility. Angular v21's zoneless architecture extends to the server, eliminating Zone.js from the server runtime and simplifying application stabilization.
- **Incremental hydration** combines `@defer` with `hydrate` triggers to defer the hydration of server-rendered content. The HTML is visible immediately; JavaScript attaches only when the user needs interactivity.
- **Event replay** captures user interactions that occur before hydration completes and replays them once the component is live, ensuring no user intent is lost.
- **Platform-specific implementations** -- via DI-based service swapping or `isPlatformBrowser`/`isPlatformServer` checks -- keep browser-only code out of the server render.
- **Hybrid routing** assigns rendering modes (`Prerender`, `Server`, `Client`) per route. Parameterized routes can be prerendered at build time with `getPrerenderParams`, and server routes have access to the HTTP request and response via injected tokens.

The key architectural decision is not whether to use SSR, but which rendering mode each route deserves. Start by identifying routes that need SEO or fast first-paint (server-render them), routes with stable content (prerender them), and routes that are purely interactive (client-render them). Layer `@defer` with incremental hydration on top to minimize the JavaScript cost of each page, regardless of how it was initially rendered. The routing configuration from [Chapter 1](ch01-getting-started.md) and [Chapter 4](ch04-router.md) provides the foundation; this chapter adds the rendering strategy on top.
