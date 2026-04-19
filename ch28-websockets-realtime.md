# Chapter 33: WebSockets & Real-Time Data

The FinancialApp's dashboard used to refresh every thirty seconds. A user would watch a position, refresh their browser, and still see a stale price. In a quiet market that was a mild annoyance; during an earnings surprise it was a problem. When the product team added live streaming prices, a news ticker for portfolio holdings, and push notifications for filled trades, the old "refresh on a timer" model collapsed. HTTP is excellent at request/response, but a dashboard that needs to react to market events in under 200 milliseconds is a different shape of problem.

This chapter covers the transport options for real-time data in an Angular application, the patterns for wrapping those transports in RxJS and signals, and the production concerns -- reconnection, backpressure, authentication, protocol design -- that determine whether the feature ships or rolls back after the first outage. Every example targets the FinancialApp's live-prices and trade-notifications features.

> **Prerequisite:** [Chapter 10](ch08-rxjs-deep-dive.md) for RxJS fundamentals; [Chapter 3](ch03-reactive-signals.md) for `resource` stream API.

> **Companion code:** `financial-app/libs/shared/realtime/` with `PriceStreamService`, reconnect utilities, and a mock WebSocket server at `mock-api/ws-server.mjs`.

---

## Why HTTP Alone Is Not Enough

A classic financial dashboard consumes three categories of data:

- **Reference data** -- the user's accounts, holdings, preferences. Changes infrequently; HTTP caching works perfectly.
- **Historical data** -- end-of-day positions, last-month's transactions, audit logs. Immutable once written; HTTP with strong `ETag` headers is ideal.
- **Live data** -- current prices, intraday portfolio values, trade-fill notifications, chat messages from an advisor. Changes continuously; pulling on a timer wastes bandwidth and lags behind reality.

The live category is the interesting one. A portfolio view with a dozen holdings, each price updating a few times per second, generates hundreds of data points per minute. If you poll every second, you pay the full TLS handshake and request overhead for every update; if you poll every thirty seconds, your users see a prices lag that looks like a bug. Neither is acceptable. The server needs to push data to the client as it becomes available, and the client needs to stay connected long enough to receive it.

Picking the right transport is a trade-off between protocol capability, infrastructure cost, and developer effort. The rest of this section walks through the options, and the rest of the chapter builds the FinancialApp around the winner.

---

## Real-Time Transport Options

Four mainstream options exist for server-to-client data flow on the web. Each has a profile worth understanding before committing.

| Transport | Direction | Reconnect | Proxy/CDN friendliness | When to pick it |
|---|---|---|---|---|
| **Polling** | Client pulls | Trivially "reconnects" every request | Perfect -- plain HTTP | Low-frequency updates (dashboards refreshed once a minute), simple infrastructure |
| **Long polling** | Client pulls, server holds | Per-request | Works over any HTTP proxy | Push-like semantics without protocol upgrades; legacy environments |
| **Server-Sent Events (SSE)** | Server -> client only | Browser reconnects automatically | Works over HTTP/1.1 and HTTP/2 with minor config | One-way push: notifications, logs, price feeds where the client does not need to send |
| **WebSockets** | Full duplex | Manual | Requires `Upgrade` handshake; some proxies strip it | Bidirectional streams: chat, collaborative editing, interactive trading |

Polling is the lowest-effort option and the easiest to reason about; it is also the least efficient at high frequencies. Long polling trades efficiency for push semantics but is operationally awkward -- connections that time out need careful handling on both ends. SSE is the sweet spot for one-way feeds: it runs over plain HTTP, the `EventSource` browser API handles reconnection for you, and any CDN that speaks HTTP/1.1 can pass it through. WebSockets are the only option when the client needs to send data upstream in the same connection as it receives downstream data -- think chat messages, order submissions interleaved with fill notifications, or collaborative editing.

The FinancialApp uses WebSockets for the live-prices and trade-events feeds because the same connection handles subscription requests (`subscribe` to these tickers) and server-initiated updates (prices, fills, news). It uses SSE in a secondary path for a lightweight notifications drawer where the server has nothing to ask the client. Both are production-grade; the choice is about protocol fit, not one being "better."

---

## WebSockets with RxJS

