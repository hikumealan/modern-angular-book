# Chapter 36: Drag and Drop

Drag-and-drop is one of those interactions that feels trivial until you try to build it well. A user grabs a ticker from a watchlist and drops it three rows up; the list should settle smoothly, not teleport. An advisor drags a pending wire transfer from "Needs Review" to "Approved"; the card should glide across columns. A client rearranges dashboard widgets on a tablet, and the interaction must behave identically with a mouse, a finger, and a keyboard -- because not every user can use every input device.

FinancialApp has three canonical drag-and-drop surfaces, each mapping to a different primitive in `@angular/cdk/drag-drop`:

- **Watchlist reordering** -- a single list sorted by personal priority.
- **Transaction kanban** -- three columns (Pending, Approved, Rejected) that move transactions through an approval workflow.
- **Dashboard widgets** -- a free-drag canvas where each user pins summaries and charts wherever they like.

All three share the same accessibility, animation, and persistence concerns. This chapter walks through them in order, then covers testing and the production details that separate a demo from a shipped feature.

> **Companion code:** Watchlist reorder in `financial-app/apps/financial-app/src/app/features/portfolios/`; dashboard widget drag in `libs/shared/ui/`.

---

## Why the CDK, and Not HTML5 Drag-and-Drop

The browser ships with a native drag-and-drop API -- `draggable="true"`, `dragstart`, `dragover`, `drop` -- and on paper it looks sufficient. In practice almost every serious Angular application reaches for `@angular/cdk/drag-drop` instead:

- **No touch support.** HTML5 drag fires only on mouse events. The CDK uses Pointer Events and works identically on mouse, pen, and touch.
- **No keyboard support.** HTML5 has no accessible equivalent. The CDK gives you Space to pick up, arrow keys to move, Space to drop, Escape to cancel.
- **Browser-constrained visuals.** The native preview is a ghost image the browser generates, with a cursor decoration you cannot override. The CDK renders a real DOM node you can style however you like.
- **No animations.** Native drop snaps into place instantly. The CDK animates every sibling that shifts and interpolates the dropped element into its final slot.
- **Clunky event model.** HTML5's `dataTransfer` only holds strings. CDK drag operations carry the actual TypeScript object through.

The CDK exposes three primitives you use constantly: `cdkDrag` marks an element as draggable, `cdkDropList` marks a container as a drop target, and `cdkDropListGroup` links multiple drop lists so items can move between them. Everything else -- previews, placeholders, handles, boundaries -- layers on top. Install with `npm install @angular/cdk`; the directives are standalone, so no provider setup is needed.

---

## Sorting Within a Single List: Watchlist Reorder

The simplest drag-drop interaction is reordering a single list in place. The watchlist is a signal of ticker entries the user ranks from most to least important:

```typescript
// financial-app/apps/financial-app/src/app/features/portfolios/watchlist.component.ts
import { Component, signal } from '@angular/core';
import {
  CdkDrag,
  CdkDragDrop,
  CdkDropList,
  moveItemInArray,
} from '@angular/cdk/drag-drop';

interface WatchlistEntry {
  ticker: string;
  company: string;
  lastPrice: number;
}

@Component({
  selector: 'app-watchlist',
  imports: [CdkDropList, CdkDrag],
  template: `
    <h2>My Watchlist</h2>
    <ul class="watchlist" cdkDropList (cdkDropListDropped)="onDrop($event)">
      @for (entry of watchlist(); track entry.ticker) {
        <li class="watchlist__row" cdkDrag>
          <span class="ticker">{{ entry.ticker }}</span>
          <span class="company">{{ entry.company }}</span>
          <span class="price">{{ entry.lastPrice | currency }}</span>
        </li>
      }
    </ul>
  `,
  styleUrl: './watchlist.component.scss',
})
export class WatchlistComponent {
  protected readonly watchlist = signal<WatchlistEntry[]>([
    { ticker: 'AAPL', company: 'Apple Inc.',      lastPrice: 189.42 },
    { ticker: 'MSFT', company: 'Microsoft Corp.', lastPrice: 418.56 },
    { ticker: 'GOOG', company: 'Alphabet Inc.',   lastPrice: 172.18 },
    { ticker: 'AMZN', company: 'Amazon.com Inc.', lastPrice: 198.03 },
    { ticker: 'NVDA', company: 'NVIDIA Corp.',    lastPrice: 131.77 },
  ]);

  protected onDrop(event: CdkDragDrop<WatchlistEntry[]>): void {
    if (event.previousIndex === event.currentIndex) return;
    this.watchlist.update((current) => {
      const next = [...current];
      moveItemInArray(next, event.previousIndex, event.currentIndex);
      return next;
    });
  }
}
```

