# Chapter 9: Performance Optimization

Performance is not a feature you bolt on before launch. It is a design constraint that shapes decisions from the first component scaffold to the final production build. A slow portfolio overview in the FinancialApp is not a cosmetic problem -- it is a trust problem. Users who wait three seconds for their account balance to appear will assume the application is broken, not busy.

Angular provides a layered set of tools for measuring and optimizing both loading performance (how quickly the application becomes interactive) and runtime performance (how smoothly it responds once running). Bundle splitting with `@defer` and lazy routes is covered at length in [Chapter 22](ch31-defer-ssr-hydration.md). This chapter focuses on measurement, profiling, and the optimizations you apply after those structural gains.

> **Companion code:** `NgOptimizedImage` usage, `enableProfiling()`, and virtual scroll optimization live in `financial-app/apps/financial-app/`.

---

## Loading Performance

The fastest code is code the browser never downloads. Loading performance optimizations reduce the bytes shipped to the client and ensure that the critical rendering path contains only what the user needs to see first.

### NgOptimizedImage

The `NgOptimizedImage` directive from `@angular/common` replaces native `<img>` tags with a version that enforces best practices automatically. It requires explicit `width` and `height` attributes (eliminating layout shift), generates responsive `srcset` attributes, and applies `loading="lazy"` by default.

For FinancialApp's portfolio view, each account card displays an avatar image. Without optimization, these images cause layout shifts as they load and block the LCP metric because the browser cannot determine their dimensions until download completes.

```typescript
// account-card.component.ts
import { Component, input } from '@angular/core';
import { NgOptimizedImage } from '@angular/common';
import { CurrencyPipe } from '@angular/common';

@Component({
  selector: 'app-account-card',
  imports: [NgOptimizedImage, CurrencyPipe],
  template: `
    <div class="account-card">
      <img
        [ngSrc]="account().avatarUrl"
        width="64"
        height="64"
        [priority]="isAboveFold()"
        [ngSrcset]="'64w, 128w, 256w'"
        sizes="64px"
        [alt]="account().name + ' avatar'" />
      <h3>{{ account().name }}</h3>
      <p class="balance">{{ account().balance | currency }}</p>
    </div>
  `,
})
export class AccountCardComponent {
  account = input.required<Account>();
  isAboveFold = input(false);
}
```

The `priority` attribute tells Angular to set `loading="eager"` and add a `<link rel="preload">` hint for that image. Use it on the image most likely to be the Largest Contentful Paint element -- typically the first visible account card. For every other image, the default lazy loading keeps them out of the critical path.

The `ngSrcset` attribute generates a responsive `srcset` so the browser downloads an appropriately sized image for the user's viewport and device pixel ratio. The `sizes` attribute tells the browser how wide the image will render, preventing it from downloading a 256-pixel image for a 64-pixel slot.

Angular will throw a development-mode warning if you use `ngSrc` without `width` and `height`, or if a priority image lacks a preconnect hint in `index.html` for its origin. These warnings are guardrails -- follow them.

### Bundle Budgets

[Chapter 1](ch01-getting-started.md) mentioned the `budgets` configuration briefly. In practice, bundle budgets are your first line of defense against accidental bloat. The `budgets` array in `angular.json` defines size thresholds for your build output:

```json
// angular.json (inside architect.build.configurations.production)
{
  "budgets": [
    {
      "type": "initial",
      "maximumWarning": "500kB",
      "maximumError": "1MB"
    },
    {
      "type": "anyComponentStyle",
      "maximumWarning": "4kB",
      "maximumError": "8kB"
    }
  ]
}
```

The `initial` budget covers the JavaScript and CSS the browser must download before the application becomes interactive. The `anyComponentStyle` budget catches individual component stylesheets that have grown unreasonably large -- often a sign that global styles have leaked into a component.

When a budget threshold is crossed, `ng build` emits a warning or fails the build entirely. This catches regressions in CI before they reach production.

To investigate what is consuming space, generate a stats file and analyze it:

```bash
ng build --stats-json
npx esbuild-visualizer --metadata dist/financial-app/stats.json --open
```

The visualizer produces a treemap showing every module and its contribution to the bundle. Common offenders include charting libraries imported at the top level (move them behind `@defer` or lazy routes), moment.js (replace with `date-fns` or native `Intl`), and barrel files that prevent tree-shaking.

---

## Runtime Performance

Once the application is loaded, performance becomes about how efficiently Angular updates the DOM in response to state changes. Signals, change detection strategy, and thoughtful template design are the primary levers.

