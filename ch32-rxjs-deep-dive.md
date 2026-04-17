# Chapter 32: RxJS Deep Dive

> **Extends:** [Chapter 3](ch03-reactive-signals.md) covered the signal/RxJS boundary. This chapter goes deeper into RxJS itself -- when to reach for it, operator composition, subjects, and marble testing.

Signals are the primary reactivity primitive in modern Angular, and most of the FinancialApp's state, derivation, and view binding lives on the signal side. But signals are snapshots -- they describe *what a value is right now*. The moment your code needs to describe *a sequence of events over time* -- HTTP responses that may never arrive, WebSocket pushes for live prices, debounced keystrokes feeding a typeahead, a chain of dependent server calls -- you are in RxJS territory. This chapter covers the operators that actually come up in production, the subject variants and their trade-offs, the cold/hot distinction, schedulers, the signal-interop surface beyond the basics, marble testing with Vitest, and the memory-leak pitfalls that dominate bug reports in long-lived Angular apps.

> **Companion code:** Advanced RxJS examples in `financial-app/libs/shared/data-access/src/rx-patterns.ts` and marble tests in its spec file.

---

## When RxJS, When Signals

The first skill in a signal-first codebase is recognizing which tool the problem wants:

| Problem shape                                                | Choice                  |
| ------------------------------------------------------------ | ----------------------- |
| Current value, derived values, template bindings             | Signals                 |
| Single HTTP call, no cancellation concerns                   | `httpResource`          |
| HTTP with debounce, cancellation, retry                      | `rxResource` or `HttpClient` + operators |
| Streams where every emission matters (WebSocket, SSE, keystrokes) | RxJS               |
| Event orchestration across multiple sources                  | RxJS                    |
| Timer-based flows (`interval`, `timer`, `delay`)             | RxJS                    |
| Forms `valueChanges` / `statusChanges`                       | RxJS -> `toSignal`      |
| NgRx Signal Store effects (`rxMethod`, `rxMutation`)         | RxJS internally, signals externally |

A useful mental rule: if your value has a meaningful *identity at rest* (balance, selected account, filter text), it is a signal. If your value is a *happening in motion* (the user just typed, the WebSocket just pushed), it is an observable. The FinancialApp's WebSocket layer in [Chapter 33](ch33-websockets.md) is the archetype of a pure-RxJS problem; only the latest processed transaction surfaces as a signal via `toSignal`.

---

## Operator Catalog by Use Case

RxJS ships well over a hundred operators; most codebases use fewer than twenty. The subset below covers almost every situation you will encounter, grouped by job.

### Transformation: `map`, `scan`

`map` applies a function to each emission:

```typescript
// libs/shared/data-access/src/account.service.ts
getAccountNames(): Observable<string[]> {
  return this.http.get<Account[]>('/api/accounts').pipe(
    map(accounts => accounts.map(a => a.name).sort()),
  );
}
```

The older `pluck('name')` operator was removed in RxJS 7. Use `map(x => x.name)` instead -- same code size, better type inference.

`scan` is `reduce` for streams: it accumulates across emissions and emits the running total on every step. The FinancialApp uses it for a live running balance:

```typescript
// libs/shared/data-access/src/rx-patterns.ts
runningBalance(transactions$: Observable<Transaction>): Observable<number> {
  return transactions$.pipe(
    scan((balance, tx) => tx.type === 'credit' ? balance + tx.amount : balance - tx.amount, 0),
  );
}
```

Unlike `reduce`, which emits once on `complete`, `scan` emits after every input.

### Filtering: `filter`, `distinctUntilChanged`, `debounceTime`

`filter` drops emissions that fail a predicate. `distinctUntilChanged` drops emissions equal to the previous one. `debounceTime` emits only after a period of silence. Together they are the front-end for every typeahead:

```typescript
// domains/accounts/account-typeahead.component.ts
readonly query = signal('');

private results$ = toObservable(this.query).pipe(
  filter(q => q.length >= 2),
  debounceTime(250),
  distinctUntilChanged(),
  switchMap(q => this.accountService.search(q)),
);

readonly results = toSignal(this.results$, { initialValue: [] });
```

Order matters: debounce *before* distinctUntilChanged, because distinctUntilChanged compares the last *emitted* value. For structural equality, pass a comparator: `distinctUntilChanged((a, b) => a.id === b.id)`.