Three things do the real work. `cdkDropList` on the `<ul>` tracks the positions of every child `cdkDrag` and emits `(cdkDropListDropped)` on release. `cdkDrag` on each `<li>` makes the row draggable and runs the "sort while hovering" animation, sliding siblings out of the way. `moveItemInArray()` mutates an array in place; because signals use reference equality, we copy the array with `[...current]`, mutate the copy, and return it from `update()`. Passing `current` directly would mutate the signal's backing array without notifying subscribers -- a subtle bug worth internalizing.

The `CdkDragDrop<T[]>` event exposes `previousIndex` and `currentIndex`; early-returning when they match avoids churning the signal when a user picks up and drops a row in place. See [Chapter 11](ch11-directives-templates.md) for why directives like `cdkDrag` attach behavior to existing elements without introducing new DOM nodes.

---

## Transferring Between Lists: The Transaction Kanban

The kanban is where drag-drop earns its keep. An advisor works a queue of pending transactions, dragging each into an Approved or Rejected column. The data shape is three arrays of `Transaction`, one per column:

```typescript
// financial-app/apps/financial-app/src/app/features/transactions/approval-kanban.component.ts
import { Component, signal, WritableSignal } from '@angular/core';
import {
  CdkDrag,
  CdkDragDrop,
  CdkDropList,
  CdkDropListGroup,
  moveItemInArray,
  transferArrayItem,
} from '@angular/cdk/drag-drop';
import { CurrencyPipe, DatePipe } from '@angular/common';

import { Transaction } from '@financial-app/shared/models';

type ApprovalStatus = 'pending' | 'approved' | 'rejected';

@Component({
  selector: 'app-approval-kanban',
  imports: [CdkDropListGroup, CdkDropList, CdkDrag, CurrencyPipe, DatePipe],
  template: `
    <div class="kanban" cdkDropListGroup>
      @for (column of columns; track column.status) {
        <section class="kanban__column">
          <h3>{{ column.label }} ({{ column.items().length }})</h3>
          <div
            class="kanban__list"
            cdkDropList
            [cdkDropListData]="column.items()"
            [id]="'list-' + column.status"
            (cdkDropListDropped)="onDrop($event, column.status)">
            @for (txn of column.items(); track txn.id) {
              <article class="txn-card" cdkDrag>
                <header>
                  <strong>{{ txn.description }}</strong>
                  <span>{{ txn.amount | currency }}</span>
                </header>
                <small>{{ txn.category }} &middot; {{ txn.date | date }}</small>
              </article>
            } @empty {
              <p class="empty">Drop transactions here.</p>
            }
          </div>
        </section>
      }
    </div>
  `,
  styleUrl: './approval-kanban.component.scss',
})
export class ApprovalKanbanComponent {
  private readonly signalFor: Record<ApprovalStatus, WritableSignal<Transaction[]>> = {
    pending:  signal<Transaction[]>([]),
    approved: signal<Transaction[]>([]),
    rejected: signal<Transaction[]>([]),
  };

  protected readonly columns = [
    { status: 'pending'  as const, label: 'Pending',  items: this.signalFor.pending  },
    { status: 'approved' as const, label: 'Approved', items: this.signalFor.approved },
    { status: 'rejected' as const, label: 'Rejected', items: this.signalFor.rejected },
  ];

  protected onDrop(event: CdkDragDrop<Transaction[]>, target: ApprovalStatus): void {
    if (event.previousContainer === event.container) {
      this.signalFor[target].update((items) => {
        const next = [...items];
        moveItemInArray(next, event.previousIndex, event.currentIndex);
        return next;
      });
      return;
    }

    const sourceStatus = event.previousContainer.id.replace('list-', '') as ApprovalStatus;
    const sourceNext = [...this.signalFor[sourceStatus]()];
    const destNext = [...this.signalFor[target]()];
    transferArrayItem(sourceNext, destNext, event.previousIndex, event.currentIndex);
    this.signalFor[sourceStatus].set(sourceNext);
    this.signalFor[target].set(destNext);
  }
}
```

