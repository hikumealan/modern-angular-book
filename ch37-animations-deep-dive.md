# Chapter 37: Angular Animations Deep Dive

A transfer confirmation appears without warning. A transaction row vanishes the moment it is saved. The portfolio page snaps to a new route as if nothing came before it. Each of these is technically correct -- the state changed, the DOM updated -- but the user is left briefly disoriented, scanning the screen to figure out what just happened.

Animation, used well, turns a functional UI into a legible one. A fade in signals arrival. A slide out signals departure. A staggered list tells the eye "this is the same collection, just updated." In a financial application, where users are routinely asked to confirm irreversible actions on their money, these small cues carry outsized weight. They build the trust that the interface is in sync with the user, not ahead of them and not lagging behind.

This chapter goes deep on Angular's animation system: composing non-trivial triggers, choreographing lists, animating route transitions, responding to animation lifecycle callbacks, honoring `prefers-reduced-motion`, staying on the compositor, falling back to the Web Animations API when the DSL runs out, and testing all of it without waiting real milliseconds.

> **Extends:** [Chapter 27](ch27-material-design-system.md) introduced animations for design-system components. This chapter goes deeper into route transitions, stagger patterns, callbacks, and reduced-motion accessibility.

---

## Composing Triggers Beyond a Single Transition

The `fadeIn` and `dialogEnterLeave` triggers from [Chapter 27](ch27-material-design-system.md) each handle one or two transitions. Real component states are messier. A transaction row can be `pending`, `cleared`, or `reverted`. A sync banner can be `idle`, `syncing`, `success`, or `error`. Each state combination may want its own animation. Angular's DSL composes cleanly up to that complexity, but only if you understand what the building blocks actually do.

### States, Wildcards, and the Void Alias

`state()` binds a style set to a named value. `transition()` describes how to animate between two states. State names are matched exactly, but two aliases cover the common cases: `*` is the wildcard matching any state, and `void` is the state of an element that is not in the DOM. The `:enter` alias is shorthand for `void => *` and `:leave` is shorthand for `* => void`.

```typescript
// libs/shared/design-system/src/lib/animations/_animations.ts
export const syncBanner = trigger('syncBanner', [
  state('idle',    style({ height: '0', opacity: 0, overflow: 'hidden' })),
  state('syncing', style({ height: '*', opacity: 1, background: 'var(--fin-color-surface-variant)' })),
  state('success', style({ height: '*', opacity: 1, background: 'var(--fin-color-success-container)' })),
  state('error',   style({ height: '*', opacity: 1, background: 'var(--fin-color-error-container)' })),

  transition('idle => syncing', animate('200ms cubic-bezier(0, 0, 0, 1)')),
  transition('syncing => success', animate('250ms cubic-bezier(0.2, 0, 0, 1)')),
  transition('syncing => error',   animate('250ms cubic-bezier(0.2, 0, 0, 1)')),
  transition('success => idle',    animate('400ms 2s cubic-bezier(0.3, 0, 1, 1)')),
  transition('error => idle',      animate('250ms cubic-bezier(0.3, 0, 1, 1)')),
]);
```

Four states, five transitions, no wildcards. Each transition is explicit about which pair it handles, and the `400ms 2s` syntax on `success => idle` encodes a two-second dwell before the banner collapses -- enough time for the user to read "Sync complete" before it disappears.

The wildcard is useful when several source states should animate the same way:

```typescript
export const transactionRowState = trigger('transactionRowState', [
  state('pending',  style({ opacity: 0.6, fontStyle: 'italic' })),
  state('cleared',  style({ opacity: 1,   fontStyle: 'normal' })),
  state('reverted', style({ opacity: 0.4, textDecoration: 'line-through' })),
  transition('* => cleared', animate('250ms cubic-bezier(0.2, 0, 0, 1)')),
  transition('* => reverted', animate('150ms cubic-bezier(0.3, 0, 1, 1)')),
  transition(':enter', [style({ opacity: 0 }), animate('200ms', style({ opacity: 1 }))]),
  transition(':leave', [animate('150ms', style({ opacity: 0 }))]),
]);
```