RxJS ships a `webSocketSubject` helper in the `rxjs/webSocket` entry point. It is a `Subject<T>` that happens to be backed by a browser `WebSocket`, which means it plays naturally with every RxJS operator covered in [Chapter 10](ch08-rxjs-deep-dive.md). Open a connection by reading the subject; send by calling `.next()`; receive by subscribing.

The FinancialApp's `PriceStreamService` wraps a subject, exposes a signal-backed API to components, and handles the lifecycle details. Here is the core:

```typescript
// libs/shared/realtime/src/lib/price-stream.service.ts
import { Injectable, inject, signal } from '@angular/core';
import { webSocket, WebSocketSubject } from 'rxjs/webSocket';
import { retry, timer, Subject, filter, map } from 'rxjs';
import { AuthService } from '@financial-app/shared/auth';

interface PriceMessage {
  kind: 'price';
  ticker: string;
  price: number;
  ts: number;
}

type OutgoingMessage =
  | { kind: 'subscribe'; tickers: string[] }
  | { kind: 'unsubscribe'; tickers: string[] }
  | { kind: 'auth'; token: string };

@Injectable({ providedIn: 'root' })
export class PriceStreamService {
  private auth = inject(AuthService);
  private socket$: WebSocketSubject<PriceMessage | OutgoingMessage> | null = null;

  readonly connectionState = signal<'disconnected' | 'connecting' | 'connected'>('disconnected');

  connect(): void {
    if (this.socket$) return;
    this.connectionState.set('connecting');
    this.socket$ = webSocket({
      url: 'wss://example.com/prices',
      openObserver: { next: () => this.onOpen() },
      closeObserver: { next: () => this.onClose() },
    });
  }

  prices(ticker: string) {
    return this.socket$!.pipe(
      filter((msg): msg is PriceMessage => (msg as PriceMessage).kind === 'price'),
      filter((msg) => msg.ticker === ticker),
      map((msg) => ({ price: msg.price, ts: msg.ts })),
    );
  }

  subscribe(tickers: string[]): void {
    this.socket$!.next({ kind: 'subscribe', tickers });
  }

  private onOpen(): void {
    this.connectionState.set('connected');
    this.socket$!.next({ kind: 'auth', token: this.auth.token() });
  }

  private onClose(): void {
    this.connectionState.set('disconnected');
  }
}
```

Three things are worth noticing. First, `webSocket()` does not open the connection until something subscribes -- it is a cold observable, like most RxJS factories. The FinancialApp triggers this on the first call to `connect()`, but in other designs the first `prices(ticker)` subscription can serve the same purpose.

Second, messages flow in both directions through the same subject. `.next(outgoing)` sends; subscribing receives. `WebSocketSubject<T>` is typed such that `T` describes both directions, which works well when incoming and outgoing messages share a discriminated-union shape.

Third, the `openObserver` and `closeObserver` hooks fire on the lifecycle events of the underlying `WebSocket`, not on RxJS subscription activity. Use them to signal connection state, not to handle subscription cleanup -- a subscriber unsubscribing does not necessarily close the socket.

---

## Reconnection and Backoff

A browser `WebSocket` will drop silently on a laptop lid close, a network switch, a server restart, or a proxy's idle timeout. Any production-grade service needs a reconnection strategy, and "exponential backoff" is the strategy that does not set your infrastructure on fire when the server is in trouble.

RxJS's `retry` operator accepts a `delay` function that returns an observable; the stream resubscribes when that observable emits. Combine it with `timer()` and a cap:

```typescript
// libs/shared/realtime/src/lib/price-stream.service.ts (continued)
import { retry, timer, EMPTY, fromEvent, merge } from 'rxjs';
import { toObservable } from '@angular/core/rxjs-interop';

private resilientSocket$ = this.socket$!.pipe(
  retry({
    delay: (_error, retryCount) => {
      const backoff = Math.min(30_000, 1_000 * 2 ** retryCount);
      this.connectionState.set('connecting');
      return timer(backoff);
    },
  }),
);
```

The sequence is `1s, 2s, 4s, 8s, 16s, 30s, 30s, ...`. Three dynamics matter here:

- **Cap the delay.** Without `Math.min(30_000, ...)`, a persistent outage grows the delay to minutes, then hours. That can be desirable for mobile clients that should stop retrying when offline, but for a dashboard you want a quick reconnect when service comes back.
- **Add jitter.** If every client disconnects at the same moment (the server restarts), they will all reconnect at the same moment and create a thundering herd. Multiply the backoff by `0.5 + Math.random() * 0.5` to spread reconnects across a window.
- **Reset on success.** `retry` automatically resets the counter when the stream emits again after reconnection, so a transient drop does not inherit the delay from an earlier outage.