### Zoneless Change Detection Deep Dive

Angular v21 defaults to zoneless change detection. Instead of relying on Zone.js to intercept every async operation and trigger a global change detection pass, Angular uses a notification model: the framework runs change detection only when it knows something has changed, because signals tell it so.

In zoneless mode, Angular schedules a change detection cycle when:

- A signal that is read in a template changes its value.
- An `output()` or DOM event handler executes.
- `markForCheck()` is called on a component's `ChangeDetectorRef`.
- An `AsyncPipe` receives a new value from its source observable.

If none of these things happen, Angular does nothing. There is no periodic polling, no monkey-patching of `setTimeout` or `Promise`, and no "check everything" fallback.

This has implications for how you write components. Code that worked under Zone.js because it mutated state inside a `setTimeout` or a raw `Promise.then` will not trigger a re-render in zoneless mode. Every state change must flow through a signal, or the component must explicitly call `markForCheck()`.

`OnPush` change detection was the stepping stone to this model. Under `OnPush`, Angular skipped components unless their inputs changed or `markForCheck()` was called. Zoneless extends this idea to every component by default -- there is no "Default" strategy that checks everything. If you are migrating an older codebase, converting components to `OnPush` first is a practical intermediate step.

Third-party libraries that modify state outside Angular's awareness need special attention. A charting library that updates the DOM via `requestAnimationFrame` callbacks, or a drag-and-drop library that tracks pointer position in raw event listeners, will not automatically trigger Angular change detection. Wrap their callbacks in a signal update or inject `ChangeDetectorRef` and call `markForCheck()` at the integration boundary.

### Slow Computations

Templates that call methods directly re-execute those methods on every change detection cycle. In Zone.js mode, this meant every click, every keystroke, every timer. In zoneless mode it is less frequent, but the principle still holds: expensive work in a template expression runs more often than you expect.

**Pure pipes** are Angular's built-in memoization for template expressions. A pure pipe re-evaluates only when its input value changes (by reference). For FinancialApp's transaction list, formatting a currency value with locale-specific rules is a textbook pipe use case -- `CurrencyPipe` computes once per value, not once per change detection cycle.

For custom computations, `computed()` provides the same benefit. A computed signal caches its result and recalculates only when an upstream signal changes:

```typescript
// portfolio-summary.component.ts
totalValue = computed(() =>
  this.holdings().reduce((sum, h) => sum + h.shares * h.currentPrice, 0)
);

percentageChange = computed(() => {
  const total = this.totalValue();
  const costBasis = this.costBasis();
  return costBasis === 0 ? 0 : ((total - costBasis) / costBasis) * 100;
});
```

Even if the template reads `percentageChange()` in multiple places, the reduction runs once. The computed signal knows its upstream dependencies (`totalValue` and `costBasis`) and skips re-evaluation when they have not changed.

Avoid triggering layout reflows inside lifecycle hooks or effects. Reading `offsetHeight` or calling `getBoundingClientRect()` forces the browser to synchronously calculate layout. If you then write a style property, the browser must recalculate again. Batch DOM reads before DOM writes, or defer measurements to a `requestAnimationFrame` callback.

### Track Expressions in `@for`

The `track` expression in `@for` tells Angular how to identify each item across renders. Without proper tracking, Angular has no way to know that the transaction with ID 42 in the old list is the same transaction with ID 42 in the new list -- it destroys every DOM node and recreates them from scratch.

```html
<!-- transaction-list.component.html -->
@for (tx of transactions(); track tx.id) {
  <app-transaction-row
    [transaction]="tx"
    [highlighted]="tx.id === selectedId()" />
} @empty {
  <p class="empty-state">No transactions to display.</p>
}
```

The choice of track expression matters:

**`track item.id`** is correct when each item has a stable, unique identifier. Angular matches old and new items by ID, moves existing DOM nodes if the order changes, and only creates or destroys nodes for items that were added or removed. This is the right default for any list backed by a database entity.

**`track $index`** tracks by position. This is cheaper for lists that are always fully replaced (a fresh API response replaces the entire array) and never reordered. But for reorderable lists -- drag-and-drop, sort-by-column, pagination -- `track $index` forces Angular to update every row's bindings when the order changes, because position 0 now contains a different item. For FinancialApp's transaction list, where users sort by date, amount, or category, `track tx.id` avoids unnecessary DOM churn on every sort.

### PendingTasks

