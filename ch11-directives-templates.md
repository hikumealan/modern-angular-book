# Directives, Templates, and Containers

Components are the building blocks of Angular applications, but they are not the only way to attach behavior to the DOM. Directives let you extend existing elements with custom logic -- highlighting a row when a transaction exceeds a threshold, formatting a currency input as the user types, or toggling a tooltip on hover. Templates and containers take this further by giving you programmatic control over *what* renders and *when*, enabling patterns like reusable data tables and dynamic modal dialogs.

This chapter builds three artifacts for the FinancialApp: a `highlight` directive that visually flags high-value transactions, a `dataTable` structural directive that renders tabular data from a caller-provided template, and a `confirmDialog` modal that asks users to confirm fund transfers before execution. All three demonstrate how directive and template APIs keep components lean while pushing cross-cutting behavior into reusable units.

> **Companion code:** The directives live in `financial-app/src/app/shared/directives/`, and the confirm dialog in `financial-app/src/app/shared/components/confirm-dialog/`.

---

## Attribute Directives

Attribute directives attach behavior to an existing element without introducing a new DOM node. Unlike components, they have no template -- they modify the host element's appearance, behavior, or accessibility attributes. If you have built components with signals and `inject()` as described in [Chapter 2](ch02-signal-components.md), attribute directives will feel familiar: they use the same dependency injection, lifecycle, and standalone registration.

### Defining Directives

A directive is a class decorated with `@Directive`. The `selector` uses CSS attribute syntax so Angular knows which elements to attach it to:

```typescript
// financial-app/src/app/shared/directives/highlight.directive.ts
import { Directive, ElementRef, inject } from '@angular/core';

@Directive({
  selector: '[appHighlight]',
})
export class HighlightDirective {
  private el = inject(ElementRef);

  constructor() {
    this.el.nativeElement.style.backgroundColor = '#fff3cd';
  }
}
```

The `inject(ElementRef)` call gives the directive a reference to its host DOM element. Because directives are standalone by default in Angular v21, no module declaration is needed -- import it directly into whatever component uses it. Applying it is as simple as adding the attribute:

```html
<tr appHighlight>
  <td>{{ transaction.description }}</td>
  <td>{{ transaction.amount | currency }}</td>
</tr>
```

Every `<tr>` tagged with `appHighlight` gets a pale yellow background. This works, but a hardcoded color is not very useful.

### Communicating with the Environment

Directives accept inputs just like components. To make the highlight configurable, add `input()` signals and switch to `Renderer2` for DOM mutations:

```typescript
import { Directive, ElementRef, Renderer2, effect, inject, input } from '@angular/core';

@Directive({ selector: '[appHighlight]' })
export class HighlightDirective {
  private el = inject(ElementRef);
  private renderer = inject(Renderer2);

  color = input<string>('#fff3cd');
  threshold = input<number>(0);
  amount = input<number>(0);

  constructor() {
    effect(() => {
      if (this.amount() >= this.threshold()) {
        this.renderer.setStyle(this.el.nativeElement, 'backgroundColor', this.color());
      } else {
        this.renderer.removeStyle(this.el.nativeElement, 'backgroundColor');
      }
    });
  }
}
```

`Renderer2` matters for server-side rendering and testing, where direct DOM access may not be available. The `effect()` re-runs whenever any input signal changes, keeping the highlight in sync. The template passes values through bindings:

```html
<tr appHighlight [threshold]="10000" [amount]="transaction.amount" [color]="'#f8d7da'">
  <td>{{ transaction.description }}</td>
  <td>{{ transaction.amount | currency }}</td>
</tr>
```

The directive knows nothing about transactions -- it only knows about numbers and colors, making it reusable across the entire application.

### Directives and Template Variables

Directives can *export* values to the template using `exportAs`. This is useful when a directive manages state that the template needs to read:

