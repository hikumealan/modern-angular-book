# Chapter 32: Debugging Angular Applications

Debugging is a feedback loop. You form a hypothesis about why the portfolio overview is rendering a stale balance, you run an experiment, you observe the result, and you refine the hypothesis. The tighter that loop -- the less time between "I think I know what is wrong" and "I have proof" -- the faster the fix. A ten-second reload cycle with a `console.log` at the right line beats a thirty-minute odyssey through a flame chart every time, provided you know which line.

Angular ships with enough debugging hooks that most production problems are tractable once you know where to look. The framework exposes its component tree, its injector graph, its change-detection cycles, and its signal dependencies to external tooling; the browser exposes the rest. This chapter walks through the places a bug in FinancialApp typically hides, and the instruments best suited to flushing each one out.

> **Companion code:** No new library -- this chapter uses existing FinancialApp code for demonstration scenarios.

---

## Angular DevTools

The Angular DevTools browser extension (available for Chrome and Edge) is the single most productive diagnostic tool in the Angular ecosystem. [Chapter 9](ch32-performance.md) introduced the Profiler tab for performance work; the extension does more than that. Install it from the Chrome Web Store, open DevTools, and a new **Angular** panel appears next to Elements and Console.

### Component Explorer

The Components tab mirrors the running application's component tree. Selecting a node shows the live values of its `input()`, `output()`, `model()`, and signal properties in the right-hand inspector. Expanding a signal reveals its current value; changing it from the inspector updates the live component, which is an underrated way to reproduce edge-case state without writing UI to get there.

For FinancialApp's portfolio overview, opening `<app-portfolio-summary>` in the explorer reveals its `totalValue` and `percentageChange` computed signals with their current derived values. If `percentageChange()` is `NaN`, the inspector makes that visible immediately -- no need to add a template binding. Clicking on a signal shows which other signals it depends on, so you can walk the graph toward the corrupted input without searching the source.

The explorer also respects the DOM selection from the Elements panel. Select a `.holdings-row` in Elements and the Components tab highlights the matching `<app-holdings-row>` instance. The reverse -- "inspect the DOM of this component" -- is available from the explorer's context menu. Pairing this with the DOM's `.scrollIntoView()` and a quick highlight color change from the inspector is often enough to physically locate an off-screen row that a user is complaining about.

For components that render inside a `@for` block, the explorer lists every instance separately, each with its own `$index` and `$count` context values. A bug that only reproduces on the 47th transaction row is exactly the kind of thing the explorer surfaces quickly -- click through rows until the divergent state appears, then copy the underlying data out of the inspector.

### Profiler

The Profiler tab records change detection cycles. Start recording, interact with the application, stop recording. The result is a flame chart where each frame is one change detection cycle, and each bar within a frame is the time spent checking a specific component.

Two patterns jump out of a profiler trace. A component that appears in every cycle despite not having changed is missing a signal-based input, reading mutable state Angular cannot track, or doing work in a template expression that should be a `computed()`. A burst of cycles firing in rapid succession on a single user interaction usually means an `effect()` is writing a signal that another `effect()` depends on, triggering a cascade. Both are structural fixes, not performance knob-twiddling.

### Router Tree

Routing bugs are particularly painful because the URL, the activated route, and the rendered component can disagree in three different ways. The Router Tree tab shows the full active route hierarchy: which route matched, what parameters were extracted, which `data` resolvers ran, and which child routes are currently activated. If the URL is `/accounts/42/transactions?category=food` but the tree shows `accountId` as `"42"` (string) when your component expects `number`, the mismatch is visible immediately.

The tree updates in real time as the user navigates, so you can watch a guard redirect happen or spot an accidental double-activation caused by a parent route re-emitting on a child parameter change.

### Dependency Injection Tree

Misconfigured providers are among the most opaque bugs in Angular. A service behaves differently depending on whether it was provided at the root, a route, a lazy-loaded module, or the component itself -- and a stale `NullInjectorError` stack trace rarely tells you where the lookup stopped.