The `:enter` and `:leave` rules are essential whenever the element is rendered conditionally. Without them, a row removed by `@for` from a filtered transaction list would disappear instantly, with no visual continuity between the list's before and after. Bind `[@syncBanner]="state()"` in the template to any signal that returns a state name; Angular picks the matching transition when the signal changes, with no manual orchestration required.

---

## Animating Lists with `query` and `stagger`

Animating a collection as a whole is a different problem from animating one element. When FinancialApp loads a filtered transaction list, dropping fifty rows into the DOM simultaneously creates a visual flash; the eye cannot latch onto any one row. Staggering the entries -- each row enters 40ms after the previous -- makes the list feel deliberate, and gives a reassuring sense of "this is the data, arriving."

`query()` selects descendant elements within the animated container. `stagger()` offsets each queried element's animation by a fixed delay.

### Transaction List Enter Stagger

```typescript
// libs/shared/design-system/src/lib/animations/_animations.ts
export const transactionListStagger = trigger('transactionListStagger', [
  transition('* => *', [
    query(':enter', [
      style({ opacity: 0, transform: 'translateY(12px)' }),
      stagger(40, [
        animate('250ms cubic-bezier(0.2, 0, 0, 1)',
          style({ opacity: 1, transform: 'translateY(0)' })),
      ]),
    ], { optional: true }),
    query(':leave', [
      stagger(-20, [
        animate('150ms cubic-bezier(0.3, 0, 1, 1)',
          style({ opacity: 0, transform: 'translateY(-8px)' })),
      ]),
    ], { optional: true }),
  ]),
]);
```

```html
<!-- apps/financial-app/.../transaction-list.component.html -->
<ul [@transactionListStagger]="transactions().length">
  @for (tx of transactions(); track tx.id) {
    <li>{{ tx.date | date:'shortDate' }} -- {{ tx.description }}
        <strong>{{ tx.amount | currency }}</strong></li>
  }
</ul>
```

Two details matter. First, `{ optional: true }` prevents "no elements matched" errors when the list is empty or when only one side (enter or leave) has items. Second, the bound expression is `transactions().length` -- a primitive number. Every length change triggers the `* => *` transition. Binding to the array itself would also work (Angular compares references), but binding to a scalar is cheaper and communicates intent.

The negative stagger on `:leave` is a common trick: it reverses the cascade so the bottom rows exit first, creating a subtle "fold up" effect that reads differently from the "unfold down" of entry. Small detail, disproportionate impact on polish.

The same pattern adapts to FinancialApp's receipt upload flow. Multiple file previews attach at once; a `stagger(60, ...)` with a scale-and-fade on `:enter` replaces a confusing simultaneous flash with a readable sequence. The 60ms offset is wider than the 40ms for transactions because thumbnails are larger and more visually complex -- each one deserves a slightly longer breath before the next arrives.

---

## Route Transitions

A navigation from `/accounts` to `/portfolio` swaps the entire main region in one synchronous tick. Without an animation, the visible change is abrupt. With one, it becomes a controlled handoff -- the old page acknowledges its departure, the new page announces its arrival, and the user's gaze has a few hundred milliseconds to follow.

### Trigger Setup

The animation trigger lives at the app level, wrapping the `<router-outlet>`. Angular exposes the outlet's currently activated route through `outlet.activatedRouteData`, which is the canonical hook for animation state:

```typescript
// libs/shared/design-system/src/lib/animations/route-animations.ts
import { trigger, transition, style, animate, query, group } from '@angular/animations';

export const routeCrossfade = trigger('routeCrossfade', [
  transition('* <=> *', [
    query(':enter, :leave', [
      style({ position: 'absolute', top: 0, left: 0, width: '100%' }),
    ], { optional: true }),
    query(':enter', [style({ opacity: 0 })], { optional: true }),
    group([
      query(':leave', [
        animate('200ms cubic-bezier(0.3, 0, 1, 1)', style({ opacity: 0 })),
      ], { optional: true }),
      query(':enter', [
        animate('300ms 100ms cubic-bezier(0.2, 0, 0, 1)', style({ opacity: 1 })),
      ], { optional: true }),
    ]),
  ]),
]);
```

```typescript
// apps/financial-app/src/app/app.component.ts
import { Component } from '@angular/core';
import { RouterOutlet } from '@angular/router';
import { routeCrossfade } from '@fin/design-system';

@Component({
  selector: 'app-root',
  imports: [RouterOutlet],
  animations: [routeCrossfade],
  template: `
    <fin-nav-toolbar />
    <main class="route-container">
      <div [@routeCrossfade]="getAnimationState(outlet)">
        <router-outlet #outlet="outlet" />
      </div>
    </main>
  `,
  styles: `
    .route-container { position: relative; overflow: hidden; min-height: 400px; }
  `,
})
export class AppComponent {
  protected getAnimationState(outlet: RouterOutlet): string | null {
    return outlet?.isActivated ? outlet.activatedRouteData?.['animation'] ?? 'default' : null;
  }
}
```

Two CSS rules carry the structural load. `position: absolute` on both entering and leaving elements lets them occupy the same space briefly, enabling the crossfade. `overflow: hidden` on the container hides the transient overflow of directional slides (next section). Without these, the leaving view would shift the entering view down as it disappears, creating a jarring layout jolt.

### Per-Route Animation Data

The `getAnimationState` helper reads a string from the route's `data` object. Add that metadata in your route configuration and you can describe animations declaratively per route:

```typescript
// apps/financial-app/src/app/app.routes.ts
export const appRoutes: Routes = [
  {
    path: 'accounts',
    loadComponent: () => import('./features/accounts/accounts.page'),
    data: { animation: 'slideRight' },
  },
  {
    path: 'portfolio',
    loadComponent: () => import('./features/portfolio/portfolio.page'),
    data: { animation: 'slideLeft' },
  },
  {
    path: 'settings/:section',
    loadComponent: () => import('./features/settings/settings.page'),
    data: { animation: 'slideOver' },
  },
];
```

The trigger inspects the state names and picks an animation accordingly. A `routeSlide` trigger differentiates direction by matching transitions like `'slideRight <=> slideLeft'` (symmetric horizontal slide) and `'* => slideOver'` (the outgoing view stays put while settings slides in from the right). Giving directional metadata to the router makes these distinctions easy to express and easy to change later, without touching component templates.

---

## Animation Callbacks

Angular animations emit two lifecycle events on the host element: `(@triggerName.start)` and `(@triggerName.done)`. The parameter is an `AnimationEvent` with `fromState`, `toState`, `totalTime`, and `element`. These callbacks unlock patterns that would otherwise require fragile timers.

### Deferring Heavy Work Until After Animation

When the portfolio's detailed breakdown expands, the animation runs for 300ms. During those 300ms, the main thread has to animate smoothly -- running a layout-heavy chart render on top will drop frames. The fix is to defer the chart initialization until the animation finishes:

```typescript
@Component({
  selector: 'fin-portfolio-breakdown',
  template: `
    <div [@expandTrigger]="state()"
         (@expandTrigger.done)="onExpandDone($event)">
      @if (chartReady()) {
        <fin-holdings-chart [holdings]="holdings()" />
      } @else {
        <div class="chart-placeholder" aria-hidden="true"></div>
      }
    </div>
  `,
})
export class FinPortfolioBreakdownComponent {
  readonly state = signal<'collapsed' | 'expanded'>('collapsed');
  protected readonly chartReady = signal(false);

  protected onExpandDone(event: AnimationEvent): void {
    if (event.toState === 'expanded') this.chartReady.set(true);
  }
}
```

