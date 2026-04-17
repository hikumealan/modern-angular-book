# Chapter 38: Web Workers

A wealth advisor opens FinancialApp and clicks "Run Monte Carlo on this portfolio." The UI freezes. The button's click animation stalls mid-ripple. The sidebar menu becomes unresponsive. Eight seconds later -- an eternity in interaction time -- a histogram appears showing the distribution of possible ten-year returns. The computation was correct. The experience was broken.

The fix is not a faster algorithm. Ten thousand Monte Carlo iterations over a forty-asset portfolio will always take a few seconds, no matter how tight the loop. The fix is *where* the work runs. JavaScript's single-threaded model gives the main thread one job: drive the UI. Anything longer than a single frame's worth of budget -- roughly 16ms at 60fps -- steals time from paints, event handlers, and scroll handlers. Long computations need to move off the main thread entirely, and Web Workers are the mechanism the platform provides for doing so.

This chapter covers dedicated Web Workers and SharedWorkers in the context of a signal-driven Angular application. We will scaffold a worker with the Angular CLI, wrap it in `Comlink` for ergonomic RPC, integrate results with signals, test both sides of the boundary, and use SharedWorker to coordinate a live price stream across multiple tabs.

> **Companion code:** `financial-app/libs/shared/workers/portfolio-analytics.worker.ts` and a Comlink-wrapped service consuming it; a `csv-parser.worker.ts` used by the transaction import from Ch 34.

---

## The Main Thread Owns the UI

Browsers give each page a single main thread. That thread runs JavaScript, computes style and layout, paints pixels, and dispatches every event from click to scroll to resize. If any task on that thread runs longer than the frame budget -- about 16ms for 60fps, 8ms for 120fps -- the browser misses a frame. Miss one frame and users rarely notice. Miss ten in a row while a computation runs and the application feels broken.

The INP metric from [Chapter 24](ch24-performance.md) captures exactly this: the time from an interaction to the next visual update. A Monte Carlo simulation executed inline in a click handler will pin INP at several seconds regardless of how well-tuned the rest of the application is. No amount of `computed()` memoization or `OnPush` discipline helps, because the thread is blocked before change detection even gets a chance to run.

For FinancialApp, three categories of work regularly threaten the main thread:

- **Portfolio risk simulations.** Monte Carlo, Value-at-Risk, and stress tests iterate tens of thousands of times over correlation matrices.
- **Large CSV parsing.** Importing a year of brokerage transactions means parsing a 20 MB file, normalizing dates and currencies, and reconciling against existing records.
- **Historical return calculations.** Computing rolling Sharpe ratios, drawdowns, or compound returns across a decade of daily prices for a forty-asset portfolio.

Each of these belongs in a worker. Leaving them on the main thread is not a performance optimization problem -- it is an architectural mistake.

---

## Web Worker vs Service Worker

These sound similar and are frequently confused. They solve different problems and rarely interact.

[Chapter 26](ch26-pwa-service-workers.md) covered **Service Workers**: a network proxy that sits between your application and the network, intercepting `fetch` events to implement caching strategies, offline fallbacks, and push notifications. A service worker has no DOM access and no synchronous API access; it receives requests, consults caches, and returns responses. There is one service worker per origin, it persists across page reloads, and its life cycle is independent of any specific tab.

This chapter covers **dedicated Web Workers** (and briefly SharedWorkers): a general-purpose mechanism for running JavaScript on a background thread. A Web Worker has no DOM access either, but it has full access to `fetch`, timers, `IndexedDB`, `WebSocket`, and most non-DOM browser APIs. It exists to run CPU-bound work off the main thread. One Web Worker typically serves one page and is torn down when that page unloads.

| Concern | Service Worker ([Ch 26](ch26-pwa-service-workers.md)) | Web Worker (this chapter) |
|---|---|---|
| Primary purpose | Network proxy, offline, push | CPU offload |
| Intercepts `fetch` | Yes | No |
| Lifetime | Persists across reloads | Lives with the page |
| Scope | One per origin | One per instance |
| DOM access | No | No |
| Typical payload | Cache manifests, routing | Hot loops, parsing, math |