The Injector Tree tab visualizes the injector graph for the currently selected component. Each injector in the chain is listed with its providers, and the path from a component to any injected dependency is highlighted. If FinancialApp's `TransactionEditorComponent` is resolving a different `AuthService` instance than `PortfolioComponent`, the tree shows exactly which injector supplied each one -- usually revealing a duplicate `providers` array on a lazy route.

---

## `ngDevMode` and Runtime Assertions

Angular maintains a global `ngDevMode` flag that is `true` in development builds and `false` in production. The framework uses this flag to gate expensive runtime checks: circular dependency detection, missing provider diagnostics, duplicate directive detection on a single element, template binding validation, and the error messages that come with them. These assertions are the reason a development build surfaces "NG0200: Circular dependency in DI detected for `AccountService`" with a full provider chain, while a production build would throw a truncated error.

The tree-shaking story is equally important. Every branch guarded by `if (ngDevMode)` is eliminated from the production bundle, along with the error message strings. A production build ships approximately 15-20% less JavaScript than a development build of the same application precisely because so much of the framework is diagnostic scaffolding. This is why you never want to rely on a development-mode warning as a production guardrail -- it will not be there.

For catching change detection issues earlier than the default, Angular v21 offers `provideCheckNoChangesConfig`:

```typescript
// app.config.ts
import { provideCheckNoChangesConfig } from '@angular/core';

export const appConfig: ApplicationConfig = {
  providers: [
    provideCheckNoChangesConfig({
      exhaustive: true,
      interval: 5000,
    }),
  ],
};
```

The `exhaustive` option tells Angular to re-check every view (not just `OnPush`/zoneless views that were marked dirty) after every change detection cycle, looking for expressions that changed between the check and the re-check. This is exactly what produces the infamous `ExpressionChangedAfterItHasBeenCheckedError`. The `interval` option schedules a periodic background check, so you catch issues that only surface under unusual timing. Enable this in development and preview environments, never in production.

---

## Debugging Template Errors

Template errors are the most common bugs in Angular and the easiest to fix once you know the vocabulary. Angular assigns a stable error code to every framework-level error, and the codes map to documentation pages at `https://angular.dev/errors/NG0100`, `NG0200`, and so on. The error message in the console always contains the code -- open the URL for the canonical explanation and a list of common causes.

A few that account for most of FinancialApp's template-related bug reports:

**"Cannot read properties of undefined (reading 'balance')"** is not an Angular error -- it is plain JavaScript leaking through a template. The usual cause is binding to a property before an async resource has resolved. The fix is to gate the binding:

```html
@if (accountResource.hasValue()) {
  <p>{{ accountResource.value()!.balance | currency }}</p>
} @else {
  <p>Loading...</p>
}
```

**NG0100: `ExpressionChangedAfterItHasBeenCheckedError`** means a binding produced a different value during the second check of the same cycle. The classic cause is a parent component reading a child's output during `ngAfterViewInit` and updating a signal that the parent also binds to. The modern fix is almost always to replace the imperative update with a `computed()` signal, which evaluates lazily and cannot produce mid-cycle drift.

**NG0200: Circular dependency in DI detected.** Two services inject each other directly, or a chain of injections loops back on itself. The error message lists the chain. The fix is either to extract the shared logic into a third service, to break the cycle with `Injector` and a lazy `inject()` inside a method, or -- most often -- to reconsider whether both services should exist at all.

When none of the above helps, the most productive technique is a logging pipe. Create a trivial pipe that returns its input unchanged and calls `console.log` as a side effect:

```typescript
// libs/shared/util-debug/src/lib/log.pipe.ts
@Pipe({ name: 'log', standalone: true, pure: false })
export class LogPipe implements PipeTransform {
  transform<T>(value: T, label = 'log'): T {
    if (ngDevMode) console.log(`[${label}]`, value);
    return value;
  }
}
```

Used in a template as `{{ account().balance | log:'balance' }}`, it renders the original value and logs every evaluation to the console. The `pure: false` flag ensures it runs on every change detection cycle, which is exactly what you want for diagnosis (and exactly what you want to remove before committing). For breakpoint-based debugging of template initialization, `ngAfterViewInit` is the right hook -- the view has rendered once and all child component references are populated.