`cdkDropListGroup` wraps every list that should accept transfers from every other; without the group, each `cdkDropList` only accepts drops from itself. `[cdkDropListData]` associates the container with its backing array so `event.previousContainer.data` and `event.container.data` expose the arrays inside the handler. `transferArrayItem()` mirrors `moveItemInArray()` but moves an entry between two arrays; like its cousin, it mutates in place, so we copy both arrays first and call `set()` on both signals. The `previousContainer === container` check distinguishes same-column reorders from cross-column moves -- collapsing the two branches is possible but hurts readability.

In production you persist both a status field (`pending` &rarr; `approved`) and an order-within-column index. Persistence gets its own section below.

---

## Drag Previews, Placeholders, and Handles

The defaults are serviceable: the element you grabbed is cloned into a floating preview, and a dimmed copy stays in the list as a placeholder. Three structural directives customize the experience. `*cdkDragPreview` replaces the cloned preview with a compact node that floats with the pointer. `*cdkDragPlaceholder` replaces the ghost left behind in the source slot -- a dashed outline often reads better than an opaque copy. `cdkDragHandle` scopes drag-start to a dedicated grip, so the rest of the card remains clickable for a detail view:

```html
<article class="txn-card" cdkDrag (click)="openDetail(txn)">
  <button type="button" cdkDragHandle class="drag-grip" aria-label="Reorder transaction">
    <svg aria-hidden="true" viewBox="0 0 16 16"><!-- six-dot grip icon --></svg>
  </button>
  <header>
    <strong>{{ txn.description }}</strong>
    <span>{{ txn.amount | currency }}</span>
  </header>

  <div *cdkDragPreview class="txn-card--preview">
    {{ txn.description }} &mdash; {{ txn.amount | currency }}
  </div>
  <div *cdkDragPlaceholder class="txn-card__ghost"></div>
</article>
```

Note the `aria-label` on the handle: a grip icon is invisible to screen readers without one. Giving it a text label is mandatory -- see [Chapter 22](ch22-accessibility-aria.md).

---

## Free Drag with Boundaries: Dashboard Widgets

Sorting and kanban both constrain movement to a container's child list. Dashboards need a different model: each widget is a floating tile the user can position anywhere inside the canvas. The CDK supports this through free drag -- `cdkDrag` used *without* a surrounding `cdkDropList`, positioned by absolute offset:

```typescript
// financial-app/libs/shared/ui/src/dashboard/dashboard-canvas.component.ts
import { Component, signal } from '@angular/core';
import { CdkDrag, CdkDragEnd, Point } from '@angular/cdk/drag-drop';

interface DashboardWidget {
  id: string;
  title: string;
  position: Point;
}

@Component({
  selector: 'fin-dashboard-canvas',
  imports: [CdkDrag],
  template: `
    <div class="canvas">
      @for (widget of widgets(); track widget.id) {
        <section
          class="widget"
          cdkDrag
          cdkDragBoundary=".canvas"
          [cdkDragFreeDragPosition]="widget.position"
          (cdkDragEnded)="onDragEnded(widget.id, $event)">
          <header cdkDragHandle>{{ widget.title }}</header>
          <div class="widget__body"><ng-content /></div>
        </section>
      }
    </div>
  `,
  styleUrl: './dashboard-canvas.component.scss',
})
export class DashboardCanvasComponent {
  protected readonly widgets = signal<DashboardWidget[]>([
    { id: 'balance',     title: 'Total Balance',     position: { x:   0, y:   0 } },
    { id: 'portfolio',   title: 'Portfolio Summary', position: { x: 320, y:   0 } },
    { id: 'recent-txns', title: 'Recent Activity',   position: { x:   0, y: 240 } },
  ]);

  protected onDragEnded(id: string, event: CdkDragEnd): void {
    const position = event.source.getFreeDragPosition();
    this.widgets.update((list) =>
      list.map((w) => (w.id === id ? { ...w, position } : w)),
    );
  }
}
```

`cdkDragBoundary=".canvas"` clamps the drag to the first ancestor matching the selector -- the widget cannot be dragged past the canvas edges. The value can also be an `ElementRef` when a CSS selector is awkward. `[cdkDragFreeDragPosition]` sets the widget's position from a signal, which lets you persist layouts to localStorage or the server and restore them on the next visit. `(cdkDragEnded)` fires when the user releases; `event.source.getFreeDragPosition()` returns the final coordinates, which we write back into the signal.

For grid-snapping, compose with `[cdkDragConstrainPosition]="snapToGrid"`, which receives the raw pointer position and returns a possibly-snapped coordinate. It runs on every pointer move, so keep it arithmetic-only:

```typescript
protected snapToGrid = (point: Point): Point => ({
  x: Math.round(point.x / 16) * 16,
  y: Math.round(point.y / 16) * 16,
});
```

---

## Animations

The CDK's polish comes from three CSS classes it adds and removes around drags: `.cdk-drag-preview` on the floating overlay, `.cdk-drag-placeholder` on the ghost that stays behind, and `.cdk-drop-list-dragging` on the container while a drag is in progress. The minimum set of styles that makes a list feel smooth:

```scss
.watchlist {
  .cdk-drag-preview     { box-shadow: 0 8px 24px rgba(0, 0, 0, 0.2); border-radius: 0.5rem; }
  .cdk-drag-placeholder { opacity: 0.25; }
  .cdk-drag-animating   { transition: transform 200ms cubic-bezier(0, 0, 0.2, 1); }
}

.watchlist.cdk-drop-list-dragging .watchlist__row:not(.cdk-drag-placeholder) {
  transition: transform 200ms cubic-bezier(0, 0, 0.2, 1);
}
```

The last rule is the one people forget: without a transition on the non-dragged siblings, they teleport into their new positions when sort order changes mid-drag. With it, they slide. The duration and easing mirror the CDK's own defaults, keeping the dropped element and its siblings synchronized. `[cdkDragReleaseAnimationDuration]` on the drop list tunes the settle duration; the default of 200ms matches Angular Material.

---

## Touch and Pointer Support

The CDK dispatches drag events through Pointer Events, which unify mouse, pen, and touch on every browser released since 2019. No configuration is required -- the watchlist, kanban, and dashboard all work on an iPad the moment you plug in a touchscreen. Two touch-specific details matter. First, the CDK uses a short hold (250ms on touch, immediate on mouse) to distinguish a drag from a scroll; if your list is inside a scrollable container, this delay is what prevents every upward swipe from initiating a drag. Tune it per element with `[cdkDragStartDelay]="{ touch: 200, mouse: 0 }"`. Second, iOS Safari will occasionally steal a drag gesture for scroll or text-selection; the CDK sets `touch-action: none` automatically on `cdkDrag` elements, but if you wrap `cdkDragHandle` around a custom component that does not inherit it, apply `touch-action: none` yourself.

---

## Accessibility: Keyboard and Screen Readers