When Angular server-renders a page, it needs to know when the application has "stabilized" -- when all asynchronous work is complete and the HTML is ready to serialize. In zoneless mode, there is no Zone.js to track pending macrotasks. The `PendingTasks` service fills this gap.

If a component performs async work that must complete before the server serializes the page -- loading critical data, resolving feature flags, establishing WebSocket state -- wrap it with `PendingTasks`:

```typescript
// market-status.component.ts
import { Component, inject } from '@angular/core';
import { PendingTasks } from '@angular/core';
import { MarketDataService } from '../services/market-data.service';

@Component({
  selector: 'app-market-status',
  template: `<span class="status">{{ marketStatus() }}</span>`,
})
export class MarketStatusComponent {
  private taskService = inject(PendingTasks);
  private marketData = inject(MarketDataService);
  marketStatus = signal('Loading...');

  constructor() {
    this.taskService.run(async () => {
      const status = await this.marketData.getCurrentStatus();
      this.marketStatus.set(status);
    });
  }
}
```

The `taskService.run()` method registers the async operation with Angular's stabilization tracking. The server waits for all pending tasks to complete before serializing the HTML. Without this, the server might serialize `Loading...` instead of the actual market status.

On the client side, `PendingTasks` is a no-op by default. The same component code works in both environments without platform checks.

### withHttpTransferCache

When Angular server-renders a page, the server makes HTTP requests to populate the initial state. Without intervention, the client makes the exact same requests again after hydration -- the user sees correct content flash to a loading state and back, and the API absorbs double the load.

`withHttpTransferCache` solves this by serializing the server's HTTP responses into the HTML transfer state. When the client hydrates, `HttpClient` checks the transfer cache before making a network request. If the response is already cached, it returns it immediately and skips the duplicate call.

```typescript
// app.config.ts
import { provideHttpClient, withInterceptors } from '@angular/common/http';
import { withHttpTransferCache } from '@angular/platform-browser';

export const appConfig: ApplicationConfig = {
  providers: [
    provideHttpClient(
      withInterceptors([authTokenInterceptor]),
      withHttpTransferCache(),
    ),
    // ...
  ],
};
```

The transfer cache works automatically with `PendingTasks`: requests made during server rendering are tracked as pending tasks, and the server waits for them to complete before serializing both the HTML and the cached responses. On the client, the cached responses are consumed once and then discarded -- subsequent navigation or refetches hit the network as usual.

This is particularly important for FinancialApp's server-rendered portfolio page. The server fetches account data, holdings, and recent transactions. Without the transfer cache, the client would refetch all three on hydration, causing a visible flash and three unnecessary API calls.

### Reactive Forms in Zoneless Mode

Reactive forms predate signals, and their update mechanism does not participate in Angular's signal-based notification model. Calling `setValue()` or `patchValue()` on a `FormControl` updates the form's internal state, but it does not notify Angular's change detection scheduler in zoneless mode.

The symptom is subtle: a form control updates its value, but the template does not reflect the change until something else triggers change detection -- a click, a signal write, or an unrelated event.

Two strategies address this. The first injects `ChangeDetectorRef` and calls `markForCheck()` after programmatic updates:

```typescript
// transfer-form.component.ts
import { Component, inject, signal } from '@angular/core';
import { ChangeDetectorRef } from '@angular/core';
import { FormBuilder, ReactiveFormsModule } from '@angular/forms';

@Component({
  selector: 'app-transfer-form',
  imports: [ReactiveFormsModule],
  template: `
    <form [formGroup]="form">
      <input formControlName="amount" type="number" />
      <input formControlName="recipient" />
      <button (click)="prefillLastTransfer()">Use Last Transfer</button>
    </form>
  `,
})
export class TransferFormComponent {
  private fb = inject(FormBuilder);
  private cdr = inject(ChangeDetectorRef);

  form = this.fb.group({
    amount: [0],
    recipient: [''],
  });

  prefillLastTransfer(): void {
    this.form.patchValue({ amount: 500, recipient: 'Savings Account' });
    this.cdr.markForCheck();
  }
}
```

The second strategy reflects form state through a signal, keeping the template bound to a reactive primitive that Angular already knows how to track:

```typescript
// transfer-form.component.ts
export class TransferFormComponent {
  private fb = inject(FormBuilder);

  form = this.fb.group({
    amount: [0],
    recipient: [''],
  });

  formValues = toSignal(this.form.valueChanges, {
    initialValue: this.form.getRawValue(),
  });
}
```

With `toSignal`, every form change -- whether triggered by user input or programmatic `patchValue` -- flows through the signal graph and triggers change detection naturally. The template binds to `formValues()` instead of reading form controls directly.