---

## Debugging Hydration Errors

Hydration ([Chapter 22](ch31-defer-ssr-hydration.md)) is brittle by nature: the server renders HTML, the client renders the same component tree, and if the two do not agree on structure, Angular throws an `NG0500`-series error. The error message lists the mismatched node, the expected structure, and the actual DOM at the point of divergence.

Common causes in FinancialApp:

- **Conditional rendering based on `window` or `document`.** A component that renders one tree on the server (where `window` is undefined) and a different tree on the client will fail to hydrate. Use the DI-based platform abstraction from [Chapter 22](ch31-defer-ssr-hydration.md) instead of direct `window` checks, so both environments render the same structure.
- **`Date.now()` or `Math.random()` in a template.** The server sees one value, the client sees another, and any DOM downstream of the difference is flagged. Compute such values once in the component constructor and store them in a signal.
- **Third-party scripts that modify the DOM before hydration.** A marketing script that inserts a tracking pixel between the server render and the client boot will shift indexes. Load such scripts after hydration completes, or wrap the affected subtree in `ngSkipHydration`.

Angular ships a development-mode hydration overlay that highlights the mismatched region directly in the page. Enable it via the `provideClientHydration` options when running a preview build:

```typescript
// app.config.ts (preview configuration)
import { provideClientHydration, withIncrementalHydration } from '@angular/platform-browser';

export const appConfig: ApplicationConfig = {
  providers: [
    provideClientHydration(
      withIncrementalHydration(),
      // withDebugTools() emits overlays + console diagnostics in dev
    ),
  ],
};
```

The overlay draws a red outline around the divergent node and prints the mismatch details in the console: the expected structure (from the server render), the actual structure (from the client render), and the binding or component that diverged. For FinancialApp, the most common offender is the `currentTimestamp` display in the footer -- the server renders `10:32:15 AM` and the client renders `10:32:17 AM` two seconds later, producing a text-node mismatch. Fixing it means stabilizing the value at a single point in the component lifecycle (usually a `signal` initialized once in the constructor with `toSignal(interval(1000))` added only after hydration).

The escape hatch, when all else fails, is the `ngSkipHydration` attribute:

```html
<div ngSkipHydration>
  <third-party-chart [data]="data()" />
</div>
```

This tells Angular to skip hydration for that subtree and re-render it on the client from scratch. It is a pragmatic tool, not a scalable solution -- treat each `ngSkipHydration` as a TODO to fix the underlying structural mismatch, not a permanent fix. FinancialApp uses it on exactly one element: the container for a legacy charting library that has not yet been migrated to the SSR-aware ApexCharts wrapper.

---

## Debugging Zoneless Change Detection

Zoneless change detection ([Chapter 9](ch32-performance.md)) eliminates a whole class of bugs -- and introduces a smaller, more specific class in exchange. The most common symptom is "the template does not update when I change this signal." There are two usual culprits.

**Missing signal reads.** Angular's view invalidation is based on which signals the template reads during rendering. If the template calls a method that internally reads a signal, the component subscribes to it; if the template reads a property that happens to return a signal's value without calling the signal, it does not.

```typescript
@Component({
  selector: 'app-account-badge',
  template: `
    <!-- WRONG: formatBalance() reads balance internally, but the
         template's dependency tracking only sees 'this'. If balance
         changes, the view does not re-render. -->
    <span>{{ formatBalance() }}</span>

    <!-- RIGHT: the template directly reads balance(), establishing
         a dependency. The method can still exist for computation. -->
    <span>{{ formatBalance(balance()) }}</span>
  `,
})
export class AccountBadgeComponent {
  balance = signal(0);

  formatBalance(value = this.balance()): string {
    return new Intl.NumberFormat('en-US', {
      style: 'currency', currency: 'USD',
    }).format(value);
  }
}
```

