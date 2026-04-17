# Chapter 44: Optimistic UI & Offline Sync

A FinancialApp user on a regional train hits "Add Transaction" to log a $42 lunch expense. The signal bar on their phone reads three bars of LTE, but the train is about to enter a tunnel. The request to `POST /api/transactions` will take 800ms on a good day, timeout on a bad one, and fail outright in the next ten seconds when the tunnel swallows the signal. The user has already moved on mentally -- they closed the wallet app, opened email, and scrolled halfway through their inbox. If they return to FinancialApp in two minutes and the transaction is not there, they will submit it again. Now the account has two $42 lunch entries and the user has lost trust in the ledger.

The cure is to stop waiting for the server. **Optimistic UI** applies the user's change locally the instant they confirm it, reconciles with the server asynchronously, and rolls back only when the server actually disagrees. The transaction appears in the list before the HTTP round-trip completes. If the network is down, the mutation is queued and submitted when connectivity returns. The user sees an app that responds to their intent, not an app that waits for permission.

This chapter covers the full pattern: optimistic updates layered on canonical server data, rollback on failure, offline persistence through IndexedDB, conflict resolution when the server and client disagree, and the UI affordances that keep the user informed about what has and has not been confirmed. We will build on the signal store patterns from [Chapter 9](ch09-ngrx-signal-store.md), the `linkedSignal` primitive from [Chapter 3](ch03-reactive-signals.md), the error surfaces from [Chapter 23](ch23-error-handling.md), and the service-worker foundation from [Chapter 26](ch26-pwa-service-workers.md).

> **Companion code:** `financial-app/libs/shared/offline-sync/` with `MutationQueue`, `SyncService`; integrated into the transaction submission flow. Extends the background sync from Ch 26.

---

## Optimistic vs Pessimistic: Choosing a Default

The default in most applications is still pessimistic: the user clicks, a spinner appears, the request completes, and only then does the UI reflect the change. This model is safe. It is also slow -- and the slowness compounds. On a 200ms connection, a sequence of five confirmations costs the user a full second of staring at spinners. Optimistic UI inverts this: the UI commits to the change immediately, and the server is allowed to catch up.

Neither approach is universally correct. The choice depends on three questions:

1. **How likely is the request to succeed?** Reads almost always succeed. Writes usually succeed -- the user is authenticated, the data is well-formed, the server is healthy. If success is the 99% case, optimistic UI is a free win. If success is the 60% case (a complex transfer that fails compliance checks half the time), optimistic UI is misleading.
2. **How painful is a rollback?** If reverting the change is visually disruptive -- an item appears in a list, sits there for two seconds, then vanishes with an error toast -- the user experience is worse than a brief spinner. If the change is small and the error is clearly communicated, rollback is fine.
3. **How confused will the user be if reversed?** A note on a transaction disappearing is a minor irritation. A $5,000 transfer that appeared to succeed, then reverted because the server rejected it, is a trust-destroying event. The user needs to know *before* they leave the app whether the operation committed.

For FinancialApp, the decision matrix looks like this:

| Operation | Optimistic? | Rationale |
|---|---|---|
| Filter/sort/search | Always | Pure client state; no server round-trip needed |
| Add a transaction (pending) | Yes | Shows up in list as `pending=true`; rollback is cheap |
| Edit a transaction note | Yes | Small text change; easy to revert with a toast |
| Tag a transaction (category) | Yes | Pure metadata; high success rate |
| Transfer money between accounts | **No** | Must show "Pending" until server confirms |
| Submit a trade order | **No** | Regulatory confirmation required before UI commits |
| Close an account | **No** | Irreversible; user must see explicit success |

The guideline writes itself: optimistic for reads and low-stakes writes; pessimistic for money movement, regulatory operations, and anything the user would want a receipt for. The rest of this chapter focuses on the optimistic half, but the critical-warning section near the end returns to where the line must be held.

---

## Optimistic Updates with `linkedSignal`