The chart renders once the container is fully expanded, so the expansion animation is never competing for CPU with chart layout. Call it "animate first, compute second" -- a common enough pattern to name.

### Analytics on Animation Completion

Animation callbacks double as honest UX analytics signals. A "Transaction saved" toast that animates in and then out tells you nothing useful if you log the save event -- the user may have already navigated away. But if you log on `(@toastSlideIn.done)` specifically when `event.toState === 'void' && event.fromState === 'visible'`, you know the toast completed its full display cycle. The `event.totalTime` property tells you exactly how long the user saw it, which is more actionable than any approximation a `setTimeout` could produce.

---

## Sequences, Groups, and Keyframes

Simple animations use a single `animate()` step. Complex choreography -- where multiple properties move with different timing relative to each other -- uses `sequence()`, `group()`, and `keyframes()` as composition primitives.

- **`sequence()`** runs animations one after another, even when they appear in what would otherwise be a parallel group.
- **`group()`** runs animations in parallel, even when they appear inside a sequence.
- **`keyframes()`** defines multiple style snapshots within a single `animate()` step, letting one step interpolate through several intermediate positions.

### Expanding Transaction Card

A transaction card expands to reveal receipts, notes, and categorization controls. The choreography: the card height grows, then the content fades in, then the action buttons slide up from below. Three phases, coordinated precisely:

```typescript
export const expandTransactionCard = trigger('expandTransactionCard', [
  state('collapsed', style({ height: '80px' })),
  state('expanded',  style({ height: '*' })),
  transition('collapsed => expanded', [
    sequence([
      animate('250ms cubic-bezier(0.2, 0, 0, 1)', style({ height: '*' })),
      group([
        query('.card-content', [
          style({ opacity: 0, transform: 'translateY(8px)' }),
          animate('200ms', style({ opacity: 1, transform: 'translateY(0)' })),
        ], { optional: true }),
        query('.card-actions', [
          style({ opacity: 0, transform: 'translateY(16px)' }),
          animate('250ms 50ms', style({ opacity: 1, transform: 'translateY(0)' })),
        ], { optional: true }),
      ]),
    ]),
  ]),
  transition('expanded => collapsed', [
    group([
      query('.card-content, .card-actions',
        [animate('150ms', style({ opacity: 0 }))], { optional: true }),
      animate('200ms 50ms', style({ height: '80px' })),
    ]),
  ]),
]);
```

The outer `sequence` makes the height animate before the content appears. The inner `group` runs both content animations in parallel, with a 50ms delay on the actions so they follow the content slightly. The collapse keeps the content fade and height collapse parallel -- symmetrical choreography reads as cheap, asymmetric as deliberate.

`keyframes()` handles animations that pass through intermediate states. An attention-grabbing shake on a failed login button needs multiple waypoints in a single step:

```typescript
export const shake = trigger('shake', [
  transition('* => error', [
    animate('400ms', keyframes([
      style({ transform: 'translateX(0)',    offset: 0 }),
      style({ transform: 'translateX(-8px)', offset: 0.25 }),
      style({ transform: 'translateX(8px)',  offset: 0.5 }),
      style({ transform: 'translateX(-4px)', offset: 0.75 }),
      style({ transform: 'translateX(0)',    offset: 1 }),
    ])),
  ]),
]);
```

The `offset` values are fractions of the total duration. Keyframes are the right tool when the animation has more than two waypoints; trying to express a shake with nested `sequence()` calls quickly becomes unreadable.

---

## Reduced Motion and Accessibility

Vestibular disorders affect roughly 1 in 20 adults. For these users, seemingly innocuous screen motion -- parallax scrolling, sliding panels, pronounced transitions -- can trigger nausea, dizziness, or migraines. The operating system exposes a user preference via the `prefers-reduced-motion` media query, and every major OS has a system setting to enable it. Our job is to listen.