If you are reaching for a Web Worker to cache API responses, you want a service worker. If you are reaching for a service worker to run a Monte Carlo simulation, you want a Web Worker. The two frequently coexist in the same application -- `ngsw-worker.js` handles caching while `portfolio-analytics.worker.ts` runs simulations -- but they are separate threads with separate responsibilities.

---

## When to Use a Web Worker

Workers have a cost. Spinning one up allocates a new JavaScript VM instance, every message across the boundary is structured-cloned (or transferred explicitly), and debugging spans two threads. A decision guide keeps the team honest:

**Use a worker when any of these apply:**

- The operation takes **longer than ~50ms** on a mid-range device. Below that threshold, the overhead of the postMessage boundary often exceeds the savings.
- The operation runs **inside a tight loop** -- reduce, map, or iterate over tens of thousands of elements -- where even a fast single pass compounds.
- The operation needs to **run in the background during user interaction**, such as re-running a risk simulation as the user drags a slider.
- The operation is **CPU-bound and has no DOM dependencies**. Parsing, number-crunching, compression, cryptography, image processing.

**Do not use a worker when:**

- The work is primarily I/O-bound. A `fetch` does not block the main thread; moving it to a worker adds ceremony without benefit.
- The work touches the DOM. Workers cannot access `document`, `window`, or any layout APIs.
- The data crossing the boundary is larger than the work itself. Serializing a 10 MB array to save 5ms of computation is a net loss -- unless you use transferable objects, covered later.
- The operation runs once at startup and never again. A one-time 80ms hit during bootstrap is acceptable; architecting a worker for it is not.

For FinancialApp, the `PortfolioAnalyticsService` and the CSV import pipeline meet the criteria. Real-time price tick handling does not -- each tick is a few hundred microseconds of work, far below the postMessage overhead.

---

## Creating a Worker with Angular

The Angular CLI scaffolds workers with a schematic that wires up the build configuration automatically. From the project root:

```bash
ng generate web-worker portfolio-analytics
```

The CLI creates or modifies four things:

1. **`src/app/portfolio-analytics.worker.ts`** -- the worker source file with a starter `onmessage` handler.
2. **`tsconfig.worker.json`** -- a TypeScript configuration that targets the `WebWorker` lib instead of `DOM`, preventing you from accidentally referencing `document` or `window` inside the worker.
3. **`angular.json`** -- the `webWorkerTsConfig` field in the project's build options is set to the new config.
4. **A sample `new Worker(new URL(...), { type: 'module' })`** snippet inserted into the file that invoked the schematic, showing the recommended instantiation syntax.

The URL-based constructor is critical: passing `new URL('./portfolio-analytics.worker', import.meta.url)` lets the Angular builder (esbuild under the hood) discover the worker at build time, bundle its dependencies, and emit a separate chunk. A plain string path would not work -- the builder would have no way to know the file is a worker entry point.

In an Nx monorepo (see [Chapter 14](ch14-monorepos-libraries.md)), workers belong in a library with clear boundaries. The FinancialApp places compute-heavy workers in `libs/shared/workers/`, alongside the services that wrap them. This keeps the worker file, its types, and its consumer in the same bounded context and lets other apps in the monorepo reuse them.

---

## Raw `postMessage` / `onmessage`

Before reaching for a library, it is worth seeing the native API. A minimal worker looks like this:

```typescript
addEventListener('message', ({ data }) => {
  if (data.type === 'simulateReturns') {
    const result = runSimulation(data.payload);
    postMessage({ type: 'simulateReturns:result', result });
  }
});

function runSimulation(input: SimulationInput): SimulationResult {
  // ... Monte Carlo iterations ...
}
```

And the consumer:

```typescript
const worker = new Worker(
  new URL('./portfolio-analytics.worker', import.meta.url),
  { type: 'module' },
);

worker.postMessage({ type: 'simulateReturns', payload: input });

worker.addEventListener('message', ({ data }) => {
  if (data.type === 'simulateReturns:result') {
    console.log(data.result);
  }
});
```

This works, but the pattern does not scale. Every method needs a message type, every result needs a paired response type, and correlating a specific call to its response requires request IDs if concurrent calls are allowed. TypeScript cannot help -- the worker boundary is a string-keyed dictionary.