### Combining: `combineLatest`, `merge`, `concat`, `forkJoin`, `withLatestFrom`

These five operators all combine streams but answer different questions.

**`combineLatest`** emits whenever *any* source emits, giving the latest from all of them; waits for every source to emit at least once:

```typescript
// domains/accounts/account-summary.component.ts
summary$ = combineLatest([
  this.accountService.current$,
  this.transactionService.recent$,
  this.portfolioService.totals$,
]).pipe(map(([account, transactions, totals]) => ({ account, transactions, totals })));
```

**`merge`** interleaves emissions from every source with no combining -- use it for event streams of the same shape:

```typescript
// libs/shared/data-access/src/rx-patterns.ts
allTransactionEvents$ = merge(
  this.add$.pipe(map(tx => ({ kind: 'add' as const, tx }))),
  this.remove$.pipe(map(id => ({ kind: 'remove' as const, id }))),
);
```

**`concat`** plays sources one after another -- the next starts only after the previous completes. Useful for sequencing initial-data loads.

**`forkJoin`** waits for every source to complete, then emits a single bundle of the last values. RxJS's cousin of `Promise.all`:

```typescript
// domains/dashboard/dashboard-loader.ts
dashboard$ = forkJoin({
  accounts: this.accountService.loadAll(),
  recent:   this.transactionService.loadRecent(),
  watchlist: this.watchlistService.loadAll(),
});
```

`forkJoin` completes only when every source completes -- use it for one-shot HTTP calls, never for `Subject` streams that never terminate.

**`withLatestFrom`** lets one stream read the latest value of another without driving emissions:

```typescript
// libs/shared/data-access/src/rx-patterns.ts
selectedAccount$ = this.accountService.all$.pipe(
  withLatestFrom(this.uiState.selectedAccountId$),
  map(([accounts, id]) => accounts.find(a => a.id === id)),
);
```

The first stream is the driver; the second provides context without triggering emissions.

### Error Handling: `catchError`, `retry`

`catchError` intercepts an error and substitutes a recovery observable:

```typescript
// libs/shared/data-access/src/portfolio.service.ts
loadPortfolio(id: PortfolioId): Observable<Portfolio | null> {
  return this.http.get<Portfolio>(`/api/portfolios/${id}`).pipe(
    catchError(err => { this.logger.error('Portfolio load failed', err); return of(null); }),
  );
}
```

The subtlety: `catchError` *swallows* the error and produces a new successful stream. To log and propagate, rethrow inside the handler.

`retry` re-subscribes on error. The modern signature takes a config object:

```typescript
// libs/shared/data-access/src/transaction.service.ts
loadTransactions(): Observable<Transaction[]> {
  return this.http.get<Transaction[]>('/api/transactions').pipe(
    retry({ count: 3, delay: (err, i) => timer(1000 * Math.pow(2, i)) }),
    catchError(() => of([])),
  );
}
```

That `delay` function implements exponential backoff: 1s, 2s, 4s. The older `retryWhen` operator was deprecated in favor of `retry({ delay })`. Layer `retry` *before* `catchError` -- "try four times total, then fall back to an empty array." The other order would swallow the first error into a success, and retry would never trigger.

### Flattening: `mergeMap` vs `switchMap` vs `concatMap` vs `exhaustMap`

This is the section that trips up every RxJS learner. All four operators solve the same problem -- each outer emission produces an inner observable whose values flatten into the output -- and differ only in what happens when a new outer emission arrives while a previous inner is still in flight.

**`switchMap`** cancels the previous inner observable and switches to the new one. Latest wins. Right choice for typeahead, route changes, and any "only the current input matters" scenario:

```typescript
// domains/accounts/account-search.component.ts
results$ = this.query$.pipe(
  debounceTime(250),
  switchMap(q => this.accountService.search(q)),
);
```

If the user types "appl" then "apple", the "appl" HTTP request is cancelled; only "apple" completes.

**`mergeMap`** runs inner observables concurrently. All results arrive; none are cancelled. Right choice when each outer emission represents independent work whose order does not matter:

```typescript
// libs/shared/data-access/src/rx-patterns.ts
loadAllAccountBalances(ids: AccountId[]): Observable<Balance> {
  return from(ids).pipe(mergeMap(id => this.accountService.getBalance(id), 5));
}
```

The second argument (`5`) caps concurrency. Without it, `mergeMap` would fire all requests simultaneously -- a denial-of-service attack against your own server for large inputs. Always cap concurrency for unbounded inputs.