```typescript
@Directive({
  selector: '[appCurrencyValidator]',
  exportAs: 'currencyValidator',
})
export class CurrencyValidatorDirective {
  private el = inject(ElementRef);

  isValid = computed(() => {
    const value = parseFloat(this.el.nativeElement.value);
    return !isNaN(value) && value > 0;
  });
}
```

In the template, reference the exported instance through a template variable:

```html
<input type="text" appCurrencyValidator #validator="currencyValidator" />

@if (!validator.isValid()) {
  <span class="error">Please enter a valid amount.</span>
}
```

The `#validator="currencyValidator"` syntax binds the directive instance to a template variable, making its public API available declaratively.

### Controlled DOM Manipulations

Sometimes a directive needs to add or remove DOM elements around its host. A tooltip directive illustrates the pattern:

```typescript
@Directive({ selector: '[appTooltip]' })
export class TooltipDirective {
  private el = inject(ElementRef);
  private renderer = inject(Renderer2);
  text = input.required<string>();
  private tooltipEl: HTMLElement | null = null;

  constructor() {
    this.renderer.listen(this.el.nativeElement, 'mouseenter', () => {
      this.tooltipEl = this.renderer.createElement('span');
      this.renderer.addClass(this.tooltipEl, 'tooltip');
      this.renderer.appendChild(this.tooltipEl, this.renderer.createText(this.text()));
      this.renderer.appendChild(this.el.nativeElement.parentNode, this.tooltipEl);
    });
    this.renderer.listen(this.el.nativeElement, 'mouseleave', () => {
      if (this.tooltipEl) {
        this.renderer.removeChild(this.el.nativeElement.parentNode, this.tooltipEl);
        this.tooltipEl = null;
      }
    });
  }
}
```

All DOM mutations go through `Renderer2`. This ensures the directive works in all rendering environments and gives Angular's change detection visibility into the modifications.

> **Rule of thumb:** Use `Renderer2` for creating, removing, or modifying elements. Use `ElementRef` only for *reading* element properties. Direct writes to `nativeElement` bypass Angular's abstraction layer.

---

## Code-based Content Projection

Content projection with `<ng-content>` works well for static slot-based layouts, but projected content is instantiated eagerly and cannot receive parameters from the projecting component. When you need to project content *lazily* or pass context data into it, you need `TemplateRef` and `ViewContainerRef`.

### Templates and Containers

A `<ng-template>` defines a block of HTML that Angular does *not* render by default -- an inert blueprint. To render it, you need a `ViewContainerRef`, a reference to a DOM location where Angular can insert views:

```typescript
import { Component, TemplateRef, ViewContainerRef, viewChild } from '@angular/core';

@Component({
  selector: 'app-lazy-panel',
  template: `
    <button (click)="toggle()">Toggle Content</button>
    <ng-container #outlet></ng-container>
    <ng-template #content>
      <div class="panel-body">
        <p>This content is created lazily, only when the button is clicked.</p>
      </div>
    </ng-template>
  `,
})
export class LazyPanelComponent {
  private outlet = viewChild.required('outlet', { read: ViewContainerRef });
  private content = viewChild.required<TemplateRef<unknown>>('content');
  private visible = false;

  toggle() {
    this.visible = !this.visible;
    this.visible
      ? this.outlet().createEmbeddedView(this.content())
      : this.outlet().clear();
  }
}
```

The `viewChild()` signal queries (covered in [Chapter 10](ch10-signal-queries.md)) give us typed references to both the container and the template. `createEmbeddedView()` stamps out the template and inserts it at the container's location; `clear()` removes it. The content is never instantiated until the user clicks the button.

### Passing Parameters to Templates

Templates become powerful when they accept context objects. The `createEmbeddedView()` method takes an optional context argument whose properties become template variables:

