# Chapter 9: State Management with NgRx Signal Store

In [Chapter 5](ch05-state-management.md), we built lightweight signal stores from scratch -- small classes that wrap `signal()` and `computed()` to manage state within a single domain. Those stores are remarkably capable for local state, but as the FinancialApp grows, certain needs recur: entity collections that must be normalized, computed state that should be reusable across domains, developer tooling for inspecting state at runtime, and a vocabulary for asynchronous mutations that teams can standardize around.

NgRx Signal Store addresses exactly these needs. It is not a replacement for lightweight stores -- it is a graduation path for stores whose complexity warrants the extra structure. The library composes small, reusable features (`withState`, `withComputed`, `withMethods`, `withEntities`, `withDevtools`) into a single store definition, while remaining fully signal-native and tree-shakeable.

> **Stability notice:** APIs in this chapter, particularly the mutations and event APIs from NgRx Toolkit, may have changed since publication. The NgRx team iterates quickly, and experimental APIs carry an explicit `@ngrx/toolkit` import path to signal their pre-stable status. Always consult the official NgRx documentation for the latest signatures.

---

## A First SignalStore

### Creating a Signal Store

A `signalStore()` call produces a fully injectable Angular service. Unlike class-based stores, you do not write a class -- you compose features:

```typescript
// financial-app/apps/.../transactions/transaction.store.ts
import { signalStore } from '@ngrx/signals';

export const TransactionStore = signalStore(
  // features compose here
);
```

The result, `TransactionStore`, is a provider token. Angular's injector treats it like any other service. The difference is in how you define its interior: through composition rather than inheritance.

### Defining State

The `withState()` feature declares the shape and initial values of your store's state. Each property becomes a signal on the store instance:

```typescript
import { signalStore, withState } from '@ngrx/signals';

interface TransactionState {
  transactions: Transaction[];
  selectedId: string | null;
  loading: boolean;
  error: string | null;
}

const initialState: TransactionState = {
  transactions: [],
  selectedId: null,
  loading: false,
  error: null,
};

export const TransactionStore = signalStore(
  withState(initialState),
);
```

After injection, `store.transactions()` returns the current array, `store.loading()` returns the boolean, and so on. Each property is a read-only signal -- you cannot assign to it directly. Mutations flow through explicit methods, which we will define shortly.

### Providing Resources

NgRx Signal Store integrates with Angular's `resource()` API through the `withResource()` feature, letting you declare async data dependencies that automatically track their loading state:

```typescript
import { withResource } from '@ngrx/signals';

export const TransactionStore = signalStore(
  withState({ accountId: null as string | null }),
  withResource({
    request: (store) => ({ accountId: store.accountId() }),
    loader: ({ request }) =>
      inject(TransactionApi).getByAccount(request.accountId),
  }),
);
```

The resource re-fetches whenever `accountId` changes, and exposes `.value()`, `.isLoading()`, `.error()`, and `.status()` signals on the store. This replaces the manual `loading`/`error` flags we defined earlier with a declarative alternative.

### Providing Computed Signals

The `withComputed()` feature defines derived state. Computeds recalculate only when their dependencies change, and they are memoized -- reading them multiple times in the same change detection cycle costs nothing:

```typescript
import { signalStore, withState, withComputed } from '@ngrx/signals';
import { computed } from '@angular/core';

export const TransactionStore = signalStore(
  withState(initialState),
  withComputed((store) => ({
    selectedTransaction: computed(() =>
      store.transactions().find((t) => t.id === store.selectedId())
    ),
    pendingCount: computed(() =>
      store.transactions().filter((t) => t.status === 'pending').length
    ),
    hasError: computed(() => store.error() !== null),
  })),
);
```

The factory function receives the accumulated store -- meaning `withComputed()` can reference state defined by `withState()`, or by any feature declared before it. Feature ordering matters.

### Providing Methods

The `withMethods()` feature attaches behavior. Methods have access to the accumulated store and can inject services:

```typescript
import { signalStore, withState, withComputed, withMethods } from '@ngrx/signals';
import { patchState } from '@ngrx/signals';
import { inject } from '@angular/core';

export const TransactionStore = signalStore(
  withState(initialState),
  withComputed((store) => ({
    selectedTransaction: computed(() =>
      store.transactions().find((t) => t.id === store.selectedId())
    ),
  })),
  withMethods((store) => {
    const api = inject(TransactionApi);

    return {
      selectTransaction(id: string): void {
        patchState(store, { selectedId: id });
      },
      async loadTransactions(accountId: string): Promise<void> {
        patchState(store, { loading: true, error: null });
        try {
          const transactions = await api.getByAccount(accountId);
          patchState(store, { transactions, loading: false });
        } catch (e) {
          patchState(store, { loading: false, error: (e as Error).message });
        }
      },
    };
  }),
);
```

`patchState()` is the only way to modify state. It performs a shallow merge, similar to React's `setState`. You pass the store and a partial state object (or an updater function), and the library applies the patch and notifies subscribers. There is no `mutate` -- immutability is enforced structurally.

### Setting up Hooks

The `withHooks()` feature lets you run logic at specific lifecycle points -- `onInit` when the store is created, and `onDestroy` when it is torn down:

```typescript
import { withHooks } from '@ngrx/signals';

export const TransactionStore = signalStore(
  withState(initialState),
  withMethods(/* ... */),
  withHooks({
    onInit(store) {
      const route = inject(ActivatedRoute);
      const accountId = route.snapshot.paramMap.get('accountId');
      if (accountId) {
        store.loadTransactions(accountId);
      }
    },
    onDestroy(store) {
      console.debug('TransactionStore destroyed');
    },
  }),
);
```

Hooks are particularly useful for triggering initial data loads. When the store is provided at the route level (as recommended in [Chapter 8](ch08-architecture.md)), `onInit` fires when the user navigates to that route, and `onDestroy` fires when they navigate away.

### Consuming the Store

A component consumes the store by injecting it and reading its signals in the template:

```typescript
@Component({
  selector: 'app-transaction-list',
  template: `
    @if (store.loading()) {
      <app-spinner />
    } @else {
      @for (txn of store.transactions(); track txn.id) {
        <app-transaction-row
          [transaction]="txn"
          [selected]="txn.id === store.selectedId()"
          (select)="store.selectTransaction(txn.id)"
        />
      }
    }
    @if (store.hasError()) {
      <app-error-banner [message]="store.error()!" />
    }
  `,
  providers: [TransactionStore],
})
export class TransactionListComponent {
  readonly store = inject(TransactionStore);
}
```

Providing the store at the component level scopes its lifetime to the component. When the component is destroyed -- typically on navigation -- the store is garbage-collected and its `onDestroy` hook fires. For state that must survive navigation, provide the store at a higher level in the injector tree.

---

## Inspecting the Store with the Redux DevTools

### Connecting to the Redux DevTools

Signal stores are reactive and immutable, but they are invisible to the browser's Redux DevTools extension by default. The `withDevtools()` feature bridges this gap:

```typescript
import { withDevtools } from '@ngrx/signals/devtools';

export const TransactionStore = signalStore(
  withDevtools({ name: 'TransactionStore' }),
  withState(initialState),
  withComputed(/* ... */),
  withMethods(/* ... */),
);
```

With this single addition, every `patchState()` call emits an action to the Redux DevTools. You can time-travel through state changes, inspect the current state tree, and diff successive states. The `name` parameter ensures the store is identifiable when multiple stores coexist in the DevTools panel.

Place `withDevtools()` first in the feature chain so that it can observe all state changes introduced by subsequent features.

### Disabling DevTools in Production

DevTools instrumentation adds overhead and exposes internal state to anyone with the browser extension. Disable it for production builds by leveraging Angular's `isDevMode()`:

```typescript
import { isDevMode } from '@angular/core';
import { withDevtools } from '@ngrx/signals/devtools';

const devtoolsFeature = isDevMode()
  ? withDevtools({ name: 'TransactionStore' })
  : [];

export const TransactionStore = signalStore(
  ...devtoolsFeature,
  withState(initialState),
  withMethods(/* ... */),
);
```

Because `signalStore()` accepts a rest parameter of features, you can conditionally spread an empty array when DevTools are unwanted. The tree-shaker will eliminate the `withDevtools` import entirely in production bundles if you guard it properly with build-time checks.

---

## Mutations

NgRx Toolkit introduces mutations as a structured vocabulary for HTTP-bound state changes. Where `withMethods()` gives you free-form functions, mutations encode the lifecycle of an async operation -- pending, success, failure -- into a repeatable pattern.

### Creating Mutations