**`concatMap`** queues inner observables, starting the next only when the current completes. Right choice when order matters:

```typescript
// libs/shared/data-access/src/rx-patterns.ts
submissions$ = this.pendingTransactions$.pipe(
  concatMap(tx => this.transactionService.submit(tx).pipe(
    catchError(err => of({ kind: 'failed' as const, tx, err })),
  )),
);
```

`concatMap` ensures the server receives three rapid submissions in order. `mergeMap` would fire in parallel; `switchMap` would cancel the first two. Neither is what a ledger wants.

**`exhaustMap`** ignores new outer emissions while an inner observable is active. Current in-flight work wins; new requests are dropped. The use case is rate-limiting actions that should not double-fire:

```typescript
// domains/accounts/login.component.ts
private loginClicks$ = new Subject<Credentials>();

constructor() {
  this.loginClicks$.pipe(
    exhaustMap(creds => this.authService.login(creds)),
    takeUntilDestroyed(),
  ).subscribe(result => this.handleLogin(result));
}
```

Three rapid clicks fire only the first request. Decision summary:

| Concern                                   | Operator     |
| ----------------------------------------- | ------------ |
| Only the latest matters                   | `switchMap`  |
| All matter, order does not                | `mergeMap`   |
| All matter, in order                      | `concatMap`  |
| Ignore new while one is in flight         | `exhaustMap` |

When in doubt, default to `switchMap` for reads and `concatMap` for writes. That covers perhaps 80% of real cases.

---

## Subjects

A Subject is both an Observable and an Observer -- it bridges imperative code into reactive pipelines. Four variants exist; they are not interchangeable.

**`Subject<T>`** emits only to current subscribers. Values emitted before anyone subscribes are lost. Use it for fire-and-forget events:

```typescript
// libs/shared/data-access/src/rx-patterns.ts
private userActions$ = new Subject<UserAction>();

emit(action: UserAction): void { this.userActions$.next(action); }
actions$(): Observable<UserAction> { return this.userActions$.asObservable(); }
```

The `asObservable()` call hides `next`/`error`/`complete` from the public API.

**`BehaviorSubject<T>`** is seeded with an initial value and replays the *current* value to every new subscriber. The closest RxJS analog to a writable signal:

```typescript
// libs/shared/data-access/src/ui-state.service.ts
private selectedAccountId$ = new BehaviorSubject<AccountId | null>(null);
current$ = this.selectedAccountId$.asObservable();
select(id: AccountId): void { this.selectedAccountId$.next(id); }
```

In greenfield code, write that as a signal. In existing code, expose it to templates via `toSignal(this.current$)` without rewriting the internals.

**`ReplaySubject<T>`** buffers the last N emissions and replays them to every new subscriber. The buffer can be bounded by count, time, or both:

```typescript
// libs/shared/data-access/src/error-log.service.ts
private errors$ = new ReplaySubject<AppError>(50, 60_000);
recent$ = this.errors$.asObservable();
```

Useful for diagnostic buffers and recent-activity panels. Watch the memory implications of a large unbounded buffer.

**`AsyncSubject<T>`** emits only its *final* value, on `complete()`. Rarely useful in modern code because `firstValueFrom` captures the same semantics more directly.

| Need                                            | Use               |
| ----------------------------------------------- | ----------------- |
| Broadcast events; late subscribers skip history | `Subject`         |
| Stateful value with initial and latest          | `BehaviorSubject` |
| Buffered history for late subscribers           | `ReplaySubject`   |
| Single final value on completion                | `AsyncSubject` (rare) |

For pure state, prefer signals. For event broadcasting, prefer `Subject`. `ReplaySubject` earns its place when history genuinely matters.

---

## Cold vs Hot Observables and Multicasting

The cold/hot distinction is the single concept most likely to produce a subtle bug. A **cold** observable produces fresh work for each subscriber -- subscribe twice to `http.get('/api/accounts')` and you get two HTTP requests. A **hot** observable shares a single source across subscribers -- subscribe twice to a WebSocket stream and both receive the same messages. Most observables are cold by default, including every `HttpClient` method. The fix when you want to share results is **multicasting**, and `shareReplay` is the operator you reach for most:

```typescript
// libs/shared/data-access/src/config.service.ts
@Injectable({ providedIn: 'root' })
export class ConfigService {
  readonly config$ = this.http.get<AppConfig>('/api/config').pipe(
    shareReplay({ bufferSize: 1, refCount: true }),
  );
}
```