```typescript
@Component({
  selector: 'app-account-summary',
  template: `
    <ng-container #outlet></ng-container>
    <ng-template #summaryTpl let-name="accountName" let-balance="balance">
      <div class="summary-card">
        <h3>{{ name }}</h3>
        <p class="balance">{{ balance | currency }}</p>
      </div>
    </ng-template>
  `,
})
export class AccountSummaryComponent implements OnInit {
  private outlet = viewChild.required('outlet', { read: ViewContainerRef });
  private summaryTpl = viewChild.required<TemplateRef<unknown>>('summaryTpl');
  accounts = input.required<Account[]>();

  ngOnInit() {
    for (const account of this.accounts()) {
      this.outlet().createEmbeddedView(this.summaryTpl(), {
        accountName: account.name,
        balance: account.balance,
      });
    }
  }
}
```

The `let-name="accountName"` syntax binds the template variable `name` to the context object's `accountName` property. Each `createEmbeddedView()` call creates an independent view with its own context. This pattern is the foundation of structural directives: accept a template, manage a container, stamp out views with context.

---

## Structural Directives

Structural directives add, remove, or manipulate DOM elements. Angular's built-in `@if`, `@for`, and `@switch` are the most common examples, but you can create your own when built-in control flow does not fit your needs.

### Desugaring Structural Directives

The asterisk syntax (`*appDirective`) is syntactic sugar. Angular wraps the host element in an `<ng-template>` and passes it to the directive. This template:

```html
<tr *appRepeat="let item of items; trackBy: trackFn">
  <td>{{ item.name }}</td>
</tr>
```

Desugars to:

```html
<ng-template appRepeat [appRepeatOf]="items" [appRepeatTrackBy]="trackFn" let-item>
  <tr><td>{{ item.name }}</td></tr>
</ng-template>
```

The `let item` declaration becomes the template's implicit context variable (`$implicit`). The `of items` becomes an input named `appRepeatOf` -- the directive name concatenated with the keyword. This naming convention is how Angular maps the microsyntax to directive inputs.

### Implementing a Simple DataTable

Here is a `dataTable` directive that renders tabular data using a caller-provided row template -- the pattern used in FinancialApp's transaction list, where different views need different column layouts for the same data:

```typescript
// financial-app/src/app/shared/directives/data-table.directive.ts
import { Directive, TemplateRef, ViewContainerRef, effect, inject, input } from '@angular/core';

export interface DataTableContext<T> {
  $implicit: T;
  index: number;
  isEven: boolean;
}

@Directive({ selector: '[appDataTable]' })
export class DataTableDirective<T> {
  private templateRef = inject(TemplateRef<DataTableContext<T>>);
  private viewContainer = inject(ViewContainerRef);
  appDataTableOf = input.required<T[]>();

  constructor() {
    effect(() => {
      this.viewContainer.clear();
      this.appDataTableOf().forEach((item, index) => {
        this.viewContainer.createEmbeddedView(this.templateRef, {
          $implicit: item, index, isEven: index % 2 === 0,
        });
      });
    });
  }
}
```

The directive injects `TemplateRef` (the desugared `<ng-template>`) and `ViewContainerRef` (the insertion point). When input data changes, the effect clears existing views and stamps out fresh ones. Usage in a component template:

```html
<table class="transaction-table">
  <thead><tr><th>Date</th><th>Description</th><th>Amount</th></tr></thead>
  <tbody>
    <tr *appDataTable="let txn of transactions()"
        appHighlight [threshold]="10000" [amount]="txn.amount">
      <td>{{ txn.date | date:'shortDate' }}</td>
      <td>{{ txn.description }}</td>
      <td>{{ txn.amount | currency }}</td>
    </tr>
  </tbody>
</table>
```

The caller controls the row layout. The directive handles iteration. The `highlight` directive layers on top, flagging high-value rows. This composition of attribute and structural directives keeps each piece focused on a single responsibility.

### Using ViewContainerRef Directly to Display Templates

Sometimes you need finer control than the asterisk syntax provides -- inserting a template at a location that is *not* the directive's host, or inserting multiple templates into the same container. The explicit approach uses `<ng-container>` as an anchor:

```typescript
@Component({
  selector: 'app-transaction-view',
  template: `
    <button (click)="showDetails()">Show Details</button>
    <ng-container #detailsSlot></ng-container>
    <ng-template #detailsTpl let-txn>
      <section class="transaction-details">
        <h3>{{ txn.description }}</h3>
        <p>{{ txn.amount | currency }} &mdash; {{ txn.status }}</p>
      </section>
    </ng-template>
  `,
})
export class TransactionViewComponent {
  private detailsSlot = viewChild.required('detailsSlot', { read: ViewContainerRef });
  private detailsTpl = viewChild.required<TemplateRef<unknown>>('detailsTpl');
  selectedTransaction = input.required<Transaction>();

  showDetails() {
    this.detailsSlot().clear();
    this.detailsSlot().createEmbeddedView(this.detailsTpl(), {
      $implicit: this.selectedTransaction(),
    });
  }
}
```

You decide when views are created, cleared, or replaced. The `<ng-container>` produces no DOM output -- it is purely a logical anchor.

### Accessing the ViewContainerRef via Signal Queries

Earlier Angular versions required `@ViewChild` with `{ read: ViewContainerRef }`. Modern Angular replaces this with `viewChild()` signal queries, which integrate naturally with the reactivity system:

```typescript
private container = viewChild.required('outlet', { read: ViewContainerRef });
```

The `{ read: ViewContainerRef }` option tells Angular to resolve the query as a container reference rather than an element reference -- the same mechanism described in [Chapter 10](ch10-signal-queries.md), applied to container access. Because `viewChild()` returns a signal, you can use it inside `effect()`:

```typescript
constructor() {
  effect(() => {
    const container = this.container();
    container.clear();
    for (const txn of this.transactions()) {
      container.createEmbeddedView(this.rowTemplate(), { $implicit: txn });
    }
  });
}
```

This reactive approach eliminates the need for `ngAfterViewInit` lifecycle hooks. The effect runs as soon as the view query resolves and re-runs whenever input data changes.

---

## Dynamic Components

Directives and templates handle most DOM manipulation needs, but sometimes you need to create an entire component at runtime -- one not declared in any template. Modal dialogs are the classic example: the dialog component does not exist in the DOM until the user triggers an action, and it should be destroyed when dismissed.

### Modal Dialogs

FinancialApp needs a confirmation dialog for sensitive operations like fund transfers. The dialog component is deliberately generic -- inputs for content, outputs for user decisions:

```typescript
// financial-app/src/app/shared/components/confirm-dialog/confirm-dialog.component.ts
import { Component, input, output } from '@angular/core';

@Component({
  selector: 'app-confirm-dialog',
  template: `
    <div class="dialog-backdrop" (click)="cancelled.emit()">
      <div class="dialog-panel" (click)="$event.stopPropagation()">
        <h2>{{ title() }}</h2>
        <p>{{ message() }}</p>
        <div class="dialog-actions">
          <button class="btn-secondary" (click)="cancelled.emit()">Cancel</button>
          <button class="btn-primary" (click)="confirmed.emit()">Confirm</button>
        </div>
      </div>
    </div>
  `,
})
export class ConfirmDialogComponent {
  title = input('Confirm Action');
  message = input('Are you sure you want to proceed?');
  confirmed = output<void>();
  cancelled = output<void>();
}
```

It knows nothing about transfers or accounts. That generality is what makes it reusable.

### Instantiating Components via Code

To display the dialog dynamically, use `ViewContainerRef.createComponent()`. It takes a component class and returns a `ComponentRef` with access to the instance's inputs and outputs:

```typescript
// financial-app/src/app/features/transactions/transfer.component.ts
import { Component, ComponentRef, ViewContainerRef, viewChild } from '@angular/core';
import { ConfirmDialogComponent } from '../../shared/components/confirm-dialog/confirm-dialog.component';

@Component({
  selector: 'app-transfer',
  template: `
    <form (ngSubmit)="initiateTransfer()">
      <label>Recipient <input type="text" [(ngModel)]="recipient" name="recipient" /></label>
      <label>Amount <input type="number" [(ngModel)]="amount" name="amount" /></label>
      <button type="submit">Transfer Funds</button>
    </form>
    <ng-container #dialogHost></ng-container>
  `,
})
export class TransferComponent {
  private dialogHost = viewChild.required('dialogHost', { read: ViewContainerRef });
  private dialogRef: ComponentRef<ConfirmDialogComponent> | null = null;
  recipient = '';
  amount = 0;

  initiateTransfer() {
    if (this.dialogRef) return;
    this.dialogRef = this.dialogHost().createComponent(ConfirmDialogComponent);

    this.dialogRef.setInput('title', 'Confirm Transfer');
    this.dialogRef.setInput('message', `Transfer $${this.amount.toFixed(2)} to ${this.recipient}?`);

    this.dialogRef.instance.confirmed.subscribe(() => this.closeDialog(true));
    this.dialogRef.instance.cancelled.subscribe(() => this.closeDialog(false));
  }

  private closeDialog(execute: boolean) {
    if (execute) { /* delegate to TransferService */ }
    this.dialogRef?.destroy();
    this.dialogRef = null;
  }
}
```

Several things to note:

1. **No template declaration.** `ConfirmDialogComponent` does not appear in the template -- it is created entirely through code when the user clicks "Transfer Funds."
2. **`setInput()` for signal inputs.** `ComponentRef.setInput()` is the public API for setting input values on dynamically created components.
3. **Output subscription.** Outputs are `Observable`-compatible, so you subscribe directly. When `destroy()` is called, Angular tears down the component and its subscriptions.
4. **Lifecycle ownership.** The creating component owns the dialog's lifecycle -- fundamentally different from template-driven rendering where Angular manages this automatically.

For more complex scenarios, wrap the pattern in a service that creates dialogs outside any specific view hierarchy:

```typescript
@Injectable({ providedIn: 'root' })
export class DialogService {
  private appRef = inject(ApplicationRef);
  private environmentInjector = inject(EnvironmentInjector);

  open<T>(component: Type<T>, inputs?: Record<string, unknown>): ComponentRef<T> {
    const host = document.createElement('div');
    document.body.appendChild(host);
    const ref = createComponent(component, {
      hostElement: host,
      environmentInjector: this.environmentInjector,
    });
    if (inputs) {
      Object.entries(inputs).forEach(([k, v]) => ref.setInput(k, v));
    }
    this.appRef.attachView(ref.hostView);
    return ref;
  }

  close(ref: ComponentRef<unknown>) {
    this.appRef.detachView(ref.hostView);
    ref.destroy();
    ref.location.nativeElement.parentNode?.removeChild(ref.location.nativeElement);
  }
}
```

This service uses `createComponent()` with `EnvironmentInjector` to attach the dialog to the application's change detection tree and append it directly to the document body -- the same pattern behind Angular Material's `MatDialog` and CDK overlay system.

---

## Summary

Directives, templates, and containers form the lower-level toolkit beneath Angular's component model. The progression follows increasing power and responsibility:

- **Attribute directives** modify existing elements -- style, behavior, accessibility -- without altering structure.
- **Templates and containers** project and repeat content lazily, with parameterized context.
- **Structural directives** use `TemplateRef` and `ViewContainerRef` to add or remove DOM subtrees based on data.
- **Dynamic components** create full component instances outside the template via `createComponent()`.

Each level gives you more control but requires you to manage more lifecycle concerns. Prefer the simplest tool that solves your problem: use attribute directives before reaching for structural ones, and use template-driven rendering before reaching for `createComponent()`.

In the next chapter, we will see how Angular's router builds on many of these same primitives -- `ViewContainerRef`, lazy loading, and dynamic instantiation -- to manage navigation and route-level component rendering.