There is also the **serialization cost** to consider. `postMessage` uses the structured clone algorithm, which recursively copies every object crossing the boundary. Primitives and plain objects clone cheaply; `Map`, `Set`, and typed arrays clone more expensively; class instances lose their prototype chain. For a 50 MB array of transaction records, the clone can take longer than the work it supports.

Two refinements address both problems: `Comlink` for ergonomics, and transferable objects for large payloads.

---

## Comlink for Ergonomic RPC

`Comlink` is a tiny library (~1 KB gzipped) that wraps the `postMessage` boundary in a proxy-based RPC layer. Methods called on the proxy generate messages, return values come back as Promises, and TypeScript types flow across the boundary as if the worker were just another class.

Install it:

```bash
npm install comlink
```

The worker exposes a class. `Comlink.expose()` registers it on the worker side; every public method becomes remotely callable:

```typescript
// libs/shared/workers/src/portfolio-analytics.worker.ts
import * as Comlink from 'comlink';

export interface AssetReturn { ticker: string; monthlyReturns: number[]; weight: number; }
export interface SimulationResult { p5: number; p50: number; p95: number; }
export interface RiskResult { valueAtRisk: number; expectedShortfall: number; }

export class PortfolioAnalyticsWorker {
  async simulateReturns(assets: AssetReturn[], horizonMonths: number, iterations = 10_000): Promise<SimulationResult> {
    const results = new Float64Array(iterations);
    for (let i = 0; i < iterations; i++) results[i] = this.simulatePath(assets, horizonMonths);
    results.sort();
    return {
      p5: results[Math.floor(iterations * 0.05)],
      p50: results[Math.floor(iterations * 0.5)],
      p95: results[Math.floor(iterations * 0.95)],
    };
  }

  async calculateSharpe(returns: number[], riskFreeRate: number): Promise<number> {
    const mean = returns.reduce((s, r) => s + r, 0) / returns.length;
    const variance = returns.reduce((s, r) => s + (r - mean) ** 2, 0) / returns.length;
    const stdDev = Math.sqrt(variance);
    return stdDev === 0 ? 0 : (mean - riskFreeRate) / stdDev;
  }

  async monteCarloRisk(assets: AssetReturn[], confidence = 0.95, iterations = 50_000): Promise<RiskResult> {
    const losses = new Float64Array(iterations);
    for (let i = 0; i < iterations; i++) losses[i] = -this.simulatePath(assets, 1);
    losses.sort((a, b) => b - a);
    const varIdx = Math.floor(iterations * (1 - confidence));
    const tail = losses.slice(0, varIdx);
    return {
      valueAtRisk: losses[varIdx],
      expectedShortfall: tail.reduce((s, l) => s + l, 0) / tail.length,
    };
  }

  private simulatePath(assets: AssetReturn[], months: number): number {
    let portfolioReturn = 0;
    for (const a of assets) {
      let path = 1;
      for (let m = 0; m < months; m++) path *= 1 + a.monthlyReturns[(Math.random() * a.monthlyReturns.length) | 0];
      portfolioReturn += a.weight * (path - 1);
    }
    return portfolioReturn;
  }
}

Comlink.expose(new PortfolioAnalyticsWorker());
```

The consumer wraps the worker:

```typescript
// libs/shared/workers/src/portfolio-analytics.service.ts
import { Injectable, OnDestroy } from '@angular/core';
import * as Comlink from 'comlink';
import type { PortfolioAnalyticsWorker } from './portfolio-analytics.worker';

@Injectable({ providedIn: 'root' })
export class PortfolioAnalyticsService implements OnDestroy {
  private worker = new Worker(
    new URL('./portfolio-analytics.worker', import.meta.url),
    { type: 'module' },
  );

  private analytics = Comlink.wrap<PortfolioAnalyticsWorker>(this.worker);

  simulateReturns = this.analytics.simulateReturns.bind(this.analytics);
  calculateSharpe = this.analytics.calculateSharpe.bind(this.analytics);
  monteCarloRisk = this.analytics.monteCarloRisk.bind(this.analytics);

  ngOnDestroy(): void {
    this.worker.terminate();
  }
}
```

Callers use it exactly as if it were a local async service. The simulation runs on a separate thread; the main thread continues painting, handling clicks, and updating signals. TypeScript enforces the method signatures on both sides from a single `PortfolioAnalyticsWorker` type.

