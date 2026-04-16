# Chapter 10: Signal Queries & Component Communication

Angular components rarely exist in isolation. A transaction card displays projected content from its parent. A toolbar reads a list of action buttons declared inside it. A chart component needs a reference to a canvas element rendered in its own template. All of these scenarios require a component to reach beyond its own inputs and outputs to interact with the DOM or with other components.

In earlier versions of Angular, the `@ViewChild`, `@ContentChild`, `@ViewChildren`, and `@ContentChildren` decorators handled these queries. They worked, but they were fundamentally imperative -- you had to worry about lifecycle hooks, undefined values before `ngAfterViewInit`, and manual change detection when results changed. Angular v21 replaces them with signal-based query functions: `viewChild()`, `viewChildren()`, `contentChild()`, and `contentChildren()`. These functions return signals, which means the rest of the signal graph -- `computed()`, `effect()`, `linkedSignal()` -- composes naturally with query results.

This chapter explores four distinct communication patterns between components and their surroundings: content projection, parent component injection, view and content queries, template variable binding, and service-mediated coordination. Each pattern solves a different problem, and knowing when to reach for which one is what separates a well-architected component from a tangled one.

> **Companion code:** The examples in this chapter live in `financial-app/libs/shared/ui/src/transaction-card/`.

---

## Content Projection

Content projection lets a component accept arbitrary template content from its consumer and render it in designated slots. This is Angular's equivalent of the Web Components `<slot>` element, and it is the foundation of any reusable container component.

The simplest form uses a single `<ng-content>` tag. But real-world components need more structure. The FinancialApp's `TransactionCardComponent` is a good example: it needs a header area for the transaction description, a body area for details, and an optional actions area for approve/reject buttons.

Multi-slot content projection solves this with the `select` attribute:

```typescript
// transaction-card.component.ts
@Component({
  selector: 'fin-transaction-card',
  template: `
    <div class="card">
      <header class="card-header">
        <ng-content select="[cardHeader]" />
      </header>
      <div class="card-body">
        <ng-content />
      </div>
      <footer class="card-actions">
        <ng-content select="[cardActions]" />
      </footer>
    </div>
  `,
})
export class TransactionCardComponent {}
```

The consumer provides content for each slot using the attribute selectors:

```html
<fin-transaction-card>
  <h3 cardHeader>Wire Transfer -- {{ transaction().description }}</h3>

  <p>Amount: {{ transaction().amount | currency }}</p>
  <p>Date: {{ transaction().date | date:'mediumDate' }}</p>

  <div cardActions>
    <button (click)="approve()">Approve</button>
    <button (click)="reject()">Reject</button>
  </div>
</fin-transaction-card>
```

Content without a matching attribute falls into the default `<ng-content />` slot (the one without a `select`). This means the body content does not need any special attribute -- it is the catch-all.

A few rules govern multi-slot projection:

- **`select` values are CSS selectors.** You can select by attribute (`[cardHeader]`), element name (`card-header`), or CSS class (`.card-header`). Attribute selectors are the most common because they do not introduce new elements or require the consumer to know about specific CSS classes.
- **Each projected node matches at most one slot.** If a node matches multiple selectors, the first matching `<ng-content>` wins.
- **Unmatched content goes to the default slot.** If there is no default slot (no `<ng-content>` without `select`), unmatched content is silently discarded.
- **Projected content retains its original context.** Bindings like `{{ transaction().description }}` resolve in the consumer's component, not in `TransactionCardComponent`. The card does not need to know anything about transactions.

Content projection is a purely structural mechanism. The card component controls *where* projected content appears, but the consumer controls *what* that content is. This separation is what makes projection-based components genuinely reusable.

---

## Referencing Parent Components

Sometimes a child component needs to talk to a specific parent. A `TransactionFieldComponent` inside a `TransactionFormComponent` might need to register itself with the form, or a tab inside a tab group needs to announce its label.

The cleanest approach is `inject()`. A child can inject its parent component directly, provided the parent is available in the injector hierarchy:

```typescript
@Component({
  selector: 'fin-transaction-form',
  providers: [{ provide: TransactionFormComponent, useExisting: TransactionFormComponent }],
  template: `<ng-content />`,
})
export class TransactionFormComponent {
  private readonly fields = signal<TransactionFieldComponent[]>([]);

  registerField(field: TransactionFieldComponent): void {
    this.fields.update(current => [...current, field]);
  }

  unregisterField(field: TransactionFieldComponent): void {
    this.fields.update(current => current.filter(f => f !== field));
  }

  readonly allValid = computed(() => this.fields().every(f => f.valid()));
}
```