Integrating the browser's online/offline events lets the service distinguish "server outage" from "laptop is offline":

```typescript
// libs/shared/realtime/src/lib/online-status.ts
import { Observable, fromEvent, merge, startWith, map } from 'rxjs';

export const online$: Observable<boolean> = merge(
  fromEvent(window, 'online').pipe(map(() => true)),
  fromEvent(window, 'offline').pipe(map(() => false)),
).pipe(startWith(navigator.onLine));
```

When `online$` emits `false`, pause the reconnect loop; when it emits `true`, resume immediately instead of waiting for the next backoff tick. This produces a better user experience on spotty networks -- the dashboard reconnects the instant Wi-Fi returns rather than at the next scheduled attempt.

The three-state `connectionState` signal (`disconnected` / `connecting` / `connected`) becomes the source of truth for the UI's connection indicator:

```typescript
// apps/financial-app/src/app/features/dashboard/connection-badge.component.ts
@Component({
  selector: 'app-connection-badge',
  template: `
    @switch (state()) {
      @case ('connected') { <span class="badge badge--ok">Live</span> }
      @case ('connecting') { <span class="badge badge--warn">Reconnecting...</span> }
      @case ('disconnected') { <span class="badge badge--err">Offline</span> }
    }
  `,
})
export class ConnectionBadgeComponent {
  private stream = inject(PriceStreamService);
  readonly state = this.stream.connectionState;
}
```

Users know exactly what the app is doing, and support tickets that start with "are your servers down?" become self-answering.

---

## Integrating with Signals

Most of the FinancialApp's UI is signal-driven. WebSockets deliver a stream of events, which is inherently an observable model. The boundary between the two is where the interop utilities from `@angular/core/rxjs-interop` earn their keep.

For a single ticker, `toSignal()` converts the observable returned by `prices(ticker)` into a signal that updates whenever a new message arrives:

```typescript
// apps/financial-app/src/app/features/dashboard/ticker-row.component.ts
import { toSignal } from '@angular/core/rxjs-interop';

@Component({
  selector: 'app-ticker-row',
  template: `
    <span class="ticker">{{ ticker() }}</span>
    <span class="price">{{ latest()?.price | currency }}</span>
  `,
})
export class TickerRowComponent {
  private stream = inject(PriceStreamService);
  readonly ticker = input.required<string>();

  readonly latest = toSignal(
    toObservable(this.ticker).pipe(switchMap((t) => this.stream.prices(t))),
    { initialValue: null },
  );
}
```

The `switchMap` is essential: when the parent component reassigns `ticker` from `AAPL` to `MSFT`, the previous subscription is unsubscribed and a new one starts, automatically routing the component to the new ticker's stream.

For a dashboard that renders many tickers, `rxResource()` with its `stream` property provides a uniform state machine (loading, error, value) over a streaming source:

```typescript
// apps/financial-app/src/app/features/dashboard/portfolio-prices.component.ts
import { rxResource } from '@angular/core/rxjs-interop';

@Component({ /* ... */ })
export class PortfolioPricesComponent {
  private stream = inject(PriceStreamService);
  readonly portfolioId = input.required<PortfolioId>();

  readonly quotes = rxResource({
    params: () => ({ portfolioId: this.portfolioId() }),
    stream: ({ params }) =>
      this.stream.pricesForPortfolio(params.portfolioId),
  });
}
```

`rxResource` was introduced in [Chapter 3](ch03-reactive-signals.md) as the bridge between observables and the signal graph. Its `stream` property is a natural fit for a feed that emits repeatedly rather than completing after one value. `quotes.value()` returns the latest map of ticker-to-price, `quotes.status()` reports whether the stream is active, and changes to `portfolioId` re-subscribe automatically -- same semantics as an HTTP call, but over a live connection.

For code that needs an imperative escape hatch -- a ticker component that persists recent prices into a local ring buffer, for instance -- an `effect()` reading the signal does the job. The rule of thumb from [Chapter 3](ch03-reactive-signals.md) still applies: derive with `computed`, side-effect with `effect`, never propagate state through an effect chain.