The rule of thumb from [Chapter 22](ch22-accessibility-aria.md)'s accessibility principles: animation that *conveys information* should stay (perhaps in a reduced form); animation that is *decorative* should disappear when reduced motion is requested.

### CSS-Only Approach

For CSS animations and transitions, `@media (prefers-reduced-motion: reduce)` is all you need. Override durations to near-zero, disable `transform` transitions, and replace directional motion with simple opacity fades:

```scss
// libs/shared/design-system/src/styles/_reduced-motion.scss
@media (prefers-reduced-motion: reduce) {
  *, *::before, *::after {
    animation-duration: 0.01ms !important;
    animation-iteration-count: 1 !important;
    transition-duration: 0.01ms !important;
    scroll-behavior: auto !important;
  }
}
```

This is a sledgehammer, and it is appropriate as a baseline. CSS transitions from [Chapter 27](ch27-material-design-system.md) go to effectively zero duration; nothing animates, nothing shifts. But it does not reach Angular's animation DSL, which compiles to `Element.animate()` calls that bypass CSS transition rules.

### ReducedMotionService

For the Angular animation system, expose the media-query state as a signal that templates and triggers can branch on:

```typescript
// libs/shared/design-system/src/lib/a11y/reduced-motion.service.ts
@Injectable({ providedIn: 'root' })
export class ReducedMotionService {
  private readonly platformId = inject(PLATFORM_ID);
  private readonly document = inject(DOCUMENT);
  readonly prefersReducedMotion = signal(false);

  constructor() {
    if (!isPlatformBrowser(this.platformId)) return;
    const mql = this.document.defaultView!.matchMedia('(prefers-reduced-motion: reduce)');
    this.prefersReducedMotion.set(mql.matches);
    mql.addEventListener('change', e => this.prefersReducedMotion.set(e.matches));
  }
}
```

Platform-guarding via `isPlatformBrowser` keeps the service SSR-safe (see [Chapter 17](ch17-defer-ssr-hydration.md)). On the server, the signal defaults to `false`, and the real preference is picked up on hydration.

Templates branch on the signal to opt out of decorative triggers entirely:

```html
@if (reducedMotion.prefersReducedMotion()) {
  <ul>
    @for (tx of transactions(); track tx.id) { <li>...</li> }
  </ul>
} @else {
  <ul [@transactionListStagger]="transactions().length">
    @for (tx of transactions(); track tx.id) { <li>...</li> }
  </ul>
}
```

For triggers that communicate information -- a sync banner's state change, for example -- keep the binding but parameterize the duration. Define the transition as `animate('{{ duration }}ms')` with default `params`, then bind `{ value: state(), params: { duration: reducedMotion() ? 0 : 200 } }`. The animation still runs and fires its `(@done)` callbacks; it just completes instantly, preserving semantic behavior while eliminating the visual transition.

---

## Performance: Main-Thread vs Compositor Animations

Every animation runs on one of two threads. **Compositor-thread animations** change properties the GPU can handle without re-layout or re-paint: `transform` and `opacity`. **Main-thread animations** touch properties that invalidate layout or paint: `top`, `left`, `width`, `height`, `margin`, and most box-model properties. Compositor animations are essentially free at any frame rate; main-thread animations compete with your JavaScript and can easily drop from 60fps to 20fps under load.

The rule is simple, if occasionally inconvenient: prefer `transform` and `opacity`. Use `transform: translate()` instead of `left`/`top`, `transform: scale()` instead of `width`/`height`, `transform: rotate()` for any rotation.

```scss
// Bad: animates width -- forces layout on every frame
.drawer-open { width: 320px; transition: width 300ms; }

// Good: animates transform -- compositor-only
.drawer { transform: translateX(-320px); transition: transform 300ms; }
.drawer-open { transform: translateX(0); }
```

The compositor version achieves the same visual effect with dramatically better frame rates, especially on lower-end devices. This matters for FinancialApp's mobile users; a dropped frame during a transfer confirmation is the kind of small UX crack that accumulates into "this app feels janky."