The `withMutations()` feature from `@ngrx/toolkit` pairs with `httpMutation()` to create mutation definitions that automatically manage call state:

```typescript
import { withMutations, httpMutation } from '@ngrx/toolkit';

export const TransactionStore = signalStore(
  withState(initialState),
  withMutations((store) => {
    const api = inject(TransactionApi);

    return {
      approveTransaction: httpMutation({
        mutate: (id: string) => api.approve(id),
        onSuccess: () => store.reload(),
        onError: (error) =>
          patchState(store, { error: error.message }),
      }),
      rejectTransaction: httpMutation({
        mutate: (id: string, reason: string) =>
          api.reject(id, reason),
        onSuccess: () => store.reload(),
      }),
    };
  }),
);
```

Each `httpMutation()` produces a callable function on the store, along with companion signals: `store.approveTransaction.status()`, `store.approveTransaction.error()`, and `store.approveTransaction.isPending()`. This eliminates the boilerplate of manually tracking `loading` and `error` state for every async operation.

### Consuming Mutations

In templates, mutation status signals drive UI feedback:

```typescript
@Component({
  template: `
    <button
      (click)="store.approveTransaction(txn.id)"
      [disabled]="store.approveTransaction.isPending()"
    >
      @if (store.approveTransaction.isPending()) {
        Approving...
      } @else {
        Approve
      }
    </button>
  `,
})
export class TransactionActionsComponent {
  readonly store = inject(TransactionStore);
  readonly txn = input.required<Transaction>();
}
```

The mutation's `isPending()` signal provides fine-grained loading state per operation, rather than a single `loading` boolean for the entire store.

### Using rxMutation

For mutations that require RxJS operators -- debouncing, retry logic, optimistic updates -- `rxMutation()` accepts a factory that returns an `OperatorFunction`:

```typescript
import { rxMutation } from '@ngrx/toolkit';
import { retry, delay } from 'rxjs';

export const TransactionStore = signalStore(
  withState(initialState),
  withMutations((store) => {
    const api = inject(TransactionApi);

    return {
      settleTransaction: rxMutation({
        mutate: (id: string) =>
          api.settle(id).pipe(retry({ count: 3, delay: 1000 })),
        onSuccess: () => store.reload(),
      }),
    };
  }),
);
```

Use `rxMutation()` when the underlying HTTP call needs operator composition. For straightforward calls, `httpMutation()` keeps the code simpler.

---

## Reactive Methods

### rxMethod

The `rxMethod()` utility bridges signal stores with RxJS. It creates a method that accepts a signal, observable, or static value and pipes it through an operator chain:

```typescript
import { rxMethod } from '@ngrx/signals/rxjs-interop';
import { pipe, switchMap, tap } from 'rxjs';

export const TransactionStore = signalStore(
  withState(initialState),
  withMethods((store) => {
    const api = inject(TransactionApi);

    return {
      loadByAccount: rxMethod<string>(
        pipe(
          tap(() => patchState(store, { loading: true })),
          switchMap((accountId) => api.getByAccount$(accountId)),
          tap((transactions) =>
            patchState(store, { transactions, loading: false })
          ),
        )
      ),
    };
  }),
);
```

A component can invoke `rxMethod` in several ways:

```typescript
// With a static value
store.loadByAccount('acct-42');

// With a signal -- re-executes when the signal changes
const accountId = input.required<string>();
store.loadByAccount(accountId);

// With an observable
store.loadByAccount(route.paramMap.pipe(map((p) => p.get('accountId')!)));
```

This flexibility makes `rxMethod` the tool of choice when a store method must react to changing inputs over time, not just execute once.

### signalMethod

For simpler reactive methods that do not require RxJS operators, `signalMethod()` provides a lighter alternative. It accepts a callback that runs whenever the input signal changes:

```typescript
import { signalMethod } from '@ngrx/signals';

export const TransactionStore = signalStore(
  withState(initialState),
  withMethods((store) => ({
    highlightTransaction: signalMethod<string>((id) => {
      patchState(store, { selectedId: id });
    }),
  })),
);
```

Like `rxMethod`, `signalMethod` accepts static values, signals, or observables. The difference is that `signalMethod` does not provide operator composition -- it is a direct callback. Choose `signalMethod` when you need reactive method invocation without the overhead of building an RxJS pipeline.

---

## Entity Management and Normalization

### Entities