### Virtual Scrolling

When FinancialApp displays a full year of transactions, the list can exceed a thousand rows. Rendering all of them creates thousands of DOM nodes that consume memory, slow initial render, and make scrolling janky on low-end devices.

The CDK's virtual scrolling module renders only the items visible in the viewport, plus a small buffer. As the user scrolls, Angular recycles DOM nodes -- removing rows that scroll out of view and creating rows that scroll in.

```typescript
// holdings-table.component.ts
import { Component, input } from '@angular/core';
import { ScrollingModule } from '@angular/cdk/scrolling';
import { CurrencyPipe, DecimalPipe } from '@angular/common';
import { Holding } from '../../models/holding.model';

@Component({
  selector: 'app-holdings-table',
  imports: [ScrollingModule, CurrencyPipe, DecimalPipe],
  template: `
    <cdk-virtual-scroll-viewport itemSize="48" class="holdings-viewport">
      <table>
        <thead>
          <tr>
            <th>Symbol</th>
            <th>Shares</th>
            <th>Price</th>
            <th>Value</th>
          </tr>
        </thead>
        <tbody>
          <tr *cdkVirtualFor="let holding of holdings(); trackBy: trackById">
            <td>{{ holding.symbol }}</td>
            <td>{{ holding.shares | number:'1.2-2' }}</td>
            <td>{{ holding.currentPrice | currency }}</td>
            <td>{{ holding.shares * holding.currentPrice | currency }}</td>
          </tr>
        </tbody>
      </table>
    </cdk-virtual-scroll-viewport>
  `,
  styles: `
    .holdings-viewport {
      height: 600px;
      overflow-y: auto;
    }
  `,
})
export class HoldingsTableComponent {
  holdings = input.required<Holding[]>();

  trackById(_index: number, holding: Holding): string {
    return holding.id;
  }
}
```

The `itemSize` attribute tells the virtual scroller the height of each row in pixels. This must be consistent -- variable-height rows require `autosize` mode from `@angular/cdk-experimental`. The viewport element needs an explicit height so the scroller knows how many items fit on screen.

For FinancialApp's holdings table with 1,200 rows, virtual scrolling reduces the DOM node count from roughly 6,000 to around 80 at any given moment. The difference is noticeable on every device, but dramatic on mobile.

---

## Measuring and Profiling

Optimization without measurement is guesswork. Angular provides profiling tools that integrate with the browser's performance infrastructure, and the Core Web Vitals framework gives you metrics that correlate with real user experience.

### Core Web Vitals

Google's Core Web Vitals are three metrics that capture the user's perception of loading, interactivity, and visual stability:

**Largest Contentful Paint (LCP)** measures how long it takes for the largest visible element to render. For FinancialApp, this is typically the account balance or the portfolio summary card. `NgOptimizedImage` with the `priority` attribute directly improves LCP by preloading the hero image. SSR ([Chapter 22](ch31-defer-ssr-hydration.md)) improves LCP by sending rendered HTML instead of an empty shell.

**Cumulative Layout Shift (CLS)** measures visual stability -- how much the page layout jumps as content loads. `NgOptimizedImage` prevents image-related layout shift by requiring explicit dimensions. `@defer` blocks with properly sized `@placeholder` containers prevent shift when deferred content loads. Skeleton screens in placeholders are not cosmetic -- they are CLS prevention.

**Interaction to Next Paint (INP)** measures responsiveness -- the time from a user interaction (click, tap, keypress) to the next visual update. This is where Angular's signal-based reactivity pays off. A signal update triggers targeted change detection only in affected components, rather than a full tree check. Long-running synchronous computations in event handlers are the primary INP killer -- move them to `computed()` signals, Web Workers, or break them into smaller chunks with `requestAnimationFrame`.

You can measure Core Web Vitals in the field with the `web-vitals` library, or in the lab with Chrome DevTools Lighthouse. Field data from real users is more valuable because lab conditions rarely match the diversity of devices and networks your users actually have.

### Chrome DevTools Angular Profiler

Angular provides a built-in profiling integration with Chrome DevTools' Performance panel. To enable it, call `enableProfiling()` during bootstrap:

```typescript
// main.ts
import { bootstrapApplication } from '@angular/platform-browser';
import { enableProfiling } from '@angular/core';
import { AppComponent } from './app/app.component';
import { appConfig } from './app/app.config';

enableProfiling();
bootstrapApplication(AppComponent, appConfig);
```