This is the section where the CDK pays for itself. Keyboard interaction is wired in automatically on every `cdkDrag` element:

| Key                    | Action                                              |
|------------------------|-----------------------------------------------------|
| **Tab**                | Focus the next draggable element                    |
| **Space** or **Enter** | Pick up the focused item                            |
| **Arrow keys**         | Move the picked-up item within or between lists     |
| **Space** or **Enter** | Drop at the current position                        |
| **Escape**             | Cancel the drag and return to the original position |

The same `onDrop` handler runs whether the drag was performed with a mouse, a finger, or arrow keys. You write *drop* logic; the CDK feeds it through every input modality.

### Live Announcements

The CDK emits ARIA live-region announcements during a drag -- "Item picked up," "Moved up," "Item dropped" -- but the defaults are generic. For a kanban where "Moved from Pending to Approved" is far more useful, override via `LiveAnnouncer` inside the drop handler:

```typescript
import { inject } from '@angular/core';
import { LiveAnnouncer } from '@angular/cdk/a11y';

private readonly announcer = inject(LiveAnnouncer);

protected onDrop(event: CdkDragDrop<Transaction[]>, target: ApprovalStatus): void {
  // ...existing move logic...

  const txn = event.container.data[event.currentIndex];
  const message = event.previousContainer === event.container
    ? `Moved ${txn.description} to position ${event.currentIndex + 1} in ${target}.`
    : `Moved ${txn.description} from ${event.previousContainer.id.replace('list-', '')} to ${target}.`;
  this.announcer.announce(message, 'polite');
}
```

`'polite'` is correct here -- we are confirming a user-initiated action, not warning them about one. Reserve `'assertive'` for errors ("Could not save reorder -- reverted"). The CDK also handles focus restoration after a keyboard drop, as long as your template does not destroy and re-create the dropped element during change detection. See [Chapter 22](ch22-accessibility-aria.md) for the full `LiveAnnouncer` API and the axe-core harness you should run against every screen that uses drag-drop.

---

## Persisting the Result

A reorder that survives page refresh is a reorder that meant something. The watchlist lives on the server keyed by user id; two patterns pay for themselves here.

**Optimistic UI with rollback.** Update the signal immediately so the UI feels instant, then fire the network call. If it fails, restore the previous state and explain the error:

```typescript
protected async onDrop(event: CdkDragDrop<WatchlistEntry[]>): Promise<void> {
  if (event.previousIndex === event.currentIndex) return;

  const previous = this.watchlist();
  const next = [...previous];
  moveItemInArray(next, event.previousIndex, event.currentIndex);
  this.watchlist.set(next);

  try {
    await this.watchlistApi.saveOrder(next.map((e) => e.ticker));
  } catch {
    this.watchlist.set(previous);
    this.announcer.announce('Could not save the new order. Reverted.', 'assertive');
  }
}
```

The pattern generalizes: snapshot, mutate, call, rollback on error. The full optimistic-UI toolkit -- debouncing, conflict resolution, pending-write indicators -- gets its own treatment in Chapter 44.

**Batched persistence.** If the user is dragging through a long sort, one HTTP request per drop is wasteful. Debounce saves through a signal effect:

```typescript
constructor() {
  toObservable(this.watchlist)
    .pipe(skip(1), debounceTime(400), takeUntilDestroyed())
    .subscribe((list) => this.watchlistApi.saveOrder(list.map((e) => e.ticker)));
}
```

`skip(1)` drops the initial synchronous emission so we do not save on mount. Debouncing has one sharp edge: if the user closes the tab within the 400ms window, the last change is lost. Pair it with a `beforeunload` listener or an explicit "Done" button for irreversible workflows.

---

## Testing Drag-and-Drop

Drag-and-drop is the kind of feature where manual QA never scales -- every refactor risks breaking the move logic or the announcement text, neither of which is visible in a screenshot diff. Two flavors of test cover the interaction.