---

## Integrating Workers with Signals

Worker results are asynchronous, and Angular v21's `resource` primitive handles async state beautifully. Pair a signal-based input with a resource whose loader invokes the worker, and you get loading, error, and value states for free -- without writing any subscription glue.

```typescript
// domains/portfolio/risk-panel.component.ts
import { Component, inject, input, signal, resource } from '@angular/core';
import { PortfolioAnalyticsService, type AssetReturn } from '@financial-app/shared/workers';

@Component({
  selector: 'app-risk-panel',
  template: `
    <h3>Portfolio Risk</h3>
    @switch (riskResource.status()) {
      @case ('loading') { <p>Running Monte Carlo simulation...</p> }
      @case ('error')   { <p class="error">Simulation failed.</p> }
      @case ('resolved') {
        <dl>
          <dt>Value at Risk</dt>
          <dd>{{ riskResource.value()?.valueAtRisk | percent:'1.2-2' }}</dd>
          <dt>Expected Shortfall</dt>
          <dd>{{ riskResource.value()?.expectedShortfall | percent:'1.2-2' }}</dd>
        </dl>
      }
    }
  `,
})
export class RiskPanelComponent {
  private analytics = inject(PortfolioAnalyticsService);
  assets = input.required<AssetReturn[]>();
  confidence = signal(0.95);

  riskResource = resource({
    params: () => ({ assets: this.assets(), confidence: this.confidence() }),
    loader: ({ params }) => this.analytics.monteCarloRisk(params.assets, params.confidence),
  });
}
```

Every time `assets()` or `confidence()` changes, the resource re-runs the loader, which dispatches a message to the worker. The template reactively renders loading, resolved, or error states based on `riskResource.status()`. The main thread never blocks. Users can change the confidence dropdown while the simulation runs and the UI stays responsive.

For long-running computations, add a cancellation token. Comlink does not cancel in-flight work natively -- the worker keeps running until the current iteration completes -- but you can terminate and recreate the worker if the user abandons the view. A provider-scoped service (`providedIn: null` with explicit route-level provisioning) gives you lifecycle control.

---

## Transferable Objects

Structured clone copies every byte across the worker boundary. For a 40 MB typed array of historical prices, the copy alone can cost 100ms on both threads -- 200ms total before any computation happens.

**Transferable objects** solve this by moving ownership instead of copying. An `ArrayBuffer`, `MessagePort`, `ImageBitmap`, or `OffscreenCanvas` marked as transferable is handed off to the receiving thread and becomes unusable on the sender. The transfer is zero-copy: just a pointer move.

With raw `postMessage`, you pass the transfer list as the second argument:

```typescript
const prices = new Float64Array(10_000_000);
worker.postMessage({ prices }, [prices.buffer]);
```

With Comlink, the `Comlink.transfer()` wrapper marks values for transfer:

```typescript
const prices = new Float64Array(10_000_000);
const result = await service.calculateRollingVolatility(
  Comlink.transfer(prices, [prices.buffer]),
  30,
);
```

After the call, `prices` on the main thread is a zero-length view of a detached buffer -- attempting to read it throws. Transfer is appropriate when the main thread no longer needs the data; copy when both sides will use it.

For FinancialApp's CSV import from [Chapter 34](ch34-file-handling.md), the file's `ArrayBuffer` is read from the `File` object and transferred to a `csv-parser.worker.ts`. The worker decodes, parses, validates, and returns structured transaction records. The main thread's buffer is gone by the time the worker begins, freeing memory and avoiding a copy of tens of megabytes:

```typescript
// csv-import.service.ts
async importTransactions(file: File): Promise<ParsedTransaction[]> {
  const buffer = await file.arrayBuffer();
  return this.csvWorker.parseTransactions(
    Comlink.transfer(buffer, [buffer]),
  );
}
```

The same pattern applies to chart rendering. [Chapter 35](ch35-charts.md) covers charts, but the integration point is worth noting here: a worker that computes an indicator -- moving average, Bollinger bands, RSI -- produces a `Float32Array` that gets transferred to the chart library for rendering. No copy, no double allocation.

---

## Testing Workers

Testing a worker requires a strategy on each side of the boundary. The worker's class -- `PortfolioAnalyticsWorker` above -- is just a class with public methods. Import and test it directly, no worker instantiation required:

```typescript
// libs/shared/workers/src/portfolio-analytics.worker.spec.ts
import { describe, expect, it } from 'vitest';
import { PortfolioAnalyticsWorker } from './portfolio-analytics.worker';

describe('PortfolioAnalyticsWorker', () => {
  const worker = new PortfolioAnalyticsWorker();

  it('calculates Sharpe ratio correctly', async () => {
    const sharpe = await worker.calculateSharpe([0.02, 0.03, -0.01, 0.04, 0.01], 0.001);
    expect(sharpe).toBeCloseTo(0.86, 1);
  });

  it('produces ordered percentiles for simulations', async () => {
    const assets = [{ ticker: 'VTI', monthlyReturns: [0.01, -0.02, 0.03], weight: 1 }];
    const result = await worker.simulateReturns(assets, 12, 1_000);
    expect(result.p5).toBeLessThan(result.p50);
    expect(result.p50).toBeLessThan(result.p95);
  });
});
```

This is the same Vitest setup from [Chapter 7](ch07-testing-vitest.md) -- no special environment, no worker globals. The worker code under test is pure TypeScript that happens to be exposed via Comlink; the testing concern is its correctness, not its threading model.

For integration tests that verify the Comlink boundary itself, Node's `worker_threads` module can execute the worker file, and Vitest's `happy-dom` environment provides the `Worker` global for tests that exercise the service. In practice most teams mock the service at the consumer level instead -- a test for `RiskPanelComponent` provides a fake `PortfolioAnalyticsService` whose methods return canned data through `vi.fn().mockResolvedValue({...})`, and `TestBed` injects it in place of the real service.

Stubbing at the service boundary means component tests never spin up a real worker, never wait for structured cloning, and never flake on CI machines with limited CPU. The worker's own logic is covered by the direct class tests above; the component's interaction with the service is covered by the mock. Both layers are fast and deterministic.

---

## Build Considerations

The Angular CLI and Vite-based builder handle workers automatically when you use the `new URL(...)` pattern. Under the hood, esbuild recognizes the URL literal as a worker entry point, bundles it into its own chunk, and rewrites the URL to the emitted file. The chunk is code-split from the main application -- it downloads only when the consumer instantiates the worker -- and supports dynamic imports from inside the worker itself.

Two Vite-specific suffixes are worth knowing if you drop below the Angular CLI into direct Vite configuration (relevant for the Vite-powered Storybook setup in [Chapter 28](ch28-storybook.md) or custom build scripts):

- **`?worker`** imports a module as a worker constructor. `import MyWorker from './my.worker?worker'` gives you a class you can instantiate with `new MyWorker()`.
- **`?sharedworker`** does the same for `SharedWorker`.

In application code you rarely need these -- the Angular builder's `new URL` pattern covers the common case -- but they come up in tooling and in shared Vite libraries.

Chunk splitting for workers is automatic, but keep the worker bundle small deliberately. A worker that imports half of Lodash pays the download cost every time it spawns. Prefer small, focused worker modules. If multiple features need similar math, colocate them in one worker class rather than spawning several workers.

---

## SharedWorker for Cross-Tab Coordination

A user opens FinancialApp in two tabs. Each tab establishes a WebSocket to the live price stream. The server pushes every tick twice -- once per connection -- and the client renders the same update twice, doubling network usage and battery drain on mobile. If the user opens five tabs, the server sends five copies of every tick.

**SharedWorker** solves this by letting multiple tabs, from the same origin, share a single worker instance. The worker opens the WebSocket once, and tabs communicate with it over `MessagePort`. Every tab sees the same stream; the server sees one connection.

```typescript
// libs/shared/workers/src/price-stream.sharedworker.ts
const socket = new WebSocket('wss://api.financial-app.com/prices');
const ports = new Set<MessagePort>();
const lastPrices = new Map<string, number>();

socket.addEventListener('message', ({ data }) => {
  const tick = JSON.parse(data);
  lastPrices.set(tick.ticker, tick.price);
  for (const port of ports) port.postMessage({ type: 'tick', tick });
});

addEventListener('connect', (event: MessageEvent) => {
  const port = (event as MessageEvent & { ports: MessagePort[] }).ports[0];
  ports.add(port);
  port.postMessage({ type: 'snapshot', prices: Object.fromEntries(lastPrices) });
  port.start();
});
```