With profiling enabled, open Chrome DevTools, switch to the **Performance** tab, and record a session while interacting with the application. The recorded flame chart includes an Angular-specific track with color-coded entries:

- **Blue** entries represent application code -- your component methods, service calls, and signal computations.
- **Purple** entries represent template rendering -- Angular evaluating bindings and updating the DOM.
- **Green** entries represent framework entry points -- change detection cycles, event handler dispatch, and initialization.

When reading a flame chart, look for these patterns:

**Bootstrap cost.** A wide green block at the start of the recording shows the time spent initializing the application. If this is large, check whether application initializers are doing too much synchronous work, or whether eagerly loaded modules are pulling in code that could be deferred.

**Component execution time.** Blue blocks during a change detection cycle show which components are expensive to evaluate. A single component that dominates the frame indicates a computation that should be memoized with `computed()` or moved to a pure pipe.

**Change detection cycles.** Multiple green entry points firing in rapid succession -- several synchronization passes within a single user interaction -- are a code smell. This often indicates that an `effect()` is writing to a signal that another effect depends on, creating a cascade of change detection rounds. Restructure effects to avoid signal writes, or consolidate the cascading logic into a single `computed()`.

### Angular DevTools Extension

The Angular DevTools browser extension (available for Chrome and Edge) provides two primary features for performance work:

**Component tree inspector.** The Components tab displays the live component hierarchy with signal values, input bindings, and dependency injection details. Use it to verify that `OnPush` or zoneless components are not being checked unnecessarily -- the inspector highlights components as they are checked during a change detection cycle.

**Change detection profiler.** The Profiler tab records change detection cycles and displays a bar chart showing how much time each component consumed. This is less granular than the Chrome DevTools flame chart but more accessible for quick diagnostics. Look for components that appear in every cycle despite not having changed -- they are missing signal-based inputs, are not using `OnPush`, or are reading mutable state that Angular cannot track.

The profiler also shows the number of change detection cycles triggered during the recording. In a well-optimized zoneless application, interacting with a single form field should trigger one or two cycles, not a dozen. If you see excessive cycles, trace the cause: an effect writing to a signal, a `setInterval` calling `markForCheck()`, or a third-party library triggering events outside Angular's awareness.

---

## Summary

Performance optimization in Angular is not a single technique but a discipline applied at every layer of the application:

- **`NgOptimizedImage`** enforces image best practices -- explicit dimensions, responsive `srcset`, `priority` preloading for LCP images -- through a single directive.
- **Bundle budgets** catch size regressions in CI. Use `--stats-json` with a bundle visualizer to diagnose what is consuming space. For the splitting itself -- `@defer` blocks and lazy routes -- see [Chapter 22](ch31-defer-ssr-hydration.md).
- **Zoneless change detection** eliminates the Zone.js overhead and enforces a signal-driven notification model. Components update only when their signal dependencies change, an event handler runs, or `markForCheck()` is called explicitly.
- **`computed()` and pure pipes** memoize expensive template expressions, preventing redundant recalculation across change detection cycles.
- **`track` expressions** in `@for` preserve DOM identity across renders. Use `track item.id` for entity lists; reserve `track $index` for static, non-reorderable collections.
- **`PendingTasks`** ensures async work completes before SSR serialization, preventing the server from shipping incomplete HTML.
- **`withHttpTransferCache()`** serializes server-side HTTP responses into the page, eliminating duplicate requests during client hydration.
- **Reactive forms in zoneless mode** require explicit `markForCheck()` calls or signal bridging via `toSignal(form.valueChanges)` to participate in change detection.
- **Virtual scrolling** with the CDK reduces DOM node count for large lists from thousands to dozens, transforming the performance profile of data-heavy views.
- **Core Web Vitals** (LCP, CLS, INP) provide user-centric metrics that map directly to Angular features: `NgOptimizedImage` and SSR for LCP, `@defer` placeholders for CLS, signals for INP.
- **`enableProfiling()`** adds Angular-specific instrumentation to Chrome DevTools flame charts. Color-coded tracks reveal bootstrap cost, component execution time, and excessive change detection cycles.
- **Angular DevTools** provides a component tree inspector and change detection profiler for quick diagnostics without the complexity of a full flame chart recording.

Measure first, then optimize. Profile in conditions that approximate real user environments -- throttled CPU, slow 3G, mid-range Android devices. The tools in this chapter give you visibility into where time is spent. The techniques give you the means to eliminate waste. Neither is useful without the other.