**Harnesses** from `@angular/cdk/drag-drop/testing` wrap the low-level pointer events:

```typescript
// financial-app/apps/financial-app/src/app/features/portfolios/watchlist.component.spec.ts
import { TestBed } from '@angular/core/testing';
import { TestbedHarnessEnvironment } from '@angular/cdk/testing/testbed';
import { CdkDropListHarness } from '@angular/cdk/drag-drop/testing';
import { describe, it, expect, beforeEach } from 'vitest';

import { WatchlistComponent } from './watchlist.component';

describe('WatchlistComponent', () => {
  it('moves the dragged ticker to the target index', async () => {
    const fixture = TestBed.createComponent(WatchlistComponent);
    const loader = TestbedHarnessEnvironment.loader(fixture);
    fixture.detectChanges();

    const list = await loader.getHarness(CdkDropListHarness);
    const items = await list.getItems();
    await items[0].drop(items[2]);

    const tickers = Array.from(
      fixture.nativeElement.querySelectorAll('.ticker'),
      (el: Element) => el.textContent?.trim(),
    );
    expect(tickers).toEqual(['MSFT', 'GOOG', 'AAPL', 'AMZN', 'NVDA']);
  });
});
```

`drop(target)` simulates grabbing one item and releasing on another. The harness works in Vitest's jsdom environment and under Playwright Component Testing alike.

**Keyboard simulation** covers the accessibility path that harnesses do not:

```typescript
it('reorders with Space + arrow keys', () => {
  const fixture = TestBed.createComponent(WatchlistComponent);
  fixture.detectChanges();
  const rows: HTMLElement[] = Array.from(fixture.nativeElement.querySelectorAll('[cdkDrag]'));
  const press = (key: string) => rows[0].dispatchEvent(new KeyboardEvent('keydown', { key }));

  rows[0].focus();
  press(' '); press('ArrowDown'); press('ArrowDown'); press(' ');
  fixture.detectChanges();

  expect(fixture.nativeElement.querySelectorAll('.ticker')[2].textContent).toBe('AAPL');
});
```

Real pointer-event and touch coverage belongs in Playwright ([Chapter 25](ch25-e2e-playwright.md)), where the browser fires actual pointer events and animations settle correctly. Unit, integration, and E2E layers each catch a different class of regression; skipping any one of them is how drag-drop bugs reach production.

---

## Summary

Drag-and-drop is the interaction that exposes how committed you are to real UX. The `@angular/cdk/drag-drop` package hands you the primitives -- `cdkDrag`, `cdkDropList`, `cdkDropListGroup` -- that cover every pattern FinancialApp needs: in-list sort for the watchlist, cross-list transfer for the kanban, and free drag with boundaries for the dashboard. Around those primitives you layer custom previews, ghost placeholders, drag handles, and CSS transitions to turn a working interaction into one that feels designed.

- **Prefer the CDK over HTML5 drag-drop.** Touch, keyboard, animation, and structured payloads come for free.
- **Use `moveItemInArray()` and `transferArrayItem()` with signal copies.** The helpers mutate in place; signals need a new reference to notify.
- **Wire `cdkDropListGroup` for multi-list transfers.** It is the single line of markup that turns a sort-only UI into a kanban.
- **Constrain free drag with `cdkDragBoundary`.** Let users arrange, but not escape.
- **Animate the siblings, not just the dragged item.** A transition on `.cdk-drop-list-dragging .cdk-drag` is what makes reorders feel smooth.
- **Announce drops via `LiveAnnouncer`.** The CDK ships keyboard support; meaningful announcements are your job.
- **Persist optimistically and debounce writes.** Snapshot, mutate, save in the background, roll back on failure.
- **Test at three layers.** Harness-based sort assertions, keyboard event simulations, and Playwright E2E for real pointer behavior.

In the next chapter we turn to virtualized scrolling -- the companion technique for rendering thousands of watchlist rows, transactions, or chart points without overwhelming the DOM.