The problem with naive optimistic UI is that the server is the source of truth. If you mutate the local transaction list and then a background poll refreshes it from the server, your local mutation vanishes. The pattern that solves this cleanly is to keep two layers -- the canonical server data and a set of pending mutations -- and render a merged view that preserves both until the server catches up.

Angular's `linkedSignal()` (covered in [Chapter 3](ch03-reactive-signals.md)) is a natural fit. It derives from a source signal but remains writable, and it receives the previous value on every recomputation -- so you can reapply pending mutations whenever the server data refreshes:

```typescript
// libs/shared/offline-sync/src/lib/optimistic.ts
import { Signal, linkedSignal } from '@angular/core';
import { Transaction } from '@financial-app/shared/models';

export interface PendingMutation<T> {
  id: string;
  kind: 'create' | 'update' | 'delete';
  payload: T;
  submittedAt: number;
}

export function optimisticTransactions(
  server: Signal<Transaction[]>,
  pending: Signal<PendingMutation<Transaction>[]>,
) {
  return linkedSignal<Transaction[]>({
    source: () => ({ server: server(), pending: pending() }),
    computation: ({ server, pending }) => {
      const byId = new Map(server.map((t) => [t.id, t]));
      for (const m of pending) {
        if (m.kind === 'create') byId.set(m.payload.id, m.payload);
        else if (m.kind === 'update') byId.set(m.payload.id, { ...byId.get(m.payload.id), ...m.payload });
        else if (m.kind === 'delete') byId.delete(m.payload.id);
      }
      return Array.from(byId.values()).sort(byDateDesc);
    },
  });
}

function byDateDesc(a: Transaction, b: Transaction) {
  return b.date.localeCompare(a.date);
}
```

The view is a pure projection of `(server, pending)`. When the server data refreshes (say, a poll or WebSocket push lands), the merged view recomputes, and any still-pending mutations stay applied on top. When a mutation confirms and is removed from `pending`, the merged view recomputes with only the server data -- and if the server-confirmed version is identical to the optimistic one (as it almost always is), the template binding sees no change at all. The row does not flicker.

Here is how a transaction list component wires the merged signal into a template:

```typescript
// libs/features/transactions/src/lib/transaction-list.component.ts
@Component({
  selector: 'app-transaction-list',
  template: `
    @for (txn of view(); track txn.id) {
      <app-transaction-row
        [transaction]="txn"
        [pending]="isPending(txn.id)"
      />
    }
  `,
})
export class TransactionListComponent {
  private store = inject(TransactionStore);

  protected view = optimisticTransactions(
    this.store.serverTransactions,
    this.store.pendingMutations,
  );

  protected isPending(id: string): boolean {
    return this.store.pendingMutations().some((m) => m.payload.id === id);
  }
}
```

The `isPending` check drives the muted styling (covered later under UI patterns). The rest of the template is ordinary -- no special-casing, no conditional branches for "confirmed vs optimistic." The optimistic state is invisible to the rendering code.

---

## Rolling Back on Error

Optimistic UI earns its name because it assumes success. When the server disagrees, the mutation must be removed from the pending list, a meaningful message must reach the user, and any local state that depended on the mutation must also revert. A signal store (see [Chapter 9](ch09-ngrx-signal-store.md)) is the right home for this logic because pending mutations are exactly the kind of structured, tracked async state the store vocabulary is built for.