---

## Backpressure

A server streaming prices at 50 Hz can overwhelm a client that renders at 60 fps if every message triggers change detection. The symptom is a dashboard that feels sluggish even though the network looks fine, because the main thread is spending all its time in Angular's render pipeline. Backpressure operators from RxJS are how you throttle the firehose.

Three operators cover the common cases:

- **`throttleTime(ms)`** -- emit the first value, then ignore emissions for `ms` milliseconds. Use for "show the latest, but no more than N per second."
- **`auditTime(ms)`** / **`audit(selector)`** -- ignore emissions for `ms` milliseconds, then emit the *last* value received during the window. Best for "always show the freshest, but bound the rate."
- **`sampleTime(ms)`** / **`sample(notifier$)`** -- emit the latest value at a fixed interval, regardless of source rate. Best for synchronized snapshots at animation frame boundaries.

For the price ticker, `auditTime` is usually the right choice:

```typescript
// libs/shared/realtime/src/lib/price-stream.service.ts (continued)
import { auditTime } from 'rxjs';

tickerAtFrameRate(ticker: string) {
  return this.prices(ticker).pipe(auditTime(16));
}
```

At most one update every 16 milliseconds -- one per 60 Hz frame -- with the freshest price always surfaced. If the server pauses for a second and then dumps five updates, the component sees the latest of those five rather than replaying the stale history. The visible behavior is smooth: price tape updates at the eye's refresh rate; memory usage stays flat.

For a grid of many tickers, combining `auditTime` with `scan` to accumulate a map is the pattern:

```typescript
// libs/shared/realtime/src/lib/price-stream.service.ts (continued)
import { scan, auditTime } from 'rxjs';

allPricesThrottled() {
  return this.socket$!.pipe(
    filter((m): m is PriceMessage => m.kind === 'price'),
    scan((map, m) => {
      map.set(m.ticker, m.price);
      return new Map(map);
    }, new Map<string, number>()),
    auditTime(16),
  );
}
```

`scan` maintains the running map; `auditTime(16)` rate-limits the emissions downstream. The component receives at most 60 full-map snapshots per second, which is plenty for any human-visible UI. At 5,000 updates per second from the server, the CPU savings are enormous -- the application no longer renders every tick, only the last one per frame.

For truly high-volume feeds (order books, dense trading dashboards), delegate the aggregation to a Web Worker and post frame-synchronized snapshots back to the main thread. The main-thread-side code is identical; the worker simply does the `scan` + `auditTime` work off the critical rendering path. That pattern is covered in [Chapter 9](ch32-performance.md).

---

## Server-Sent Events

For the FinancialApp's lightweight notifications drawer -- price alerts triggered by server-side rules, trade-fill messages, system announcements -- the client never needs to send anything to the server after the initial connection. SSE is a better fit than WebSockets in this case: simpler infrastructure, free reconnection, and it works over plain HTTP.

The browser's `EventSource` API is the primitive:

```typescript
// libs/shared/realtime/src/lib/notifications.service.ts
import { Injectable, NgZone, inject, signal } from '@angular/core';
import { Observable } from 'rxjs';

interface NotificationPayload {
  id: string;
  title: string;
  body: string;
  severity: 'info' | 'warn' | 'error';
}

@Injectable({ providedIn: 'root' })
export class NotificationsService {
  private zone = inject(NgZone);

  readonly unread = signal(0);

  stream(): Observable<NotificationPayload> {
    return new Observable<NotificationPayload>((subscriber) => {
      const source = this.zone.runOutsideAngular(
        () => new EventSource('/api/notifications/stream', { withCredentials: true }),
      );

      source.onmessage = (event) => {
        const payload = JSON.parse(event.data) as NotificationPayload;
        this.zone.run(() => subscriber.next(payload));
      };

      source.onerror = () => {
        // EventSource reconnects on its own; surface if it enters CLOSED.
        if (source.readyState === EventSource.CLOSED) {
          subscriber.error(new Error('SSE stream closed'));
        }
      };

      return () => source.close();
    });
  }
}
```

Two Angular-specific concerns shape this code.