The first form works in a zoned application because Zone.js triggers change detection on every microtask. The second form works in both zoned and zoneless modes because the template reads the signal directly. Migrating an application to zoneless surfaces every instance of the first pattern at once -- usually as "nothing updates anymore." The fix is almost always to pull the signal read into the template or to convert the method into a `computed()`.

A shallow version of this bug: a computed signal that depends on an untracked read via `untracked()` will not re-evaluate when the untracked signal changes. If you accidentally wrap a dependency in `untracked()`, the view goes stale without warning.

**Stale views when `markForCheck` is forgotten.** Non-signal state updates (a mutation on a plain property, a `BehaviorSubject.next()` without `toSignal()`, a callback from a non-Angular library) do not notify the change detection scheduler. The fix is to either convert the state to a signal or inject `ChangeDetectorRef` and call `markForCheck()` explicitly at the boundary. The latter is the pragmatic approach for integrating with legacy code.

For diagnosing excessive change detection work, the browser console exposes `ng.profiler.timeChangeDetection()`:

```javascript
// In the browser console, after the app has loaded
ng.profiler.timeChangeDetection({ record: true });
```

This runs change detection in a tight loop for five hundred iterations and prints the average time per cycle plus a flame chart. A healthy FinancialApp page runs a cycle in under 1ms; if yours takes 10ms, something in the bindings is doing too much work per check.

---

## Signal Graph Debugging

The reactive flow from [Chapter 3](ch03-reactive-signals.md) -- signals feeding computeds feeding resources feeding templates -- is ideal in the steady state and subtle when something goes wrong. A derived signal produces a stale value and you need to know why: did the upstream signal not fire, did the derivation skip because of a broken equality check, or did the consumer not read the signal at all?

Angular DevTools visualizes the graph for a selected component in the Signal Graph view (available in recent versions of the extension). Nodes are signals, edges are dependencies, and the current value of each node is displayed inline. Invalidating an upstream node by editing it from the inspector shows which downstream nodes re-evaluate and which do not -- usually making the culprit obvious.

For ad-hoc debugging without DevTools, `effect()` is the right tool:

```typescript
import { effect } from '@angular/core';

// In the component constructor
effect(() => {
  console.log('[debug] filters changed to', this.filters());
});
```

This logs every change to `filters` without modifying the business logic. Remove the effect before committing -- or better, wrap it in an `if (ngDevMode)` so it is tree-shaken from production builds automatically.

A subtler class of bug: a **signal glitch**. Two synchronous writes to different signals that feed a common computed should produce a single downstream evaluation; if you see two in the profiler, a dependency is not what you think it is. The most common cause is an `effect()` that writes to a signal in response to another signal change, producing a cascade that runs until the graph stabilizes. Reserving effects for true side effects (logging, DOM synchronization, storage) and using `computed()` for derivation eliminates this category entirely.

---

## Source Maps in Production

A production stack trace without source maps is a list of minified identifiers: `qe at bundle.a8c01f.js:42:18931`. To symbolicate a trace back to `AccountService.getBalance at account.service.ts:47`, your error tracker (Sentry, Datadog, Rollbar, Bugsnag, any of them) needs the source map that was generated alongside the bundle. Without it, every production exception is effectively anonymous.

The Angular CLI defaults to disabling source maps in production builds for payload reasons. Re-enable them explicitly:

```json
// angular.json
{
  "configurations": {
    "production": {
      "sourceMap": {
        "scripts": true,
        "styles": true,
        "hidden": true,
        "vendor": false
      }
    }
  }
}
```

The `hidden: true` option is the key: it generates the `.map` files but omits the `//# sourceMappingURL=` comment from the bundle, so browsers do not download them on every page load but your error tracker can still consume them. The `vendor: false` option skips maps for third-party libraries, which are usually published with their own maps already.

The production pipeline then uploads the maps to the error tracker and -- critically -- deletes them from the deployed artifact. Keeping source maps on the CDN alongside the bundle means anyone can download them and un-minify your production code. The upload-and-delete dance is a one-liner in most CI systems:

```bash
npx @sentry/cli sourcemaps upload ./dist/financial-app/browser
find ./dist/financial-app/browser -name "*.map" -delete
```