Once a store manages collections of domain objects, common operations emerge: add, update, remove, upsert, set all. Writing these by hand for every entity type is tedious and error-prone. The `withEntities()` feature from `@ngrx/signals/entities` provides a standardized solution:

```typescript
import { withEntities } from '@ngrx/signals/entities';

export const TransactionStore = signalStore(
  withEntities<Transaction>(),
  withDevtools({ name: 'TransactionStore' }),
);
```

This single feature adds several signals and operations to the store:

- `store.entities()` -- an array of all entities, preserving insertion order.
- `store.entityMap()` -- a record keyed by entity ID for O(1) lookups.
- `store.ids()` -- an array of all entity IDs.

### Entity Maps

Entity maps are the internal representation that makes `withEntities()` performant. When the store holds hundreds of transactions, looking up a single transaction by ID via `store.entities().find(...)` is O(n). The entity map provides O(1) access:

```typescript
const selected = store.entityMap()[selectedId];
```

This is especially valuable in computed signals that derive state from a single entity. Instead of scanning the full array, the computed reads directly from the map:

```typescript
withComputed((store) => ({
  selectedTransaction: computed(() => {
    const id = store.selectedId();
    return id ? store.entityMap()[id] : undefined;
  }),
})),
```

### Identifying Entities with IDs

By default, `withEntities()` expects each entity to have an `id` property of type `string` or `number`. If your domain uses a different identifier -- like `transactionRef` or a composite key -- you provide a custom `idKey`:

```typescript
withEntities<Transaction>({ idKey: 'transactionRef' }),
```

Alternatively, provide a `selectId` function for computed identifiers:

```typescript
withEntities<Holding>({
  selectId: (holding) => `${holding.accountId}-${holding.instrumentId}`,
}),
```

The ID must be stable across updates. If the ID changes, the entity system treats it as a removal of the old entity and insertion of a new one.

### Managing Several Entities in a Store

A store that manages a single entity collection is common, but some domains require multiple collections. The FinancialApp's portfolio view, for instance, might need both holdings and benchmark allocations in the same store. Use the `collection` option to namespace entities:

```typescript
export const PortfolioStore = signalStore(
  withEntities<Holding>({ collection: 'holdings' }),
  withEntities<Allocation>({ collection: 'benchmarks' }),
);
```

Each collection gets its own set of signals: `store.holdingsEntities()`, `store.holdingsEntityMap()`, `store.benchmarksEntities()`, and so on. The naming convention follows the pattern `{collection}Entities`, `{collection}EntityMap`, `{collection}Ids`.

When updating a named collection, pass the collection identifier to entity updaters:

```typescript
import { setAllEntities, updateEntity } from '@ngrx/signals/entities';

patchState(
  store,
  setAllEntities(holdings, { collection: 'holdings' }),
);

patchState(
  store,
  updateEntity(
    { id: holdingId, changes: { quantity: newQty } },
    { collection: 'holdings' },
  ),
);
```

### Normalization

Real-world API responses are often nested. A transaction response might embed the full account object, which itself embeds the client reference. Storing this nested structure directly leads to data duplication and stale references -- the same account appears in dozens of transactions, and updating it requires touching every copy.

Normalization solves this by flattening nested data into separate entity collections, linked by IDs:

```typescript
interface NormalizedData {
  transactions: Transaction[];
  accounts: Account[];
  holdings: Holding[];
}

function normalize(apiResponse: TransactionResponse[]): NormalizedData {
  const accounts = new Map<string, Account>();
  const holdings = new Map<string, Holding>();

  const transactions = apiResponse.map((res) => {
    accounts.set(res.account.id, res.account);
    res.account.holdings?.forEach((h) => holdings.set(h.id, h));
    return { ...res, accountId: res.account.id, account: undefined };
  });

  return {
    transactions,
    accounts: [...accounts.values()],
    holdings: [...holdings.values()],
  };
}
```

A store that consumes normalized data uses multiple `withEntities()` collections:

```typescript
export const TransactionStore = signalStore(
  withEntities<Transaction>({ collection: 'transactions' }),
  withEntities<Account>({ collection: 'accounts' }),
  withEntities<Holding>({ collection: 'holdings' }),
  withMethods((store) => {
    const api = inject(TransactionApi);

    return {
      async loadAll(accountId: string): Promise<void> {
        const response = await api.getByAccount(accountId);
        const data = normalize(response);
        patchState(store,
          setAllEntities(data.transactions, { collection: 'transactions' }),
          setAllEntities(data.accounts, { collection: 'accounts' }),
          setAllEntities(data.holdings, { collection: 'holdings' }),
        );
      },
    };
  }),
);
```