### FLIP for List Reorders

Some animations inherently involve layout changes -- a list reorder after changing sort order, a grid item repositioning after a filter. You cannot avoid the layout change, but you can still animate it on the compositor using the **FLIP** technique (First, Last, Invert, Play), popularized by Paul Lewis: measure each element's position **F**irst, apply the change and measure the **L**ast position, **I**nvert the delta as a `transform` so each element appears unmoved despite its new layout, then **P**lay the transform back to zero.

A small directive wraps the pattern:

```typescript
// libs/shared/design-system/src/lib/a11y/flip.directive.ts
@Directive({ selector: '[finFlip]' })
export class FinFlipDirective<T> {
  private readonly host = inject(ElementRef<HTMLElement>);
  readonly finFlip = input.required<T>();
  private previousRect: DOMRect | null = null;

  constructor() {
    effect(() => {
      this.finFlip();
      const el = this.host.nativeElement;
      const next = el.getBoundingClientRect();
      const prev = this.previousRect;
      this.previousRect = next;
      if (!prev) return;

      const dx = prev.left - next.left;
      const dy = prev.top - next.top;
      if (dx === 0 && dy === 0) return;

      el.animate(
        [{ transform: `translate(${dx}px, ${dy}px)` }, { transform: 'translate(0, 0)' }],
        { duration: 300, easing: 'cubic-bezier(0.2, 0, 0, 1)' },
      );
    });
  }
}
```

Apply with `<li [finFlip]="sortBy()">` on each reorderable row. When `sortBy()` changes, the effect runs after the DOM update; each row plays a transform-only animation from its old position back to the new. The layout already happened; the animation is pure compositor work. This pattern scales to lists of hundreds without dropping frames.

---

## Web Animations API as an Escape Hatch

Angular's animation DSL is expressive but not universal. Some scenarios -- imperative triggers not tied to component state, animations that must coordinate with scroll position, complex physics-based motion -- are cleaner in direct Web Animations API calls. `element.animate()` returns an `Animation` object with `play()`, `pause()`, `cancel()`, `reverse()`, and a `finished` promise.

The FLIP directive is one such case. Another is a pulse fired imperatively when a "deposit received" push arrives:

```typescript
@Component({
  selector: 'fin-balance-tile',
  template: `<div #balance class="balance">{{ balance() | currency }}</div>`,
})
export class FinBalanceTileComponent {
  private readonly balanceEl = viewChild.required<ElementRef<HTMLElement>>('balance');
  private readonly reducedMotion = inject(ReducedMotionService);
  readonly balance = input.required<number>();

  pulse(): void {
    if (this.reducedMotion.prefersReducedMotion()) return;
    this.balanceEl().nativeElement.animate(
      [
        { transform: 'scale(1)',   backgroundColor: 'var(--fin-color-surface)' },
        { transform: 'scale(1.1)', backgroundColor: 'var(--fin-color-success-container)' },
        { transform: 'scale(1)',   backgroundColor: 'var(--fin-color-surface)' },
      ],
      { duration: 600, easing: 'cubic-bezier(0.2, 0, 0, 1)' },
    );
  }
}
```

There is no state transition to model, no component input to drive a trigger -- just a one-shot visual acknowledgement when the parent calls `pulse()`. The DSL would require a synthetic state and a trigger; `element.animate()` keeps the intent local. The early-return on reduced motion honors user preference the same way a parameterized trigger would.

---

## Testing Animations

Animations are awkward in tests. Real timing introduces flake; disabled animations skip the transitions entirely but also the lifecycle events your code may depend on. Angular ships `NoopAnimationsModule` precisely for this: triggers are registered, events fire, but the durations collapse to zero. Substitute it for the `provideAnimationsAsync()` bootstrap provider from [Chapter 1](ch01-getting-started.md) in every component spec that touches animated state.

```typescript
// libs/shared/ui/src/lib/toast/fin-toast.component.spec.ts
import { TestBed } from '@angular/core/testing';
import { NoopAnimationsModule } from '@angular/platform-browser/animations';
import { FinToastComponent } from './fin-toast.component';