```typescript
// libs/shared/offline-sync/src/lib/transaction-store.ts
import { signalStore, withState, withMethods, patchState } from '@ngrx/signals';
import { inject } from '@angular/core';
import { TransactionApi } from '@financial-app/shared/data-access';
import { ErrorToastService } from '@financial-app/shared/ui';
import { PendingMutation } from './optimistic';

interface State {
  serverTransactions: Transaction[];
  pendingMutations: PendingMutation<Transaction>[];
}

export const TransactionStore = signalStore(
  { providedIn: 'root' },
  withState<State>({ serverTransactions: [], pendingMutations: [] }),
  withMethods((store) => {
    const api = inject(TransactionApi);
    const toast = inject(ErrorToastService);

    return {
      async addTransaction(draft: Omit<Transaction, 'id'>): Promise<void> {
        const optimistic: Transaction = {
          ...draft,
          id: `local-${crypto.randomUUID()}`,
          pending: true,
        };
        const mutation: PendingMutation<Transaction> = {
          id: optimistic.id,
          kind: 'create',
          payload: optimistic,
          submittedAt: Date.now(),
        };

        patchState(store, (s) => ({
          pendingMutations: [...s.pendingMutations, mutation],
        }));

        try {
          const confirmed = await api.create(draft);
          patchState(store, (s) => ({
            serverTransactions: [confirmed, ...s.serverTransactions],
            pendingMutations: s.pendingMutations.filter((m) => m.id !== mutation.id),
          }));
        } catch (err) {
          patchState(store, (s) => ({
            pendingMutations: s.pendingMutations.filter((m) => m.id !== mutation.id),
          }));
          toast.show('Could not save the transaction. Please try again.');
          throw err;
        }
      },
    };
  }),
);
```

Three responsibilities are cleanly separated. The local `id` uses a `local-*` prefix so the store can match the optimistic row against the eventual server response and swap them atomically. The `pendingMutations` array is the single source of truth for "what has the user done that the server has not yet acknowledged." The error path removes the mutation and delegates the user-facing message to `ErrorToastService` -- the transient-error pattern covered in [Chapter 23](ch23-error-handling.md).

**Partial-success handling.** Some mutations touch multiple resources: tagging a transaction might update both the transaction and a category counter. If the transaction update succeeds but the counter update fails, you have two choices: treat the whole batch as a transaction and roll back both, or commit what succeeded and surface a warning for the rest. Prefer the former when the resources are logically one thing; prefer the latter when they are independent and the user would rather keep partial progress. Either way, make the choice explicit in the store method -- do not let it depend on which call happens to fail first.

---

## Offline Support: The Mutation Queue

Once a mutation is encoded as a `PendingMutation` object, offline support is a small extension: instead of submitting every mutation immediately, queue them in memory (and in IndexedDB for durability), and drain the queue whenever the browser is online. This extends the background-sync service from [Chapter 26](ch26-pwa-service-workers.md) -- the chapter there introduced the queue; here we make it typed, observable, and integrated with the optimistic store.

### Tracking Online State

The browser exposes `navigator.onLine` and fires `online` / `offline` events on `window`. Wrapping these in a signal gives the rest of the app a reactive handle:

```typescript
// libs/shared/offline-sync/src/lib/network-status.service.ts
import { Injectable, signal } from '@angular/core';

@Injectable({ providedIn: 'root' })
export class NetworkStatusService {
  private readonly _online = signal(navigator.onLine);
  readonly online = this._online.asReadonly();

  constructor() {
    window.addEventListener('online', () => this._online.set(true));
    window.addEventListener('offline', () => this._online.set(false));
  }
}
```

`navigator.onLine` is a coarse signal -- it reports "does the OS think we have a network interface" rather than "can we actually reach the server." Treat it as a hint, not a guarantee. The queue drainer below still tries to submit and handles failures, so a false-positive `onLine=true` just causes a retry later.

### The MutationQueue

The queue persists pending mutations to IndexedDB via `idb-keyval`, a minimal wrapper over the raw IndexedDB API. Persistence survives tab reloads and browser restarts -- critical for mobile users whose browsers aggressively kill background tabs.