Every consumer gets the same cached config. The first subscriber triggers the HTTP request; subsequent subscribers receive the cached response. `refCount: true` is not optional: without it (the default!), the source subscription stays alive forever -- a classic memory leak.

`share()` is the non-replaying variant; new subscribers see only future emissions. Useful for hot streams (WebSocket, Router events) that do not need history. For full manual control, `connectable()` creates a multicast observable you start with `.connect()`, but you rarely need it in application code.

Heuristic: if two places subscribe to the same observable and you did not intend two HTTP requests, add `shareReplay({ bufferSize: 1, refCount: true })`.

---

## Schedulers

Schedulers control *when* and *on what execution context* RxJS work runs:

| Scheduler                    | Runs on                                    | Use when                                                    |
| ---------------------------- | ------------------------------------------ | ----------------------------------------------------------- |
| `asyncScheduler`             | `setTimeout(0)`                            | Default for time-based operators                            |
| `queueScheduler`             | Synchronously on the current call stack    | Tests where you want synchronous behaviour                  |
| `asapScheduler`              | Microtask queue                            | Faster than async, defers past the current stack            |
| `animationFrameScheduler`    | `requestAnimationFrame`                    | Animations and UI updates tied to display refresh           |

Ninety percent of application code never specifies a scheduler and that is correct. Three situations where an override pays off:

**Animations driven by an observable.** A stream of scroll positions or price ticks feeds `animationFrameScheduler` so updates sync with browser paints:

```typescript
// libs/shared/ui/src/live-chart/live-chart.component.ts
pricePoints$.pipe(
  throttleTime(0, animationFrameScheduler),
).subscribe(point => this.drawPoint(point));
```

That collapses a high-frequency source (a WebSocket emitting hundreds of ticks per second) to one update per frame -- no wasted renders.

**Deterministic tests.** `TestScheduler` (below) replaces real time with virtual time; production code using `asyncScheduler` behaves identically because TestScheduler patches the default.

**Breaking synchronous recursion.** A `Subject` whose emissions feed back into itself can stack-overflow; `observeOn(asapScheduler)` lets the current stack unwind first.

If you reach for a scheduler to "fix" a timing issue, stop. Most timing fixes are really correctness fixes: the wrong flattening operator, a missing `shareReplay`, or a nested subscribe.

---

## Signal Interop Beyond the Basics

[Chapter 3](ch03-reactive-signals.md) introduced `toSignal`, `toObservable`, and `rxResource`. The `@angular/core/rxjs-interop` package ships more utilities worth knowing.

### `rxResource` for RxJS-Native Resources

`rxResource` is the RxJS mirror of `httpResource`. Where `httpResource` hides the HTTP details behind a URL template, `rxResource` lets you compose the full operator pipeline and exposes the result as a resource with `value()`, `status()`, `error()`, and `reload()`:

```typescript
// libs/shared/data-access/src/rx-patterns.ts
transactions = rxResource({
  request: () => ({ accountId: this.accountId(), range: this.range() }),
  stream: ({ request }) => this.http.get<Transaction[]>('/api/transactions', { params: request }).pipe(
    retry({ count: 2, delay: 500 }),
    map(list => list.sort((a, b) => b.date.localeCompare(a.date))),
  ),
});
```

The `request` function tracks signal dependencies; changing `accountId()` re-invokes `stream` with `switchMap` semantics -- any in-flight request is cancelled.

### `outputFromObservable` and `outputToObservable`

Component `output()`s are the modern replacement for `EventEmitter`, but some integrations want to treat them as observables. Two helpers handle each direction:

```typescript
// libs/shared/ui/src/transaction-form.component.ts
import { outputFromObservable, outputToObservable } from '@angular/core/rxjs-interop';

export class TransactionFormComponent {
  private readonly saves$ = new Subject<Transaction>();
  readonly saved = outputFromObservable(this.saves$);

  readonly formRef = viewChild.required(ChildForm);
  constructor() {
    outputToObservable(this.formRef().saved).pipe(
      concatMap(tx => this.transactionService.submit(tx)),
      takeUntilDestroyed(),
    ).subscribe();
  }
}
```

`outputFromObservable` turns an observable into a component output. `outputToObservable` does the reverse -- the parent can apply operators to a child's output events instead of handling them imperatively.