When a single account update arrives, you update it in one place -- the `accounts` collection -- and every computed signal that derives from it recalculates automatically. No more hunting through nested objects to patch stale data.

---

## Event API: Flux and Redux

> **Experimental:** The event API described in this section is currently experimental. Its surface area may change significantly in future NgRx releases. Use it in production only with an explicit upgrade strategy.

### Mental Model

The NgRx Signal Store's event API brings the Flux/Redux mental model -- actions, reducers, side effects -- into the signal-based world. This is useful when you need an audit trail of what happened and why, or when multiple independent handlers must react to the same event without coupling to each other.

The flow is familiar to anyone who has used NgRx global store: an event is dispatched, one or more handlers observe it, and state changes are applied through reducer-like functions. The difference is scope. In the Signal Store, events are local to the store instance -- not broadcast across the entire application.

### Events

Events are defined as typed factories using `event()`:

```typescript
import { event, emptyProps, props } from '@ngrx/signals/events';

export const TransactionEvents = {
  opened: event('[Transaction] Opened', props<{ id: string }>()),
  approved: event('[Transaction] Approved', props<{ id: string }>()),
  rejected: event(
    '[Transaction] Rejected',
    props<{ id: string; reason: string }>()
  ),
  loaded: event(
    '[Transaction] Loaded',
    props<{ transactions: Transaction[] }>()
  ),
  cleared: event('[Transaction] Cleared', emptyProps),
};
```

Each event has a string identifier for DevTools readability and a typed payload. The `emptyProps` constant signals events that carry no data.

### Reducer

The `withReducer()` feature defines how events transform state. It mirrors the classic reducer pattern, but operates on signal store state:

```typescript
import { withReducer, on } from '@ngrx/signals/events';

export const TransactionStore = signalStore(
  withState(initialState),
  withReducer(
    on(TransactionEvents.loaded, (state, { transactions }) => ({
      ...state,
      transactions,
      loading: false,
    })),
    on(TransactionEvents.approved, (state, { id }) => ({
      ...state,
      transactions: state.transactions.map((t) =>
        t.id === id ? { ...t, status: 'approved' as const } : t
      ),
    })),
    on(TransactionEvents.cleared, (state) => ({
      ...state,
      transactions: [],
      selectedId: null,
    })),
  ),
);
```

Reducers are pure functions. They receive the current state and the event payload, and return a new state. No side effects, no injections, no async operations. This constraint makes them trivially testable -- see [Chapter 7](ch07-testing-vitest.md) for patterns on testing store reducers.

### Event Handlers

For side effects -- API calls, logging, navigation -- use `withEventHandler()`. Handlers observe events but do not modify state directly. They can dispatch further events or call store methods:

```typescript
import { withEventHandler } from '@ngrx/signals/events';

export const TransactionStore = signalStore(
  withState(initialState),
  withReducer(/* ... */),
  withMethods(/* ... */),
  withEventHandler((store) => {
    const api = inject(TransactionApi);
    const router = inject(Router);

    return [
      on(TransactionEvents.approved, async ({ id }) => {
        await api.notifyApproval(id);
      }),
      on(TransactionEvents.rejected, ({ id }) => {
        router.navigate(['/transactions', id, 'rejection-summary']);
      }),
    ];
  }),
);
```

The separation is deliberate: reducers handle state transitions, handlers handle side effects. This keeps each concern testable in isolation.

### Dispatching Events

Events are dispatched through the store's `dispatch()` method (or via convenience methods that wrap it):

```typescript
@Component({
  template: `
    <button (click)="approve()">Approve</button>
    <button (click)="reject()">Reject</button>
  `,
})
export class TransactionActionsComponent {
  private readonly store = inject(TransactionStore);
  readonly txn = input.required<Transaction>();

  approve(): void {
    this.store.dispatch(TransactionEvents.approved({ id: this.txn().id }));
  }

  reject(): void {
    this.store.dispatch(
      TransactionEvents.rejected({ id: this.txn().id, reason: 'Policy violation' })
    );
  }
}
```

In practice, most teams wrap `dispatch()` calls inside store methods to keep the event vocabulary encapsulated:

```typescript
withMethods((store) => ({
  approve(id: string): void {
    store.dispatch(TransactionEvents.approved({ id }));
  },
  reject(id: string, reason: string): void {
    store.dispatch(TransactionEvents.rejected({ id, reason }));
  },
})),
```

Components then call `store.approve(id)` without knowing that events exist under the hood. This gives you the traceability of the event model without leaking its ceremony into every component.

---

## Custom Features

### Defining Custom Features

The composition model of `signalStore()` invites reuse. If multiple stores share a pattern -- call state tracking, pagination, optimistic updates -- you can extract it into a custom feature:

```typescript
import { signalStoreFeature, withState, withComputed } from '@ngrx/signals';
import { computed } from '@angular/core';

type CallState = 'idle' | 'loading' | 'loaded' | 'error';

export function withCallState() {
  return signalStoreFeature(
    withState<{ callState: CallState }>({ callState: 'idle' }),
    withComputed((store) => ({
      isLoading: computed(() => store.callState() === 'loading'),
      isLoaded: computed(() => store.callState() === 'loaded'),
      hasError: computed(() => store.callState() === 'error'),
    })),
  );
}
```

The `signalStoreFeature()` function bundles multiple features into one. The result is a factory function that can be dropped into any `signalStore()` call.

Helper functions simplify state updates from within methods:

```typescript
export function setLoading(): { callState: CallState } {
  return { callState: 'loading' };
}

export function setLoaded(): { callState: CallState } {
  return { callState: 'loaded' };
}

export function setError(): { callState: CallState } {
  return { callState: 'error' };
}
```

### Using Custom Features

Consuming a custom feature is no different from consuming a built-in one:

```typescript
export const TransactionStore = signalStore(
  withCallState(),
  withDevtools({ name: 'TransactionStore' }),
  withEntities<Transaction>(),
  withMethods((store) => {
    const api = inject(TransactionApi);

    return {
      async loadTransactions(accountId: string): Promise<void> {
        patchState(store, setLoading());
        try {
          const transactions = await api.getByAccount(accountId);
          patchState(store, setAllEntities(transactions), setLoaded());
        } catch {
          patchState(store, setError());
        }
      },
    };
  }),
);
```

The `TransactionStore` now has `isLoading()`, `isLoaded()`, and `hasError()` signals without any manual wiring. Every store in the FinancialApp that needs call state tracking can use the same feature, ensuring consistent naming and behavior across the codebase.

Custom features compose with each other and with built-in features. You can stack `withCallState()`, `withEntities()`, and `withDevtools()` in any order, and the accumulated state, computed signals, and methods merge into a single store surface. This composability is the defining advantage of the NgRx Signal Store architecture -- it turns cross-cutting concerns into plug-and-play building blocks.

---

## Summary

NgRx Signal Store provides a structured, composable layer on top of Angular's signal primitives. It does not replace lightweight stores for simple cases, but it earns its place when stores grow complex enough to benefit from standardized entity management, developer tooling, and reusable feature extraction.

The core ideas of this chapter:

- **`signalStore()` composes features** -- `withState()`, `withComputed()`, `withMethods()`, `withHooks()`, and `withEntities()` -- into a single injectable service. Feature ordering determines what each subsequent feature can access.
- **Entity management** with `withEntities()` provides normalized collections with O(1) lookups, named collections for multi-entity stores, and built-in CRUD updaters. Normalize API responses to avoid data duplication and stale references.
- **Redux DevTools** integration via `withDevtools()` enables time-travel debugging and state inspection. Guard it with `isDevMode()` to exclude it from production bundles.
- **Mutations** from `@ngrx/toolkit` encode the async operation lifecycle -- pending, success, failure -- into a repeatable pattern with per-mutation status signals.
- **Reactive methods** (`rxMethod` and `signalMethod`) let store methods accept signals and observables as inputs, enabling automatic re-execution when dependencies change.
- **The event API** brings the Flux/Redux mental model -- events, reducers, handlers -- into signal stores, providing an audit trail and decoupled side-effect handling for complex domains.
- **Custom features** extract cross-cutting patterns like call-state tracking into reusable `signalStoreFeature()` definitions. This composability turns recurring store patterns into plug-and-play building blocks.

In [Chapter 10](ch10-http-resource.md), we will examine how Angular's `resource()` and `httpResource()` APIs integrate with signal stores to provide a fully declarative data-fetching layer.