`NgZone.runOutsideAngular()` is relevant only in applications that still use zone.js. Constructing the `EventSource` triggers zone patching on `onmessage`/`onerror`, which can cause a change-detection run for every notification, swamping the scheduler on busy feeds. Running the constructor outside the zone and hopping back in with `this.zone.run()` only when a notification needs to surface keeps the main thread cool. In a zoneless application (the default since Angular v19), this entire concern disappears -- `NgZone` is still injectable for compatibility, but its methods are no-ops.

The browser's `EventSource` reconnects automatically with exponential backoff baked in, so the manual reconnection code from the WebSocket section is unnecessary. If the server sends an `id:` field in its event stream, the browser will send a `Last-Event-ID` header on reconnect, allowing the server to resume from where it left off. This is the major operational advantage of SSE over raw WebSockets.

Converting the observable to a signal and the unread-count side effect to a signal-aware pattern looks the same as with WebSockets:

```typescript
// apps/financial-app/src/app/features/notifications/notifications.component.ts
readonly notifications = toSignal(
  this.notificationsService.stream().pipe(scan((list, n) => [n, ...list].slice(0, 50), [] as NotificationPayload[])),
  { initialValue: [] },
);
```

The accumulated list is capped at 50 to avoid unbounded growth during long sessions. Persistence for notifications that outlive the session is a job for the SignalStore from [Chapter 12](ch10-ngrx-signal-store.md) plus IndexedDB caching from [Chapter 26](ch33-pwa-service-workers.md).

---

## Protocol Design

A real-time feed shares its contract between frontend and backend the same way an HTTP API does. Skipping the design step produces an apparent shortcut that causes pain on the first backward-incompatible change; doing it up front costs a small amount of forethought and saves months of migration work later.

### Message envelopes

Every message sent or received carries a `kind` discriminant and a version tag. That enables discriminated unions in TypeScript and reserves room for protocol evolution:

```typescript
// libs/shared/realtime/src/lib/protocol.ts
export interface Envelope<T extends string, P> {
  v: 1;
  kind: T;
  payload: P;
}

export type Incoming =
  | Envelope<'price', { ticker: string; price: number; ts: number }>
  | Envelope<'fill', { orderId: string; filled: number; ts: number }>
  | Envelope<'news', { headline: string; url: string; ts: number }>
  | Envelope<'pong', { ts: number }>;

export type Outgoing =
  | Envelope<'auth', { token: string }>
  | Envelope<'subscribe', { tickers: string[] }>
  | Envelope<'unsubscribe', { tickers: string[] }>
  | Envelope<'ping', { ts: number }>;
```

The `v: 1` literal forces the server and client to agree on the protocol version. When a breaking change ships, both sides move to `v: 2` and the client can negotiate support during the handshake. Envelopes also let the frontend log unknown `kind` values safely rather than crashing on schema drift -- a structured failure mode is far better than a silent one.

### Heartbeats

A proxy or load balancer will close an idle WebSocket somewhere between 30 seconds and 5 minutes of silence. A heartbeat -- a `ping`/`pong` exchange at a known cadence -- keeps the connection alive and gives both sides a way to detect half-open sockets:

```typescript
// libs/shared/realtime/src/lib/heartbeat.ts
import { interval, timer, takeUntil, race, EMPTY, Subject } from 'rxjs';

export function heartbeat$(ping: () => void, interval_ms = 20_000, timeout_ms = 5_000) {
  const timeout$ = new Subject<void>();
  return interval(interval_ms).pipe(
    tap(() => {
      ping();
      setTimeout(() => timeout$.next(), timeout_ms);
    }),
    takeUntil(timeout$),
  );
}
```

Fire a `ping` every 20 seconds; if no `pong` arrives within 5 seconds, assume the socket is dead and force-close it so the reconnect logic can run. Without this, a broken socket can "succeed" on `.send()` for minutes before the TCP stack gives up, during which the user sees stale data with no warning. On the server side, matching logic should disconnect a client that does not ping as expected. Symmetry here saves debugging time when production connections behave strangely.

---

## Testing

Real-time code is notoriously hard to test if you treat the `WebSocket` global as opaque. The trick is to substitute a controllable mock, then assert on the full lifecycle -- connection open, message arrival, reconnection after close.

A minimal mock:

```typescript
// libs/shared/realtime/src/testing/mock-websocket.ts
export class MockWebSocket {
  static instances: MockWebSocket[] = [];

  url: string;
  readyState = 0;
  onopen?: () => void;
  onmessage?: (event: MessageEvent) => void;
  onclose?: () => void;
  onerror?: (event: Event) => void;
  sent: string[] = [];

  constructor(url: string) {
    this.url = url;
    MockWebSocket.instances.push(this);
    queueMicrotask(() => {
      this.readyState = 1;
      this.onopen?.();
    });
  }

  send(data: string): void { this.sent.push(data); }
  close(): void { this.readyState = 3; this.onclose?.(); }

  simulateMessage(data: unknown): void {
    this.onmessage?.(new MessageEvent('message', { data: JSON.stringify(data) }));
  }
}
```

With the mock in place, Vitest substitutes it for the global `WebSocket` and drives the scenarios:

```typescript
// libs/shared/realtime/src/lib/price-stream.service.spec.ts
import { vi, describe, it, expect, beforeEach } from 'vitest';
import { TestBed } from '@angular/core/testing';
import { MockWebSocket } from '../testing/mock-websocket';
import { PriceStreamService } from './price-stream.service';

describe('PriceStreamService', () => {
  beforeEach(() => {
    MockWebSocket.instances = [];
    vi.stubGlobal('WebSocket', MockWebSocket);
  });

  it('routes price messages by ticker', async () => {
    const service = TestBed.inject(PriceStreamService);
    service.connect();
    const received: number[] = [];
    service.prices('AAPL').subscribe((p) => received.push(p.price));

    await Promise.resolve();
    MockWebSocket.instances[0].simulateMessage({ kind: 'price', ticker: 'AAPL', price: 150, ts: 1 });
    MockWebSocket.instances[0].simulateMessage({ kind: 'price', ticker: 'MSFT', price: 300, ts: 2 });

    expect(received).toEqual([150]);
  });

  it('reconnects with backoff on close', async () => {
    vi.useFakeTimers();
    const service = TestBed.inject(PriceStreamService);
    service.connect();
    await Promise.resolve();

    MockWebSocket.instances[0].close();
    expect(service.connectionState()).toBe('disconnected');

    vi.advanceTimersByTime(1_000);
    expect(MockWebSocket.instances.length).toBe(2);
  });
});
```

Two patterns appear repeatedly. First, every scenario begins by clearing the `instances` array so one test does not see sockets left by another. Second, fake timers from Vitest let reconnection tests execute in microseconds instead of waiting real backoff windows -- essential for keeping the suite fast.

For integration tests, the companion codebase's `mock-api/ws-server.mjs` is a tiny Node WebSocket server that emits scripted messages on demand. Playwright specs from [Chapter 19](ch37-e2e-playwright.md) launch it alongside the mock HTTP API, letting end-to-end tests exercise the full stack against a real network socket without hitting a production server.

---

## Production Concerns

Shipping a WebSocket-backed feature is a different operational animal than shipping HTTP endpoints. A handful of concerns turn from "interesting" into "this caused a 3 AM page" on the first outage.

### Authentication

A WebSocket handshake is a regular HTTP upgrade request. It can carry cookies, and authenticated cookies work fine if your backend supports same-origin WebSockets. For cross-origin connections, three options exist:

- **Token in the first message.** The client connects unauthenticated and sends `{ kind: 'auth', token: '...' }` as the first frame. The server rejects any other message until the token validates. This is what `PriceStreamService.onOpen` does above. The upside is simplicity; the downside is that `Origin` / CSRF protections on the handshake itself are the only defense before the auth message arrives.
- **Signed short-lived URL.** The backend issues a single-use connection token via an HTTP API, the client includes it as a query parameter (`wss://example.com/prices?token=...`), and the server validates the token during the upgrade. This works across CDNs that do not pass the `Authorization` header and avoids storing the token in a cookie.
- **Header-based auth.** Some non-browser clients can set `Authorization` on the upgrade request. Browsers cannot -- the native `WebSocket` constructor has no options for custom headers. For browser clients, use one of the two options above.

Pair any of these with JWT short TTLs (minutes, not hours) so that a leaked connection token does not grant indefinite access. Rotation -- having the client reconnect and re-authenticate before the token expires -- is the responsibility of the service layer and ties naturally into the reconnect logic already in place. See [Chapter 27](ch17-auth-patterns.md) for how token lifecycles are managed across the wider application.

### Cross-origin, proxies, CDNs