The consumer connects with `SharedWorker` rather than `Worker` and bridges ticks into a signal:

```typescript
// price-stream.service.ts
@Injectable({ providedIn: 'root' })
export class PriceStreamService {
  private worker = new SharedWorker(
    new URL('./price-stream.sharedworker', import.meta.url),
    { type: 'module', name: 'price-stream' },
  );

  readonly prices = signal<Record<string, number>>({});

  constructor() {
    this.worker.port.addEventListener('message', ({ data }) => {
      if (data.type === 'snapshot') this.prices.set(data.prices);
      else if (data.type === 'tick') this.prices.update(p => ({ ...p, [data.tick.ticker]: data.tick.price }));
    });
    this.worker.port.start();
  }
}
```

The first tab to open creates the SharedWorker, which establishes the WebSocket. Subsequent tabs connect to the same worker and receive the current snapshot followed by live ticks. When the last tab closes, the browser tears down the SharedWorker and its WebSocket. One connection, N tabs.

SharedWorker support is excellent on Chrome, Edge, and Firefox; Safari shipped support in 16.4 (March 2023) after a long hiatus. For applications that must support older Safari versions, fall back to `BroadcastChannel` with one tab electing itself as the WebSocket owner -- more complex, but compatible.

---

## Summary

Web Workers move CPU-bound work off the main thread, keeping the UI responsive during computation that would otherwise jank every frame. They are an architectural tool, not a micro-optimization.

- **The main thread owns the UI.** Any work over the 16ms frame budget causes jank. Monte Carlo, CSV parsing, and historical return calculations belong in a worker.
- **Web Workers and Service Workers are different.** Service workers ([Chapter 26](ch26-pwa-service-workers.md)) proxy the network for caching and offline; Web Workers run CPU-bound code. They cooperate but do not overlap.
- **Decide by operation cost.** Operations over ~50ms, tight loops, work that must proceed during interaction, and CPU-bound code with no DOM access all qualify. Sub-millisecond event handlers and I/O-bound `fetch` calls do not.
- **`ng generate web-worker`** scaffolds the worker file, a dedicated `tsconfig.worker.json` targeting the `WebWorker` lib, and the builder configuration. The `new Worker(new URL(...), { type: 'module' })` pattern lets the Angular builder discover and bundle the worker.
- **Raw `postMessage` works but does not scale.** Every method needs a message type, responses need correlation IDs, and the boundary is untyped.
- **`Comlink.wrap<T>()`** turns the worker into a typed async proxy. Methods on the proxy dispatch messages, return Promises, and share TypeScript types across the boundary.
- **Signals integrate via `resource()`.** A resource whose loader calls a Comlink method gives you loading, error, and resolved states bound to signal inputs, with no subscription glue.
- **Transferable objects** move `ArrayBuffer` ownership across the boundary instead of copying, essential for CSV imports ([Chapter 34](ch34-file-handling.md)) and chart datasets ([Chapter 35](ch35-charts.md)).
- **Test the worker class directly.** Import the class and call its methods -- no worker instantiation required. Stub the Comlink-wrapped service at the consumer level for component tests.
- **Vite and esbuild handle bundling automatically.** The `new URL` pattern emits a code-split chunk. `?worker` and `?sharedworker` query suffixes are Vite-native alternatives for direct Vite configuration.
- **SharedWorker coordinates across tabs.** One WebSocket, one parsing pipeline, one in-memory cache shared across every open tab of the origin. Ideal for live price streams, authentication token refresh, and cross-tab state propagation.

The architectural payoff compounds with the rest of the book. Zoneless change detection ([Chapter 24](ch24-performance.md)) means worker-driven signal updates propagate without Zone.js interception. Service workers ([Chapter 26](ch26-pwa-service-workers.md)) cache the worker bundle so subsequent page loads skip the network. Monorepo boundaries ([Chapter 14](ch14-monorepos-libraries.md)) keep worker implementations and their service wrappers in one library. The worker is not a special-case optimization; it is another thread in the application's architecture, and treating it as such is what keeps FinancialApp responsive even when the math is heavy.