```typescript
@Component({
  selector: 'fin-transaction-field',
  template: `<label><ng-content /></label>`,
})
export class TransactionFieldComponent implements OnInit, OnDestroy {
  private readonly form = inject(TransactionFormComponent);
  readonly valid = signal(true);

  ngOnInit(): void {
    this.form.registerField(this);
  }

  ngOnDestroy(): void {
    this.form.unregisterField(this);
  }
}
```

This pattern works because Angular's dependency injection walks up the element injector tree. When `TransactionFieldComponent` asks for `TransactionFormComponent`, Angular finds it on the parent element (or any ancestor element that provides it).

Two important caveats:

1. **Tight coupling.** The child knows about the parent's concrete type. If you want the child to work with multiple parent types, introduce an abstract class or an `InjectionToken` as the contract.
2. **Optional parents.** If the child might exist outside the parent, use `inject(TransactionFormComponent, { optional: true })` to avoid a runtime error.

For simple parent-child coordination -- especially when the child needs to register itself or read configuration from the parent -- `inject()` is direct and readable. For more complex cases, consider the service-based patterns discussed later in this chapter.

---

## View and Content

Signal queries are the mechanism by which a component inspects its own template (the *view*) and projected content (the *content*). Angular v21 provides four query functions, all of which return signals:

| Function | Returns | Queries |
|---|---|---|
| `viewChild()` | `Signal<T \| undefined>` | A single element/component in the component's own template |
| `viewChildren()` | `Signal<readonly T[]>` | All matching elements/components in the template |
| `contentChild()` | `Signal<T \| undefined>` | A single element/component projected via `<ng-content>` |
| `contentChildren()` | `Signal<readonly T[]>` | All matching elements/components projected via `<ng-content>` |

The "view" is what the component declares in its own template. The "content" is what a consumer projects into the component through `<ng-content>`. This distinction matters: a `TransactionCardComponent` that wants to count how many action buttons were projected into its `[cardActions]` slot uses `contentChildren()`. A component that needs a reference to a `<canvas>` element it declared in its own template uses `viewChild()`.

### Interacting with Content

The `TransactionCardComponent` can use content queries to react to what the consumer projects. Suppose the card should only render the actions footer if the consumer actually provided actions content. A `contentChild()` query on a directive makes this possible:

```typescript
@Directive({ selector: '[cardActions]' })
export class CardActionsDirective {}
```

```typescript
@Component({
  selector: 'fin-transaction-card',
  imports: [CardActionsDirective],
  template: `
    <div class="card">
      <header class="card-header">
        <ng-content select="[cardHeader]" />
      </header>
      <div class="card-body">
        <ng-content />
      </div>
      @if (hasActions()) {
        <footer class="card-actions">
          <ng-content select="[cardActions]" />
        </footer>
      }
    </div>
  `,
})
export class TransactionCardComponent {
  private readonly actions = contentChild(CardActionsDirective);
  protected readonly hasActions = computed(() => !!this.actions());
}
```

The `contentChild()` query returns a signal that emits the directive instance when present, or `undefined` when the consumer omits the `[cardActions]` content. The `computed()` signal derived from it drives the `@if` block, so the footer only renders when there are actions to show.

For multiple projected items, `contentChildren()` returns a signal of an array:

```typescript
@Component({
  selector: 'fin-transaction-list',
  template: `
    <p>Displaying {{ cards().length }} transactions</p>
    <ng-content />
  `,
})
export class TransactionListComponent {
  readonly cards = contentChildren(TransactionCardComponent);
}
```

Every time the consumer adds or removes a `<fin-transaction-card>` from the projected content, the `cards` signal updates automatically. No lifecycle hooks. No manual subscription management.

### Interacting with the View

View queries target elements and components that live in the component's own template. A common use case is interacting with a DOM element that Angular does not manage -- a chart canvas, a video player, or a third-party widget.

In the FinancialApp, a `TransactionChartComponent` might render a `<canvas>` and need a reference to it for a charting library:

```typescript
@Component({
  selector: 'fin-transaction-chart',
  template: `<canvas #chartCanvas width="600" height="300"></canvas>`,
})
export class TransactionChartComponent {
  private readonly canvas = viewChild.required<ElementRef<HTMLCanvasElement>>('chartCanvas');

  constructor() {
    effect(() => {
      const ctx = this.canvas().nativeElement.getContext('2d');
      if (ctx) {
        this.renderChart(ctx);
      }
    });
  }

  private renderChart(ctx: CanvasRenderingContext2D): void {
    // charting logic
  }
}
```

The `.required` variant asserts that the element will always be present. If the template guarantees the element exists (no `@if` wrapping it), `viewChild.required()` eliminates the `undefined` from the signal's type. If the element might not exist -- because it is inside a conditional block -- use the standard `viewChild()` and handle `undefined`.

For querying multiple view children:

```typescript
@Component({
  selector: 'fin-transaction-dashboard',
  template: `
    <fin-transaction-card
      [transaction]="pending()"
    />
    <fin-transaction-card
      [transaction]="recent()"
    />
  `,
})
export class TransactionDashboardComponent {
  readonly allCards = viewChildren(TransactionCardComponent);

  readonly cardCount = computed(() => this.allCards().length);
}
```

### Questioning the Use of viewChild and viewChildren

Signal queries are powerful, but they introduce coupling between a component and its template structure. Before reaching for `viewChild()`, ask whether a simpler pattern would suffice:

- **Need to pass data to a child?** Use `input()` signals (see [Chapter 2](ch02-signal-components.md)).
- **Need to react to a child's events?** Use `output()`.
- **Need to coordinate multiple children?** Consider a shared service (covered later in this chapter).
- **Need to call a method on a child component?** This is the one case where `viewChild()` is genuinely the right tool. But even then, consider whether the method call could be replaced with an input that the child reacts to.

The rule of thumb: **`viewChild()` and `viewChildren()` are for DOM integration and component method invocation -- the cases where declarative data flow through inputs and outputs is not sufficient.** If you find yourself using view queries to read state from a child, you are probably working against Angular's data flow model.

### Static Child Components

By default, signal queries resolve after change detection runs, which means they are available in effects and computed signals but not during component construction. In rare cases -- when the queried element is statically present in the template (no `@if`, `@for`, or `@defer` wrapping it) -- you might want earlier access.

The signal-based query functions do not have a direct `static: true` option like the old decorator API did. Instead, the signal simply resolves on the first change detection pass. For elements that are always present, `viewChild.required()` communicates this guarantee at the type level. The signal will be populated by the time any `effect()` or `computed()` first evaluates, which is sufficient for virtually all use cases.

If you find yourself needing a reference during construction (before the first change detection), that is usually a sign that the logic belongs in an `effect()` or `afterNextRender()` callback rather than in the constructor body itself.

---

## Communication via Template Variables

Angular components and directives can expose themselves as template variables using the `exportAs` metadata property. This allows templates to interact with a component's public API without the parent component needing a `viewChild()` query at all.

Consider a `TooltipDirective` used across the FinancialApp:

```typescript
@Directive({
  selector: '[finTooltip]',
  exportAs: 'finTooltip',
})
export class TooltipDirective {
  readonly visible = signal(false);

  show(): void {
    this.visible.set(true);
  }

  hide(): void {
    this.visible.set(false);
  }
}
```

A consumer can grab a reference to the directive instance using a template variable:

```html
<button
  finTooltip="Click to approve this transaction"
  #tooltip="finTooltip"
  (mouseenter)="tooltip.show()"
  (mouseleave)="tooltip.hide()"
>
  Approve
</button>
<div [class.visible]="tooltip.visible()">
  {{ tooltip.tooltipText }}
</div>
```

The `#tooltip="finTooltip"` syntax assigns the directive instance (not the DOM element) to the template variable `tooltip`. From there, the template can call methods and read signals directly.

This pattern is useful when:

- **Interaction is purely template-driven.** The parent component's TypeScript class does not need to know about the tooltip at all -- everything happens in the template.
- **You want to avoid tight coupling.** The parent component has no `viewChild()` reference and no import of the directive type. It interacts through the template variable only.
- **The directive's API is simple.** If coordination requires complex logic, a service or a `viewChild()` query gives you more control.

`exportAs` works with both directives and components. A component's `exportAs` name can differ from its selector, which is useful when a component needs to expose a simplified API for template-level interaction while keeping its full API available through injection or queries.