Three infrastructure details bite new deployments repeatedly.

First, WebSockets and CORS behave differently than HTTP. The `Origin` header is sent on the upgrade request, but browsers do *not* enforce same-origin policy on WebSocket connections -- any page can connect to any `wss://` URL. The server is the only line of defense; check `Origin` and reject foreign connections explicitly.

Second, reverse proxies need configuration. NGINX requires `proxy_http_version 1.1`, `Upgrade`, and `Connection` headers to pass through the upgrade handshake. Cloudflare, CloudFront, and other CDNs support WebSockets but often require enabling the feature explicitly in the configuration console. Idle timeouts on proxies typically default to 60-120 seconds; increase them to match the heartbeat cadence, or the proxy will close sockets that are otherwise healthy.

Third, load balancers that use round-robin (rather than sticky sessions) will scatter a reconnecting client across instances. If the server holds per-connection state in memory, that state is lost on reconnect and the client sees stale data. The fix is either sticky sessions at the LB layer or externalizing connection state to Redis or a similar store. This is an architecture decision; make it before scaling to multiple backend instances, not after.

### Scaling server-side

A single WebSocket server instance holds roughly 10,000-50,000 concurrent connections comfortably, depending on message rate. Beyond that, horizontal scaling requires a pub/sub layer -- Redis, Kafka, NATS -- to fan messages out to whichever server instance holds the target connection. The frontend code does not change; it is a server-side concern. But the protocol envelope with a stable schema (from the protocol-design section above) becomes essential when multiple server versions coexist during a rolling deployment.

---

## Summary

Real-time data is a different problem than request/response. HTTP is excellent for the latter and inefficient for the former; picking the right transport and wrapping it cleanly is the difference between a dashboard that feels alive and one that frustrates users.

- **Transport choice.** Polling for low-frequency updates, long polling for legacy environments, SSE for one-way feeds, WebSockets when the client also needs to send. The FinancialApp uses both SSE (notifications) and WebSockets (prices, trades).
- **RxJS with `webSocketSubject`.** A cold observable that opens on subscribe, sends with `.next()`, receives by subscribing, and composes with every operator from [Chapter 10](ch08-rxjs-deep-dive.md).
- **Reconnection with backoff.** `retry({ delay: (_, n) => timer(Math.min(30_000, 1_000 * 2 ** n)) })` plus jitter and integration with `online`/`offline` events covers the realistic failure modes.
- **Connection state signal.** Three states -- `disconnected`, `connecting`, `connected` -- drive both UI indicators and internal logic, and integrate naturally with the rest of the signal graph.
- **Signal interop.** `toSignal()` for single streams, `rxResource({ stream: ... })` for resource-shaped consumption, following the patterns from [Chapter 3](ch03-reactive-signals.md).
- **Backpressure.** `auditTime(16)` caps rendering at 60 fps while always surfacing the freshest value. `scan` accumulates state across the stream without unbounded memory growth.
- **Server-Sent Events.** The simpler option when the client only listens. `EventSource` with `NgZone.runOutsideAngular()` for zone-based applications; automatic reconnection and `Last-Event-ID` resumption out of the box.
- **Protocol design.** Envelopes with `kind` discriminants and version tags, plus heartbeats to detect half-open sockets, are the difference between a protocol that evolves gracefully and one that breaks on the first backward-incompatible change.
- **Testing.** `vi.stubGlobal('WebSocket', MockWebSocket)` substitutes a controllable fake; fake timers let reconnection tests run in microseconds. The companion `mock-api/ws-server.mjs` provides end-to-end integration against a real socket.
- **Production concerns.** Auth via first-message token or signed URL, `Origin` checking on the server side, proxy and CDN configuration for the upgrade handshake, and sticky sessions or externalized state for horizontal scaling.

Real-time features feel magical to users when they work and ruin the product when they do not. The RxJS primitives, the signal interop surface, and the reconnection patterns in this chapter are the scaffolding that makes them reliable -- not clever, not minimal, but boring in the best way. A dashboard that reconnects without a spinner, re-authenticates without a logout, and throttles prices to the eye's refresh rate is one that users stop noticing. That is the goal.

In [Chapter 48](ch34-data-visualization.md), we will render these streams into charts and sparklines, handling the cross-cutting concerns of WebGL rendering, canvas throttling, and accessibility for data-dense visualizations.