```typescript
// libs/shared/offline-sync/src/lib/mutation-queue.ts
import { Injectable, effect, inject, signal } from '@angular/core';
import { get, set, del } from 'idb-keyval';
import { NetworkStatusService } from './network-status.service';
import { ErrorToastService } from '@financial-app/shared/ui';
import { TransactionApi } from '@financial-app/shared/data-access';

export type MutationKind = 'create' | 'update' | 'delete';

export interface QueuedMutation {
  id: string;
  resource: 'transaction' | 'note' | 'tag';
  kind: MutationKind;
  payload: unknown;
  enqueuedAt: number;
  attempts: number;
}

const STORAGE_KEY = 'financial-app:mutation-queue';

@Injectable({ providedIn: 'root' })
export class MutationQueue {
  private readonly network = inject(NetworkStatusService);
  private readonly toast = inject(ErrorToastService);
  private readonly api = inject(TransactionApi);

  private readonly queue = signal<QueuedMutation[]>([]);
  readonly pending = this.queue.asReadonly();
  readonly size = () => this.queue().length;

  constructor() {
    void this.hydrate();
    effect(() => {
      if (this.network.online() && this.queue().length > 0) {
        void this.drain();
      }
    });
  }

  async enqueue(mutation: Omit<QueuedMutation, 'id' | 'enqueuedAt' | 'attempts'>): Promise<void> {
    const queued: QueuedMutation = {
      ...mutation,
      id: crypto.randomUUID(),
      enqueuedAt: Date.now(),
      attempts: 0,
    };
    this.queue.update((q) => [...q, queued]);
    await this.persist();
  }

  async retry(id: string): Promise<void> {
    if (this.network.online()) await this.drain();
  }

  private async hydrate(): Promise<void> {
    const saved = (await get<QueuedMutation[]>(STORAGE_KEY)) ?? [];
    this.queue.set(saved);
  }

  private async persist(): Promise<void> {
    const current = this.queue();
    if (current.length === 0) await del(STORAGE_KEY);
    else await set(STORAGE_KEY, current);
  }

  private async drain(): Promise<void> {
    for (const m of this.queue()) {
      try {
        await this.submit(m);
        this.queue.update((q) => q.filter((x) => x.id !== m.id));
      } catch {
        this.queue.update((q) => q.map((x) => (x.id === m.id ? { ...x, attempts: x.attempts + 1 } : x)));
        if (m.attempts + 1 >= 5) {
          this.toast.show(`A change could not be synced after 5 attempts. Retry from the sync panel.`);
        }
        break;
      }
    }
    await this.persist();
  }

  private submit(m: QueuedMutation): Promise<unknown> {
    if (m.resource === 'transaction' && m.kind === 'create') return this.api.create(m.payload as Transaction);
    if (m.resource === 'transaction' && m.kind === 'update') return this.api.update(m.payload as Transaction);
    if (m.resource === 'transaction' && m.kind === 'delete') return this.api.delete((m.payload as { id: string }).id);
    throw new Error(`Unsupported mutation: ${m.resource}/${m.kind}`);
  }
}
```

Four invariants make this queue safe:

- **Ordered submission.** `drain()` processes mutations in insertion order and stops on the first failure. A create-then-update sequence must not reorder.
- **Idempotent keys.** Each mutation carries a client-generated UUID that the server uses to deduplicate, so a retry after a response lost in transit does not produce a double-post.
- **Bounded retries.** A mutation that has failed five times is parked -- the user is notified and can retry manually from a sync panel. Infinite retries on a permanently broken request just burn battery.
- **Durable state.** `persist()` writes after every queue mutation so a crash between enqueue and submit does not lose the write.

The store method from earlier integrates with the queue: when offline, it enqueues instead of calling the API directly.

```typescript
async addTransaction(draft: Omit<Transaction, 'id'>): Promise<void> {
  // ...apply optimistic mutation...
  if (this.network.online()) {
    try {
      const confirmed = await this.api.create(draft);
      // ...swap optimistic for confirmed...
    } catch {
      await this.queue.enqueue({ resource: 'transaction', kind: 'create', payload: draft });
    }
  } else {
    await this.queue.enqueue({ resource: 'transaction', kind: 'create', payload: draft });
  }
}
```

The optimistic UI stays exactly the same whether the user is online or offline. The only difference is whether the pending mutation is confirmed in 400ms or in twenty minutes when the tunnel ends.

---

## Conflict Resolution

When the client and server diverge, someone has to adjudicate. Three strategies cover the practical space:

**Last-write-wins (LWW).** The most recent timestamp wins. Simple to implement, easy to reason about, and guaranteed to converge -- but in genuine conflicts, the loser's change is silently discarded. Fine for idempotent updates (toggling a "starred" flag); dangerous for destructive operations (replacing a long note).

**Server-authoritative with client retry.** The server rejects the client's mutation with a `409 Conflict`, returns the current state, and the client decides whether to reapply, transform, or abandon its change. This is the right default for most financial data: the server holds the truth, the client takes responsibility for merging. The retry logic can be as simple as a prompt ("The note was changed by another session. Keep your version?") or as sophisticated as a diff viewer.

**CRDTs for mergeable types.** A Conflict-free Replicated Data Type is a data structure whose operations are commutative, associative, and idempotent -- so any order of operations from any set of replicas produces the same final state. CRDTs are worth a brief primer because they show up in collaborative editors, chat histories, and (occasionally) trading systems.

The canonical example is a **grow-only counter**: clients only increment, and each client tracks its own count. To read the counter, sum across all client counts. Merging two replicas is just a per-client `Math.max`:

```typescript
// Commutative counter for portfolio share totals across devices
type GCounter = Record<string, number>; // clientId -> count

const increment = (c: GCounter, clientId: string, n: number): GCounter => ({
  ...c,
  [clientId]: (c[clientId] ?? 0) + n,
});

const value = (c: GCounter): number =>
  Object.values(c).reduce((sum, n) => sum + n, 0);

const merge = (a: GCounter, b: GCounter): GCounter => {
  const out: GCounter = { ...a };
  for (const [id, n] of Object.entries(b)) out[id] = Math.max(out[id] ?? 0, n);
  return out;
};
```

Two devices that increment offline can merge their states in any order and agree on the final share count. For a simple portfolio where users buy shares across web and mobile without coordination, this eliminates a whole class of race conditions.

**Why CRDTs are usually overkill for a financial app.** Most of FinancialApp's state is not collaboratively edited. A transaction is owned by one user. A portfolio has a single canonical ledger on the server. The server is authoritative, online, and fast -- the scenarios where CRDTs shine (long-lived offline collaboration, peer-to-peer sync, multiple concurrent writers to the same field) are rare. The complexity cost of a CRDT library is real: you replace every primitive value with an operation log, every read with a materialization, and every test with a concurrency test. For a banking ledger, server-authoritative with a thoughtful retry strategy is almost always the right answer. Know CRDTs exist; reach for them only when the problem is genuinely peer-to-peer.

---

## Sync Strategies

Once you have optimistic UI and a mutation queue, the last piece is keeping local state fresh. Three patterns cover the space:

- **Pull.** The client polls the server on an interval or on focus. Simple, robust, and easy to rate-limit, but trades staleness for simplicity. Fine for data that changes rarely (account metadata, portfolio definitions) or when eventual consistency is acceptable (transaction history at day-end).
- **Push.** The server pushes updates over a WebSocket or Server-Sent Events connection. Near-real-time, but requires a long-lived connection, reconnection logic, and server-side fan-out infrastructure. The right choice for price ticks, balance updates, and anything where staleness is measured in seconds.
- **Bidirectional.** Both sides publish, and a conflict resolver merges incoming server state with local pending mutations. This is what the `linkedSignal` pattern from earlier naturally produces -- the server pushes a new canonical list, the merged view reapplies pending mutations, and the UI converges.

A per-resource table is usually the clearest way to document the choice:

| Resource | Strategy | Freshness target | Notes |
|---|---|---|---|
| Account balance | Push (WebSocket) | < 2s | High-stakes; reuse the quote-feed channel |
| Holdings pricing | Push (WebSocket) | < 1s | Tick data; dedicated subscription |
| Transaction list | Pull on focus + push for new rows | < 5s | Too large to full-refresh; diff by timestamp |
| Transaction notes/tags | Bidirectional | Eventual | Pending mutations tolerated for minutes |
| Portfolio definitions | Pull (every 5 min) | Minutes | Changes rarely; not worth a subscription |
| User preferences | Pull on app start | On login | Loaded once; invalidate on logout |