One limitation: template variables are scoped to the template. You cannot access a `#tooltip` variable from a parent or sibling template. If you need cross-template communication, use a service or `inject()` instead.

---

## Communication via Services

When components need to communicate but do not have a direct parent-child relationship -- or when the communication pattern is complex enough that inputs, outputs, and queries become unwieldy -- a shared service is the right tool.

The pattern is straightforward: a service holds shared state as signals, and components inject the service to read or update that state. The service acts as a mediator, decoupling the communicating components from each other.

In the FinancialApp, the transaction approval workflow involves multiple components that need to stay synchronized: a list view, a detail panel, and a notification banner. A `TransactionSelectionService` coordinates them:

```typescript
@Injectable()
export class TransactionSelectionService {
  private readonly _selectedId = signal<number | null>(null);
  readonly selectedId = this._selectedId.asReadonly();

  private readonly _transactions = signal<Transaction[]>([]);
  readonly transactions = this._transactions.asReadonly();

  readonly selectedTransaction = computed(() => {
    const id = this._selectedId();
    return id ? this._transactions().find(t => t.id === id) ?? null : null;
  });

  select(id: number): void {
    this._selectedId.set(id);
  }

  clearSelection(): void {
    this._selectedId.set(null);
  }

  updateTransactions(transactions: Transaction[]): void {
    this._transactions.set(transactions);
  }
}
```

Notice that the service is `@Injectable()` without `providedIn: 'root'`. This is deliberate. The service is provided at the route level, scoping its lifetime to the transaction feature:

```typescript
export const transactionRoutes: Routes = [
  {
    path: '',
    providers: [TransactionSelectionService],
    children: [
      { path: '', component: TransactionListComponent },
      { path: ':id', component: TransactionDetailComponent },
    ],
  },
];
```

Both `TransactionListComponent` and `TransactionDetailComponent` inject the same instance of `TransactionSelectionService`. The list calls `select()` when a row is clicked; the detail panel reads `selectedTransaction()` to display the current selection. Neither component knows the other exists.

This pattern scales in ways that direct component communication does not:

- **Adding a new consumer** (say, a transaction sidebar) requires zero changes to existing components. The sidebar simply injects the service and reads from it.
- **Testing is straightforward.** Each component can be tested in isolation by providing a mock service.
- **State logic is centralized.** Validation, derived state, and side effects live in the service, not scattered across components.

The trade-off is indirection. When communication is simple -- a parent passing data to a child -- an `input()` is clearer than a service. Reserve service-based communication for cases where multiple components need to share state, or where the communication topology is not strictly hierarchical.

For further patterns on structuring services and stores within your architecture, see [Chapter 8](ch08-architecture.md). For directives and structural patterns that complement the query techniques in this chapter, see [Chapter 11](ch11-directives-templates.md).

---

## Summary

Component communication in Angular v21 is built on a small set of composable primitives. Each one solves a specific problem:

- **Content projection** with `<ng-content select="...">` lets container components accept and arrange arbitrary content from their consumers. Multi-slot projection using attribute selectors creates structured layouts without coupling the container to what it displays.

- **Parent injection** via `inject()` gives child components direct access to an ancestor component. This is useful for registration patterns (a field registering with a form) but introduces tight coupling that should be managed through abstractions when flexibility is needed.

- **Signal queries** -- `viewChild()`, `viewChildren()`, `contentChild()`, and `contentChildren()` -- replace the old decorator-based queries with reactive signals. View queries target the component's own template; content queries target projected content. Both integrate naturally with `computed()` and `effect()`.

- **Template variables** with `exportAs` enable template-only interaction with a component or directive's public API. This avoids the need for `viewChild()` references when the coordination is entirely declarative.

- **Shared services** decouple components that do not have a direct template relationship. By holding state as signals and providing the service at the appropriate scope (route, component, or root), services enable communication patterns that scale beyond parent-child hierarchies.

The guiding principle: start with the simplest mechanism that works. Use `input()` and `output()` for direct parent-child communication (see [Chapter 2](ch02-signal-components.md)). Reach for content projection when a component needs to be a generic container. Use signal queries when you need DOM access or must call a method on a child. Introduce a service when multiple unrelated components need shared state. Each step up in complexity should be motivated by a concrete need, not by anticipation.