### `pendingUntilEvent`

SSR and testing both care about when the application is "done" with async work. `pendingUntilEvent` registers a pending task with Angular's `PendingTasks` service until the observable emits or completes:

```typescript
// libs/shared/data-access/src/rx-patterns.ts
import { pendingUntilEvent } from '@angular/core/rxjs-interop';

readonly onboardingData$ = this.http.get<OnboardingPayload>('/api/onboarding').pipe(
  pendingUntilEvent(),
  shareReplay({ bufferSize: 1, refCount: true }),
);
```

SSR waits for every pending task to clear before serializing the DOM -- the transfer-cache mechanism from [Chapter 17](ch17-defer-ssr-hydration.md) depends on this. Under zoneless tests, `ComponentFixture.whenStable()` respects the same API. `HttpClient` already registers its requests, so you rarely add `pendingUntilEvent` directly; custom observables (WebSocket handshakes, IndexedDB reads) often need it.

### Forms `valueChanges` to Signals

The forms package -- even Signal Forms from [Chapter 6](ch06-signal-forms.md) -- exposes `valueChanges` as observables. Bridge at the component boundary:

```typescript
// domains/transactions/transaction-filter.component.ts
private readonly category$ = this.filterForm.category.valueChanges.pipe(
  debounceTime(300),
  distinctUntilChanged(),
);

readonly category = toSignal(this.category$, { initialValue: '' });
```

The observable pipeline does the work that signals cannot express cleanly; `toSignal` terminates it into the signal world for the template.

---

## Testing Observables with Marble Syntax

Time-based observable pipelines are the hardest Angular code to test. Real timers are slow and flaky; `fakeAsync` couples tests to Zone.js and does not survive in zoneless code. The modern answer is RxJS's `TestScheduler` with marble syntax.

A marble diagram describes an observable timeline: `-` is one frame of nothing, `a`/`b`/`c` are emitted values (supplied separately), `|` is completion, `#` is an error, and `(abc)` groups emissions in the same frame.

### Setup with Vitest

```typescript
// libs/shared/data-access/src/rx-patterns.spec.ts
import { TestScheduler } from 'rxjs/testing';
import { beforeEach, describe, expect, it } from 'vitest';

describe('runningBalance', () => {
  let scheduler: TestScheduler;

  beforeEach(() => {
    scheduler = new TestScheduler((actual, expected) => expect(actual).toEqual(expected));
  });

  it('accumulates credits and debits', () => {
    scheduler.run(({ cold, expectObservable }) => {
      const input = cold<Transaction>('-a-b-c|', {
        a: { type: 'credit', amount: 100 } as Transaction,
        b: { type: 'debit',  amount: 30  } as Transaction,
        c: { type: 'credit', amount: 20  } as Transaction,
      });

      expectObservable(runningBalance(input)).toBe('-x-y-z|', { x: 100, y: 70, z: 90 });
    });
  });
});
```

Inside `scheduler.run`, `asyncScheduler`, `asapScheduler`, and `animationFrameScheduler` are all patched to virtual time; `debounceTime(250)` advances 250 virtual frames instantly. A `debounceTime` + `switchMap` pipeline is exactly where marble tests excel:

```typescript
// libs/shared/data-access/src/rx-patterns.spec.ts
it('debounces query and switches to the latest result', () => {
  scheduler.run(({ cold, expectObservable }) => {
    const query = cold<string>('a-b-c---|', { a: 'ap', b: 'app', c: 'appl' });
    const search = (q: string) => cold<string[]>('--r|', { r: [`${q}-result`] });

    const results = query.pipe(
      debounceTime(3),
      switchMap(q => search(q)),
    );

    expectObservable(results).toBe('--------(r|)', { r: ['appl-result'] });
  });
});
```

Virtual time lets you write tests that would take multiple real seconds in under a millisecond, with no timer mocks. Marble tests are not a replacement for integration tests -- they are the right tool for operator-pipeline behavior in isolation. Keep them tight, one chain per test, and reserve them for pipelines complex enough that a hand-wired test would obscure the intent.

---

## Common Pitfalls

### Memory Leaks from Long-Lived Subscriptions

A component that subscribes without unsubscribing leaks the subscription and everything it closes over. The modern fix is `takeUntilDestroyed` from `@angular/core/rxjs-interop`:

```typescript
// domains/accounts/account-list.component.ts
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';

export class AccountListComponent {
  constructor() {
    this.accountService.all$.pipe(
      takeUntilDestroyed(),
    ).subscribe(list => this.handleList(list));
  }
}
```

`takeUntilDestroyed()` ties the observable's lifetime to the current injection context's `DestroyRef`. When the component destroys, the observable completes and every subscription releases -- no manual unsubscribe, no `Subject<void>` destroy flag.

The rule: every `.subscribe()` inside a component or service with a `DestroyRef` should end with `takeUntilDestroyed()`, unless the subscription is explicitly short-lived. Outside an injection context, pass the `DestroyRef` explicitly: `takeUntilDestroyed(destroyRef)`.

### Unterminated Subjects

A `Subject` exposed to consumers without termination never completes -- downstream `takeUntil` and `last()` never fire, and lingering subscriptions accumulate. Complete every Subject you own when its owner is destroyed:

```typescript
// libs/shared/data-access/src/ui-state.service.ts
export class UiStateService {
  private destroyRef = inject(DestroyRef);
  private selectedAccountId$ = new BehaviorSubject<AccountId | null>(null);

  constructor() {
    this.destroyRef.onDestroy(() => this.selectedAccountId$.complete());
  }
}
```

Inside a component's pipeline, `takeUntilDestroyed` on the subscription is sufficient. Any Subject exposed publicly -- through a root-scoped service, a store, or a library API -- should `complete()` explicitly on destruction.

### Subscribe Inside Subscribe

```typescript
this.accountService.current$.subscribe(account => {
  this.transactionService.loadFor(account.id).subscribe(transactions => this.display(transactions));
});
```

This is a bug. The inner subscription has no lifecycle tied to anything; it leaks on every emission of the outer. The flattening operators exist precisely for this problem:

```typescript
this.accountService.current$.pipe(
  switchMap(account => this.transactionService.loadFor(account.id)),
  takeUntilDestroyed(),
).subscribe(transactions => this.display(transactions));
```

One subscription, cancelled when the component dies, and any in-flight transaction load cancelled when the account changes. Nested subscribes should fail code review.

### Async Pipe Plus Manual Subscription

The `async` pipe auto-subscribes and unsubscribes. Subscribing manually to the same observable doubles the work -- especially for cold observables. Pick one place per observable. In signal-first templates, prefer `toSignal(observable$)` in the component class and render the signal directly; `async` is still supported but no longer idiomatic.

---

## Summary

Signals are the default reactivity primitive in modern Angular, but RxJS is not legacy -- it is the right tool for streams, time, event orchestration, and anything where snapshots lose information.

- **Reach for RxJS** for HTTP with cancellation, WebSocket and SSE streams, event orchestration, timers, and forms' `valueChanges`. Everything else lives on the signal side.
- **Flattening operators** are the highest-leverage category: `switchMap` for reads, `mergeMap` (capped) for independent concurrent work, `concatMap` for ordered writes, `exhaustMap` for ignore-while-busy actions.
- **Subjects** reduce in practice to `Subject` for broadcast and `BehaviorSubject` for stateful values (a legacy cousin of `signal`). `ReplaySubject` earns its place when history genuinely matters.
- **Multicasting** with `shareReplay({ bufferSize: 1, refCount: true })` is non-negotiable when two subscribers must share work. `refCount: true` is not optional -- the default leaks.
- **Schedulers** defaults are correct 95% of the time; override to `animationFrameScheduler` for render-synchronized streams and `queueScheduler` for deterministic tests.
- **Signal interop** extends beyond `toSignal`/`toObservable`: `rxResource` for RxJS-native resources, `outputFromObservable`/`outputToObservable` for bridging component outputs, `pendingUntilEvent` for SSR and test stability.
- **Marble testing** with `TestScheduler` produces deterministic, fast tests in zoneless code. Virtual time makes `debounceTime(250)` cost zero wall-clock milliseconds.
- **Pitfalls** cluster around lifecycle: `takeUntilDestroyed()` on every component subscription, `complete()` on every publicly-exposed Subject, flattening operators instead of nested subscribes.

RxJS rewards investment. The operator vocabulary is compact once grouped by use case, the subject zoo collapses to two variants you actually reach for, and the testing story is better than most frameworks offer. [Chapter 33](ch33-websockets.md) applies these patterns to a domain where RxJS is not optional -- live market data, portfolio updates, and advisor chat streams delivered over WebSocket.