No single strategy covers everything. A well-tuned app uses all three across different resources, with the mutation queue as a common substrate underneath. See [Chapter 33](ch33-websockets-realtime.md) for the WebSocket transport patterns that power the push channels.

---

## UI Patterns for Pending State

Optimistic UI works only if the user can see the difference between confirmed and unconfirmed state. Three small affordances carry most of the weight.

**Muted pending rows.** An optimistic row should render in a subtle, lower-contrast style until the server confirms. Reduce opacity to `0.6`, use a dashed border, or show a small spinner in the corner. The goal is "this is here but not final" -- not so subtle the user misses it, not so loud they think something is broken.

```scss
// transaction-row.component.scss
.transaction-row {
  transition: opacity 200ms ease;
  &.pending {
    opacity: 0.6;
    border-left: 2px dashed var(--color-warning);
  }
}
```

**Sync status banner.** A thin banner at the top of the screen tells the user the truth about their network and queue state:

```typescript
@Component({
  selector: 'app-sync-status-banner',
  template: `
    @if (!network.online()) {
      <div class="sync-banner offline" role="status">
        You're offline — {{ queue.size() }} pending
        {{ queue.size() === 1 ? 'change' : 'changes' }}
        will sync when connection is restored.
      </div>
    } @else if (queue.size() > 0) {
      <div class="sync-banner syncing" role="status" aria-live="polite">
        Syncing {{ queue.size() }} pending
        {{ queue.size() === 1 ? 'change' : 'changes' }}…
      </div>
    }
  `,
})
export class SyncStatusBannerComponent {
  protected network = inject(NetworkStatusService);
  protected queue = inject(MutationQueue);
}
```

**Manual retry.** For mutations that have hit the retry ceiling, a dedicated sync panel lists the stuck items with "Retry" and "Discard" buttons. Users who have seen the same change fail five times deserve the agency to decide what happens next -- not a silent background loop that drains the battery.

---

## Critical Warning: Money Movement Must Be Pessimistic

Every pattern in this chapter stops at the moment real money moves. **Never apply optimistic UI to money-movement confirmations.** A transfer, a trade, a payout, a withdrawal -- the user must see "Pending" until the server returns a definitive answer. The reasons are operational, legal, and psychological:

- **Regulatory requirements.** In many jurisdictions, a confirmation shown to the user implies that the institution has committed to the operation. Showing "Transfer complete" optimistically -- and later reverting -- is a compliance violation waiting to be reported.
- **Reconciliation errors.** A user who sees "$500 transferred" and navigates away assumes the money moved. If the operation later fails, they do not learn about it until the next statement. The cost of tracking down the discrepancy falls on support, not on whichever backend service dropped the request.
- **Trust.** A single optimistic rollback on a five-figure transfer is a story the user tells for the rest of their life, usually to the institution's competitors.

The right pattern for money movement is: disable the button, show a spinner, wait for the server, render either an explicit confirmation screen or an error. Treat the round-trip as part of the transaction, not a detail to be hidden. Optimistic UI belongs on notes, tags, filters, UI state, and the many low-stakes writes that surround the core ledger -- it does not belong on the ledger itself.

---

## Testing

Optimistic flows have more branches than pessimistic ones, so tests matter more. Three scenarios deserve dedicated coverage: the happy path, the rollback path, and the offline path.

**Simulating offline.** Vitest's `vi.stubGlobal` swaps `navigator` for a stubbed object so the `NetworkStatusService` reads `online: false`:

```typescript
// libs/shared/offline-sync/src/lib/mutation-queue.spec.ts
import { beforeEach, describe, expect, it, vi } from 'vitest';
import { TestBed } from '@angular/core/testing';
import { MutationQueue } from './mutation-queue';
import { TransactionApi } from '@financial-app/shared/data-access';

describe('MutationQueue (offline)', () => {
  beforeEach(() => {
    vi.stubGlobal('navigator', { onLine: false });
    TestBed.configureTestingModule({
      providers: [{ provide: TransactionApi, useValue: { create: vi.fn() } }],
    });
  });

  it('holds mutations while offline and flushes when back online', async () => {
    const queue = TestBed.inject(MutationQueue);
    const api = TestBed.inject(TransactionApi) as { create: ReturnType<typeof vi.fn> };
    api.create.mockResolvedValue({ id: 'server-1' });

    await queue.enqueue({ resource: 'transaction', kind: 'create', payload: { amount: 42 } });
    expect(queue.size()).toBe(1);
    expect(api.create).not.toHaveBeenCalled();

    vi.stubGlobal('navigator', { onLine: true });
    window.dispatchEvent(new Event('online'));
    await vi.waitUntil(() => queue.size() === 0);
    expect(api.create).toHaveBeenCalledOnce();
  });
});
```

**Testing rollback paths.** Mock the API to reject, trigger the store method, and assert both the pending mutation is removed and the error toast was shown. Follow the testing patterns from [Chapter 7](ch07-testing-vitest.md) -- `TestBed.inject` for the store, a fake `ErrorToastService` that records calls, and `vi.spyOn` to verify the cleanup sequence.

**E2E offline scenarios.** Playwright's `context.setOffline(true)` (see [Chapter 25](ch25-e2e-playwright.md)) drives the same code path end-to-end. A realistic scenario: enter a transaction, go offline, enter another, come back online, verify both appear exactly once and the pending-row styling disappears on confirmation. Cover it once with a full browser test; rely on unit tests for the branches.

---

## Summary

Optimistic UI is a trade of complexity for responsiveness: the app commits to the user's change immediately, the server reconciles in the background, and rollback handles the minority case where the server disagrees. Done well, it makes a financial application feel local even on flaky networks. Done poorly, it erodes trust and creates ghost data.

- **Pick optimistic deliberately.** Use it for reads, low-stakes writes, filters, notes, and tags. Keep money movement, trades, and irreversible operations pessimistic -- the user must see "Pending" until the server confirms.
- **Layer optimistic state on canonical server data** with a `linkedSignal` that merges pending mutations on top of the server snapshot. The view is a pure projection; refreshes reapply pending mutations automatically.
- **Track pending mutations in a signal store.** Add to the list when the mutation is dispatched, remove on success or failure, and surface errors through the toast infrastructure from [Chapter 23](ch23-error-handling.md).
- **Extend the background sync from [Chapter 26](ch26-pwa-service-workers.md) with a `MutationQueue`** that persists to IndexedDB, reacts to `navigator.onLine` through a signal, processes mutations in order, deduplicates via client-generated UUIDs, and stops on failure to preserve ordering. Survive page reloads; bound retries; expose a manual retry path for parked mutations.
- **Resolve conflicts explicitly.** Last-write-wins is simple and lossy. Server-authoritative with client retry is the right default for financial data. CRDTs are elegant but rarely worth the complexity outside genuinely collaborative scenarios.
- **Choose a sync strategy per resource.** Pull for rarely-changing data, push (WebSocket) for real-time feeds, bidirectional for user-editable state with server enrichment. A single app uses all three.
- **Communicate pending state visually.** Muted rows, a sync-status banner, and a manual retry UI keep the user informed without being intrusive. Announce state changes to assistive technology with `aria-live`.
- **Test all three paths.** Stub `navigator.onLine` for unit tests, drive online/offline transitions with events, and cover the end-to-end flow with Playwright's `setOffline`.

The recurring theme is that optimistic UI is not a single feature -- it is a set of patterns that touch data fetching, state management, persistence, UI affordances, and testing. Build the pieces once into shared libraries, integrate them into the transaction submission flow, and the rest of the application inherits responsiveness for free. The user on the train types "42, lunch, café" into their phone, the entry appears before the tunnel swallows the signal, the mutation queues silently, and when the train rolls into the next station it syncs without the user ever knowing how close it came to being lost.