describe('FinToastComponent', () => {
  beforeEach(() => {
    TestBed.configureTestingModule({ imports: [FinToastComponent, NoopAnimationsModule] });
  });

  it('invokes analytics after leave animation completes', async () => {
    const analytics = { track: vi.fn() };
    TestBed.overrideProvider(AnalyticsService, { useValue: analytics });
    const fixture = TestBed.createComponent(FinToastComponent);
    fixture.componentRef.setInput('message', 'Transaction saved');
    fixture.detectChanges();

    fixture.componentInstance.state.set('void');
    fixture.detectChanges();
    await fixture.whenStable();

    expect(analytics.track).toHaveBeenCalledWith('toast_dismissed',
      expect.objectContaining({ message: 'Transaction saved' }));
  });
});
```

`NoopAnimationsModule` ensures the `(@toastSlideIn.done)` event fires immediately when the state flips, so the test exercises the callback handler without waiting. Under the real module, the same test would need `fakeAsync` and `tick(200)` to advance virtual time.

For end-to-end tests (see [Chapter 25](ch25-e2e-playwright.md)), Playwright can disable animations globally:

```typescript
await page.emulateMedia({ reducedMotion: 'reduce' });
```

This runs the reduced-motion code paths automatically, which has the happy side-effect of giving reduced-motion support dedicated test coverage every time the E2E suite runs.

---

## Summary

Animation is a communication tool. Used sparingly and purposefully -- guiding attention, communicating state, smoothing discontinuity -- it makes a financial application feel trustworthy and legible. Used gratuitously or without respect for user preferences, it becomes noise at best and an accessibility barrier at worst.

- **Trigger composition** with `state()`, `transition()`, `*`, `:enter`, and `:leave` covers the full matrix of component state changes. Use explicit state names when the state set is small; wildcards when one animation handles many source states.
- **`query()` and `stagger()`** animate collections as coherent groups. Negative stagger reverses the cascade for exit animations; `{ optional: true }` keeps empty lists from throwing.
- **Route transitions** bind `[@trigger]="outlet.activatedRouteData"` on the `<router-outlet>` wrapper, with per-route `data: { animation: '...' }` providing directional metadata. Absolute positioning on `:enter` and `:leave` prevents layout jolts during the crossfade.
- **`(@trigger.start)` and `(@trigger.done)`** callbacks expose animation lifecycle. Use them to defer heavy work (chart initialization, data loads) and to log honest analytics events.
- **`sequence()`, `group()`, and `keyframes()`** compose multi-step choreography. `sequence` is serial, `group` is parallel, `keyframes` expresses multi-waypoint single-step animations.
- **Reduced motion** is a baseline accessibility requirement. A CSS sledgehammer catches CSS transitions; a `ReducedMotionService` signal lets templates and parameterized triggers branch for Angular animations.
- **Stay on the compositor** by animating `transform` and `opacity`. For unavoidable layout changes, FLIP converts a layout animation into a transform-only one.
- **`element.animate()` from the Web Animations API** is the right tool for imperative or physics-driven animations that do not fit cleanly into the DSL.
- **`NoopAnimationsModule`** runs tests with zero-duration animations but preserved lifecycle events, so callback-dependent logic can be tested synchronously.

The guiding principle: motion should answer a question the user is about to ask. "Where did that go?" "What just happened?" "Did it work?" If an animation does not answer one of those, it is decorative -- and decorative animation is the first thing to cut when the page feels slow or when the user has told their OS they would rather not see it.

> **Companion code:** Route transition module and `ReducedMotionService` in `financial-app/libs/shared/design-system/src/`; extends the existing `_animations.ts`.