The trade-off is a slightly larger build artifact during the upload window and a bit of CI time. The payoff is that every production error lands in your tracker with a fully resolved stack trace, a readable line of code, and the surrounding context needed to diagnose it without a local repro.

---

## Browser DevTools for Angular

Chrome and Edge DevTools are still the primary instrument for client-side debugging. A few features earn their keep in an Angular codebase.

**Conditional breakpoints** pause execution only when a predicate is true. Right-click a line number in the Sources panel, select "Add conditional breakpoint", and enter an expression. For FinancialApp, a conditional breakpoint inside the transaction posting logic might read `account.balance < 0` -- the debugger only pauses when an overdraft would occur, skipping the thousands of routine transactions.

**Logpoints** are a conditional breakpoint's quieter cousin: they log a value to the console without pausing execution. Right-click a line, select "Add logpoint", and enter a message like `"balance before fee:", account.balance, "for", account.name`. The effect is identical to adding a `console.log` and reloading, but without modifying the source file or triggering a rebuild. On a production build with source maps, this lets you instrument live code without a deploy.

**Inspecting component instances from the console** is the Angular equivalent of jQuery's `$0`. With an element selected in the Elements panel (so `$0` refers to it), the Angular dev utilities expose:

```javascript
ng.getComponent($0)      // the component instance for the selected element
ng.getContext($0)        // the template context (for @for/@if blocks)
ng.getDirectives($0)     // all directives on the element
ng.getHostElement(cmp)   // the DOM element for a component instance
ng.getOwningComponent($0) // the nearest ancestor component
ng.applyChanges(cmp)     // force change detection on a component
```

Combine with DevTools' automatic `$0`, `$1`, `$2`, `$3`, `$4` variables (which always refer to the last five elements you selected, in order) to inspect several components at once without writing a line of code:

```javascript
// Select the transaction row in Elements tab, then:
const row = ng.getComponent($0);
console.table(row.transaction());
row.highlighted.set(true);  // temporarily highlight to confirm identity

// Select the parent list, then:
const list = ng.getComponent($0);
list.sortMode.set('amount');  // re-sort live without clicking UI
ng.applyChanges(list);
```

This is how most "it works on my machine but not on yours" debugging sessions end: five minutes in the DevTools console poking at live component state until the discrepancy surfaces. The pattern works particularly well for reproducing "the user sees a bug after thirty minutes of use" scenarios -- instead of waiting thirty minutes, you drive the component directly into the state that usually takes thirty minutes to reach naturally.

---

## Debugging HTTP Issues

When an HTTP call misbehaves, the first stop is always the browser's Network tab. It shows the full request and response -- URL, headers, body, status code, timing breakdown -- without any code changes. Filter by "Fetch/XHR" to strip out asset requests, and click a request to see the full payload on the Headers and Response tabs. The interceptor chain registered in [Chapter 48](ch13-initialization-routes.md) runs before the request appears in the Network tab, so everything visible there is what the server actually received -- no ambiguity about whether a header was attached.

For FinancialApp's transaction submission, the Network tab reveals the distinction between several failure modes that look similar to the user. A 401 means the auth interceptor failed to attach a token. A 422 means the server rejected the payload's validation. A 409 means the server detected a duplicate transaction and refused to apply it twice. A network error with no response at all means the request never left the client, probably because an interceptor threw or a CORS preflight failed. A pending request that never resolves (visible with the "Waiting" bar stretched to the edge of the timeline) means the server never responded -- the problem is downstream of Angular.

The **Timing** tab on a selected request breaks down the lifecycle: queueing, connection setup, request sent, server response, content download. A request that spends 400ms in "Queueing" means the browser is holding it back because too many requests are open to the same origin -- the classic "six connections per host" limit. Turning HTTP/2 on at the CDN fixes this immediately.

When the issue is in an interceptor itself, a temporary logging interceptor is the cleanest way to see what is happening. Add it early in the chain so it sees the raw outgoing request:

```typescript
// libs/shared/util-http/src/lib/debug.interceptor.ts
export const debugInterceptor: HttpInterceptorFn = (req, next) => {
  if (!ngDevMode) return next(req);

  const start = performance.now();
  console.groupCollapsed('[http]', req.method, req.url);
  console.log('request', req);
  console.groupEnd();

  return next(req).pipe(
    tap({
      next: (event) => {
        if (event.type === HttpEventType.Response) {
          console.log(
            '[http] response',
            req.url,
            event.status,
            `${(performance.now() - start).toFixed(0)}ms`,
          );
        }
      },
      error: (err) => console.error('[http] error', req.url, err),
    }),
  );
};
```

Register it first in `withInterceptors` so it captures the request before any transformation and the response after all transformations. The `ngDevMode` guard ensures the interceptor is tree-shaken out of production builds -- you can leave the registration in place permanently, and it costs nothing at runtime. For interceptors that need to observe requests in preview environments as well, register it behind an environment flag instead of `ngDevMode`.

For handing a reproducible failure to a backend team, the Network tab's "Save all as HAR with content" option exports the full request/response log as a single JSON file. The HAR file includes headers, bodies, timings, and cookie state, and it can be replayed in tools like HAR Viewer or Insomnia. A HAR capture attached to a bug report removes an entire round-trip of "what exactly did the client send?" questions.

---

## Debugging RxJS

RxJS ([Chapter 3](ch03-reactive-signals.md)) streams are inherently harder to debug than signals because values flow through a pipeline rather than being stored anywhere you can inspect. The two techniques that work on every codebase:

**Tagging with `tap`.** Insert `tap(value => console.log('after switchMap:', value))` at every operator boundary you want to observe. The stream continues unchanged; the side effect is your logging. For a breakpoint rather than a log, use `tap(() => { debugger; })` -- execution pauses in DevTools at that exact point in the pipeline.

```typescript
results$ = toObservable(this.query).pipe(
  tap((q) => console.log('[search] raw', q)),
  debounceTime(250),
  distinctUntilChanged(),
  tap((q) => console.log('[search] after debounce/distinct', q)),
  switchMap((q) => this.http.get<Account[]>(`/api/accounts?q=${q}`)),
  tap((r) => console.log('[search] response', r)),
);
```

Five lines of instrumentation show which operator is dropping, duplicating, or rearranging emissions. Remove the `tap`s before committing or conditionalize them on `ngDevMode`.

**`rxjs-spy`** is a developer-tools library that monkey-patches RxJS globally and exposes every subscription, emission, and unsubscription through a browser console API. Install it in development only (`npm install --save-dev rxjs-spy`), call `spy({ defaultPlugins: true })` in `main.ts` behind an `if (ngDevMode)`, and pipe streams through `.pipe(tag('search'))` at any boundary you care about. In the console, `rxSpy.show()` prints the current state of every tagged stream, and `rxSpy.log('search')` prints every event for that tag. It is the closest thing to a time-travel debugger for RxJS.

---

## A Debugging Playbook

Tools matter, but process matters more. The debugging sequence that moves fastest, from first report to pull request, is roughly:

**Reproduce locally.** An unreproducible bug is a rumor. If the reported issue only happens in production, the first job is to build an environment where you can see it: a preview deployment against the same backend, a staging database with anonymized production data, or a Docker Compose stack with the exact service versions from prod. Preview environments from [Chapter 34](ch41-ci-cd.md) exist for exactly this -- the PR that tries the fix should spin up its own environment where the fix can be verified against the original reproduction steps.

**Narrow the scope.** Once you can reproduce it, strip everything away that is not involved. A 5,000-line FinancialApp repro that touches auth, routing, and twelve services is useless for escalation. A 40-line StackBlitz with a single component, a mock HTTP call, and the failing binding is a bug report anyone on the Angular team can reason about. Every step you remove from the reproduction removes a possible cause.

**Binary-search the history.** When the bug was introduced is often easier to identify than why. `git bisect` automates this:

```bash
git bisect start
git bisect bad HEAD
git bisect good v2024.10.01  # last known-good release
# Git checks out a commit halfway between
# You test: npm ci && npm run test:e2e -- --grep "account overdraft"
git bisect bad  # or good, depending on the result
# Repeat until git identifies the offending commit
```

The `--first-parent` flag restricts bisect to merge commits, which in a squash-merge workflow gives you one commit per PR rather than every intermediate. Most first-time users of `git bisect` find the offending commit in seven or eight steps on a year's worth of history -- it is the most productive ninety minutes of debugging you will have in a quarter.

If the reproduction can be automated, `git bisect run` fully automates the search:

```bash
git bisect start HEAD v2024.10.01
git bisect run npm run test:e2e -- --grep "account overdraft"
```

Git checks out commits, runs the test, and reports the first bad commit -- no human interaction. For a flaky reproduction that fails intermittently, wrap the test in a retry loop or use `git bisect skip` for commits where the test is inconclusive.

**Escalate with a minimal reproduction.** If the fix turns out to belong in Angular, RxJS, or a third-party library rather than FinancialApp, the bug report that gets attention is the one the maintainer can run in under a minute. Link to a StackBlitz with reproduction steps, include the output of `ng version`, and quote the exact error message with its error code. A bug that says "NG0100 at line 47, here is a 40-line repro" closes faster than a bug that says "the app is broken, here is a screenshot."

Tests from [Chapter 7](ch07-testing-vitest.md) close the loop. A reproduction is also a failing test; once you have the fix, convert the repro into a regression test, run it red, apply the fix, watch it go green. The same reproduction that found the bug prevents it from coming back.

---

## Summary

Debugging Angular is not one skill but a layered toolkit, and the fastest developers know which tool matches which symptom:

- **Angular DevTools** is the first stop for anything involving component state, change detection, routing, or DI. The Component Explorer, Profiler, Router Tree, and Injector Tree cover four of the five most common bug categories in the framework.
- **`ngDevMode`** gates the framework's runtime assertions -- circular DI detection, missing provider errors, template validation. `provideCheckNoChangesConfig({ exhaustive: true, interval: 5000 })` enables extra checks in development and preview environments without reaching production.
- **Template errors** are indexed by code. `angular.dev/errors/NG0100` (`ExpressionChangedAfterItHasBeenCheckedError`) and `NG0200` (circular DI) account for most of them, and the URL pattern is the canonical documentation source.
- **Hydration errors** (NG0500 series) come from structural differences between server and client renders. Common causes are `window`/`document` checks, `Date.now()`, and third-party DOM mutation. `ngSkipHydration` is the pragmatic escape hatch, not a fix.
- **Zoneless change detection** fails loudly and locally. Missing signal reads and forgotten `markForCheck` calls are the two recurring bugs; `ng.profiler.timeChangeDetection()` gives a quantitative read on cycle cost.
- **Signal graph debugging** uses DevTools' graph visualizer for structural problems and ad-hoc `effect()` blocks for value-level logging. Glitches almost always trace back to an `effect()` writing a signal where a `computed()` belongs.
- **Source maps in production** are non-negotiable if you run an error tracker. Configure `hidden: true` in `angular.json`, upload the maps to your tracker, and delete them from the deployed artifact.
- **Browser DevTools** conditional breakpoints and logpoints let you instrument production code without redeployment. `ng.getComponent($0)` and friends bridge DOM selection to Angular's runtime for console-based inspection.
- **HTTP debugging** starts in the Network tab. Logging interceptors and HAR captures turn a client-side hunch into evidence the backend team can act on.
- **RxJS debugging** uses `tap` for inline logging and `rxjs-spy` for stream-wide introspection.
- **The playbook** -- reproduce, narrow, bisect, escalate with a minimal repro -- turns an open-ended bug into a closed pull request with a regression test.

The feedback loop is the thing to protect. Every tool in this chapter shortens it: DevTools turns minutes of console-logging into seconds of inspection, source maps turn anonymous stack traces into readable code, `git bisect` turns an archaeology expedition into eight command invocations. Invest in making the loop tight once, and every future bug in FinancialApp gets cheaper to fix.
