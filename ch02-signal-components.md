# Chapter 2: Signal-Based Components

Angular components have always been the basic building block of a user interface. What has changed dramatically in recent versions is *how* those components manage state and communicate. The decorator-heavy style of `@Input` and `@Output` has given way to a function-based API built on signals: `input()`, `output()`, `model()`, `signal()`, and `computed()`. Combined with the built-in control flow syntax (`@if`, `@for`, `@switch`) and the `inject()` function for dependency injection, the result is components that are more concise, more type-safe, and easier to reason about.

This chapter walks through building a transaction search feature for the FinancialApp. By the end you will have a `TransactionSearch` component that fetches data from an API, a `TransactionCard` sub-component that displays individual transactions with content projection, and a working understanding of how modern Angular components are wired together.

> **Companion code**: The completed source for this chapter lives in
> `financial-app/apps/financial-app/src/app/features/transactions/`.

---

## Creating a Signal-based Component

### Scaffolding Components with the Angular CLI

The fastest way to create a component is with the Angular CLI. From the workspace root, run:

```bash
ng generate component features/transactions/transaction-search --flat
```

The `--flat` flag places the files directly in the `transactions/` folder rather than creating a nested sub-directory. The CLI generates four files:

- `transaction-search.component.ts` -- the component class
- `transaction-search.component.html` -- the template
- `transaction-search.component.scss` -- styles
- `transaction-search.component.spec.ts` -- a skeleton test

In Angular v21 the generated component is standalone by default. There is no `standalone: true` in the decorator because the compiler treats standalone as the baseline -- you only need to opt *out* with `standalone: false` if you have a legacy module-based setup.

### Suffixes and the Updated Angular Style Guide

Starting with Angular v21, the style guide no longer requires the `.component` suffix on class names or file names. You are free to name the file `transaction-search.ts` and the class `TransactionSearch`. The CLI still generates the suffix by default, but you can suppress it:

```bash
ng generate component features/transactions/transaction-search --flat --type=
```

This book keeps the `.component` suffix in filenames for clarity, but omits the `Component` suffix from class names when the context is unambiguous. Both conventions are valid -- pick one and stay consistent within your project.

### Data Model

Before writing any component logic, define the data the component will work with. The FinancialApp uses a `Transaction` interface that mirrors the API response:

```typescript
// models/transaction.model.ts

export interface Transaction {
  id: number;
  accountId: number;
  amount: number;
  type: 'credit' | 'debit';
  category: string;
  date: string;
  description: string;
  pending: boolean;
}
```

This interface appears across multiple chapters. Keeping it in a shared `models/` directory prevents duplication and gives the compiler a single source of truth for type checking.

### Component Logic

Here is the `TransactionSearch` component class with signal-based state:

```typescript
// transaction-search.component.ts

import { Component, signal, computed } from '@angular/core';
import { FormsModule } from '@angular/forms';
import { CurrencyPipe, DatePipe } from '@angular/common';
import { Transaction } from '../../models/transaction.model';

@Component({
  selector: 'app-transaction-search',
  templateUrl: './transaction-search.component.html',
  styleUrl: './transaction-search.component.scss',
  imports: [FormsModule, CurrencyPipe, DatePipe],
})
export class TransactionSearchComponent {
  searchTerm = signal('');
  filterType = signal<'all' | 'credit' | 'debit'>('all');

  transactions = signal<Transaction[]>([
    {
      id: 1, accountId: 101, amount: 2500, type: 'credit',
      category: 'Salary', date: '2026-04-01',
      description: 'Monthly payroll deposit', pending: false,
    },
    {
      id: 2, accountId: 101, amount: 85.5, type: 'debit',
      category: 'Utilities', date: '2026-04-03',
      description: 'Electric bill payment', pending: false,
    },
    {
      id: 3, accountId: 101, amount: 42.0, type: 'debit',
      category: 'Dining', date: '2026-04-05',
      description: 'Restaurant dinner', pending: true,
    },
  ]);

  filteredTransactions = computed(() => {
    const term = this.searchTerm().toLowerCase();
    const type = this.filterType();

    return this.transactions().filter(tx => {
      const matchesTerm =
        tx.description.toLowerCase().includes(term) ||
        tx.category.toLowerCase().includes(term);
      const matchesType = type === 'all' || tx.type === type;
      return matchesTerm && matchesType;
    });
  });

  totalAmount = computed(() =>
    this.filteredTransactions().reduce((sum, tx) =>
      tx.type === 'credit' ? sum + tx.amount : sum - tx.amount, 0)
  );
}
```

A few things to notice:

- **`signal()`** creates a writable signal. Reading the value requires calling it as a function (`this.searchTerm()`), and writing requires calling `.set()` or `.update()`.
- **`computed()`** derives a value from other signals. Angular tracks which signals the computation reads and re-evaluates only when those signals change. We explore the reactive graph in depth in [Chapter 3](ch03-reactive-signals.md).
- There are no lifecycle hooks, no `OnChanges`, and no manual subscription management. The computed signals react automatically.

### Importing Dependencies for the Template

Angular v21 standalone components declare their template dependencies in the `imports` array of the `@Component` decorator. This replaces the old `NgModule` declarations array. In the example above, `FormsModule` provides `ngModel` for the search input, while `CurrencyPipe` and `DatePipe` are imported so the template can use them directly.

If you forget to add a dependency here, the compiler will produce a clear error: *"'ngModel' is not a known property of 'input'"*. This is a significant improvement over the module-based approach, where the error could originate from a completely different file.

### Template and Data Binding

The template ties together several binding mechanisms. Here is the full template for `TransactionSearch`:

```html
<!-- transaction-search.component.html -->

<div class="transaction-search">
  <h2>Transaction Search</h2>

  <div class="search-controls">
    <input
      type="text"
      placeholder="Search transactions..."
      [ngModel]="searchTerm()"
      (ngModelChange)="searchTerm.set($event)" />

    <select
      [ngModel]="filterType()"
      (ngModelChange)="filterType.set($event)">
      <option value="all">All</option>
      <option value="credit">Credits</option>
      <option value="debit">Debits</option>
    </select>
  </div>

  <p class="summary">
    Showing {{ filteredTransactions().length }} transactions
    (Net: {{ totalAmount() | currency }})
  </p>

  @if (filteredTransactions().length === 0) {
    <p class="empty-state">No transactions match your search.</p>
  } @else {
    <ul class="transaction-list">
      @for (tx of filteredTransactions(); track tx.id) {
        <li [class.pending]="tx.pending">
          <span class="description">{{ tx.description }}</span>
          <span class="category">{{ tx.category }}</span>
          <span
            class="amount"
            [class.credit]="tx.type === 'credit'"
            [class.debit]="tx.type === 'debit'">
            @switch (tx.type) {
              @case ('credit') { + }
              @case ('debit') { - }
            }
            {{ tx.amount | currency }}
          </span>
          <span class="date">{{ tx.date | date:'mediumDate' }}</span>
        </li>
      }
    </ul>
  }
</div>
```

This template demonstrates every major data-binding mechanism. The following subsections break them down.

#### Event Bindings

Event bindings use the `(event)="handler"` syntax. In the search input, `(ngModelChange)` fires whenever the user types, and the handler calls `searchTerm.set($event)` to push the new value into the signal. You can also bind to native DOM events:

```html
<button (click)="filterType.set('all')">Reset Filter</button>
```

The parentheses tell Angular to listen for the event on the left and execute the expression on the right.

#### Property Bindings

Property bindings use square brackets: `[ngModel]="searchTerm()"`. The expression on the right is evaluated and the result is assigned to the property on the left. Because `searchTerm` is a signal, you call it as a function to read its current value.

Property bindings are one-way: data flows from the component class into the template. For two-way binding with signals, see the section on `model()` later in this chapter.

#### Built-in Control Flow

Angular's built-in control flow replaces the structural directives `*ngIf`, `*ngFor`, and `*ngSwitch` from earlier versions. The new syntax is block-based and does not require importing any directive.

**`@if` / `@else`** renders content conditionally:

```html
@if (filteredTransactions().length === 0) {
  <p>No transactions found.</p>
} @else {
  <ul>...</ul>
}
```

**`@for` with `track`** iterates over a collection. The `track` expression is mandatory and tells Angular how to identify each item for efficient DOM updates:

```html
@for (tx of filteredTransactions(); track tx.id) {
  <li>{{ tx.description }}</li>
} @empty {
  <li>No results.</li>
}
```

The `@empty` block is optional and renders when the collection is empty -- a convenient alternative to a separate `@if` check.

**`@switch`** selects among discrete values:

```html
@switch (tx.type) {
  @case ('credit') { <span class="badge credit">Credit</span> }
  @case ('debit') { <span class="badge debit">Debit</span> }
  @default { <span class="badge">Unknown</span> }
}
```

These blocks are parsed at compile time, which enables better type-narrowing and smaller bundle output compared to the directive-based equivalents.

#### Interpolation and Pipes

Double-curly interpolation (`{{ expression }}`) renders a value as text. Pipes transform the value before display:

```html
{{ tx.amount | currency }}
{{ tx.date | date:'mediumDate' }}
```

In standalone components, pipes must be listed in the `imports` array just like directives. If you forget to import `CurrencyPipe`, the template compiler will tell you.

#### Conditional Styling

The `[class.name]` binding conditionally adds or removes a CSS class:

```html
<li [class.pending]="tx.pending">
<span [class.credit]="tx.type === 'credit'">
```

When the expression evaluates to `true`, the class is applied. This is often more readable than building class strings manually, especially when multiple conditions are involved.

### Calling the Component

To display `TransactionSearch` in the application shell, import it and use its selector:

```typescript
// app.component.ts

import { Component } from '@angular/core';
import { TransactionSearchComponent } from './features/transactions/transaction-search.component';

@Component({
  selector: 'app-root',
  imports: [TransactionSearchComponent],
  template: `<app-transaction-search />`,
})
export class AppComponent {}
```

The self-closing tag (`<app-transaction-search />`) is valid in Angular templates and preferred when the component has no projected content.

### Running and Debugging the Application

#### Starting the Application

From the workspace root:

```bash
ng serve
```

The development server starts on `http://localhost:4200` by default. Changes to source files trigger an incremental rebuild and the browser reloads automatically.

If you are using the Nx monorepo layout from the companion codebase, the command is:

```bash
npx nx serve financial-app
```

#### Discovering Errors in the Developer Console

Open the browser's developer tools (F12 or Cmd+Option+I on macOS) and check the **Console** tab. Angular logs detailed error messages here, including:

- Template binding errors ("Cannot read properties of undefined")
- Missing import errors ("X is not a known element")
- Expression evaluation errors with the exact line in the template

The Angular DevTools browser extension adds a **Components** tab that lets you inspect the component tree, view signal values, and trigger change detection manually.

#### Debugging the Application in the Browser

The **Sources** tab in Chrome DevTools maps directly to your TypeScript files when source maps are enabled (the default in development). Set breakpoints in your component class, step through computed signal evaluations, and inspect signal values by hovering over them.

A useful technique for signals: in the console, type `ng.getComponent($0)` after selecting an element in the Elements tab. This gives you a reference to the component instance where you can call signals directly -- for example, `ng.getComponent($0).searchTerm()` returns the current search term.

#### Debugging with Visual Studio Code

VS Code (and Cursor) can attach to the running Chrome instance for a fully integrated debugging experience. Add this configuration to `.vscode/launch.json`:

```json
{
  "type": "chrome",
  "request": "launch",
  "name": "Debug Financial App",
  "url": "http://localhost:4200",
  "webRoot": "${workspaceFolder}/apps/financial-app/src"
}
```

Press F5 to launch. Breakpoints set in your `.ts` files will pause execution directly in the editor, with full access to the call stack and variable inspection. This is particularly valuable when debugging complex computed signals that depend on multiple upstream signals.

---

## Data Access

The hardcoded transaction array served its purpose for demonstrating bindings, but a real application loads data from a server. Angular provides two mechanisms: the established `HttpClient` service and the newer `httpResource` function.

### Data Access Using the HttpClient

The `HttpClient` is Angular's long-standing HTTP abstraction. Inject it with the `inject()` function:

```typescript
import { Component, inject, signal } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Transaction } from '../../models/transaction.model';

@Component({
  selector: 'app-transaction-search',
  templateUrl: './transaction-search.component.html',
  styleUrl: './transaction-search.component.scss',
  imports: [FormsModule, CurrencyPipe, DatePipe],
})
export class TransactionSearchComponent {
  private http = inject(HttpClient);

  searchTerm = signal('');
  filterType = signal<'all' | 'credit' | 'debit'>('all');
  transactions = signal<Transaction[]>([]);
  loading = signal(false);
  error = signal<string | null>(null);

  filteredTransactions = computed(() => {
    const term = this.searchTerm().toLowerCase();
    const type = this.filterType();
    return this.transactions().filter(tx => {
      const matchesTerm =
        tx.description.toLowerCase().includes(term) ||
        tx.category.toLowerCase().includes(term);
      const matchesType = type === 'all' || tx.type === type;
      return matchesTerm && matchesType;
    });
  });

  constructor() {
    this.loadTransactions();
  }

  private loadTransactions(): void {
    this.loading.set(true);
    this.error.set(null);

    this.http.get<Transaction[]>('/api/transactions').subscribe({
      next: (data) => {
        this.transactions.set(data);
        this.loading.set(false);
      },
      error: (err) => {
        this.error.set('Failed to load transactions.');
        this.loading.set(false);
        console.error(err);
      },
    });
  }
}
```

This works, but notice how much boilerplate is involved: three separate signals for data, loading state, and error state, plus a manual `subscribe()` call with `next` and `error` callbacks. The `httpResource` function eliminates most of this.

### Data Access Using the httpResource

> **Note**: `httpResource` is an experimental API as of Angular v21. Its surface area may change in future releases. It is included here because it represents the direction Angular is heading for data fetching.

The `httpResource` function wraps an HTTP request in a `Resource` object that exposes the response value, loading status, and error state as signals:

```typescript
import { Component, signal, computed } from '@angular/core';
import { httpResource } from '@angular/common/http';
import { Transaction } from '../../models/transaction.model';

@Component({
  selector: 'app-transaction-search',
  templateUrl: './transaction-search.component.html',
  styleUrl: './transaction-search.component.scss',
  imports: [FormsModule, CurrencyPipe, DatePipe],
})
export class TransactionSearchComponent {
  searchTerm = signal('');
  filterType = signal<'all' | 'credit' | 'debit'>('all');

  transactionResource = httpResource<Transaction[]>(() => '/api/transactions');

  filteredTransactions = computed(() => {
    const transactions = this.transactionResource.value() ?? [];
    const term = this.searchTerm().toLowerCase();
    const type = this.filterType();

    return transactions.filter(tx => {
      const matchesTerm =
        tx.description.toLowerCase().includes(term) ||
        tx.category.toLowerCase().includes(term);
      const matchesType = type === 'all' || tx.type === type;
      return matchesTerm && matchesType;
    });
  });

  totalAmount = computed(() =>
    this.filteredTransactions().reduce((sum, tx) =>
      tx.type === 'credit' ? sum + tx.amount : sum - tx.amount, 0)
  );
}
```

The resource provides `.value()`, `.status()`, `.error()`, and `.isLoading()` out of the box. The template can use these directly:

```html
@if (transactionResource.isLoading()) {
  <p>Loading transactions...</p>
} @else if (transactionResource.error()) {
  <p class="error">Failed to load transactions.</p>
} @else {
  <!-- render filteredTransactions() as before -->
}
```

When the URL function references other signals, `httpResource` automatically refetches when those signals change. This is useful for parameterized queries:

```typescript
accountId = signal(101);

transactionResource = httpResource<Transaction[]>(() =>
  `/api/accounts/${this.accountId()}/transactions`
);
```

Changing `accountId` triggers a new HTTP request with no additional code. We will revisit `httpResource` and its sibling `resource()` in [Chapter 3](ch03-reactive-signals.md) when we discuss the reactive signal graph in detail.

---

## Sub Components with Properties and Events

A single monolithic component quickly becomes difficult to maintain. The standard solution is to extract presentation logic into smaller sub-components that receive data through inputs and communicate back through outputs.

### Preparations

Create the sub-component:

```bash
ng generate component features/transactions/transaction-card --flat
```

The goal is a `TransactionCard` that displays a single transaction. The parent `TransactionSearch` will render one card per item in the filtered list.

### Planning Properties and Events

Before writing code, define the component's API:

| Direction | Name | Type | Purpose |
|---|---|---|---|
| Input | `transaction` | `Transaction` | The transaction to display |
| Input | `highlighted` | `boolean` | Whether the card has a visual highlight |
| Output | `flagged` | `Transaction` | Emitted when the user flags a transaction for review |

Thinking in terms of inputs and outputs before implementation prevents the sub-component from reaching into parent state or relying on shared mutable objects.

### Components with InputSignals and OutputEmitterRefs

The `input()` and `output()` functions replace the `@Input` and `@Output` decorators:

```typescript
// transaction-card.component.ts

import { Component, input, output, computed } from '@angular/core';
import { CurrencyPipe, DatePipe } from '@angular/common';
import { Transaction } from '../../models/transaction.model';

@Component({
  selector: 'app-transaction-card',
  templateUrl: './transaction-card.component.html',
  styleUrl: './transaction-card.component.scss',
  imports: [CurrencyPipe, DatePipe],
})
export class TransactionCardComponent {
  transaction = input.required<Transaction>();
  highlighted = input(false);

  flagged = output<Transaction>();

  isCredit = computed(() => this.transaction().type === 'credit');

  formattedSign = computed(() => this.isCredit() ? '+' : '-');

  onFlag(): void {
    this.flagged.emit(this.transaction());
  }
}
```

Key points:

- **`input.required<Transaction>()`** creates a required input. Angular will throw a compile-time error if the parent does not provide it.
- **`input(false)`** creates an optional input with a default value. The type is inferred as `boolean`.
- **`output<Transaction>()`** creates an `OutputEmitterRef`. Call `.emit()` to fire the event.
- Inputs are read-only signals. You read them by calling `this.transaction()`, just like any other signal. You cannot call `.set()` on an input signal -- data flows in one direction only.
- **`computed()`** derives values from input signals. `isCredit` and `formattedSign` update automatically when the parent passes a different transaction.

### Component Template

```html
<!-- transaction-card.component.html -->

<div
  class="transaction-card"
  [class.highlighted]="highlighted()"
  [class.pending]="transaction().pending">

  <div class="card-body">
    <h3 class="description">{{ transaction().description }}</h3>
    <span class="category">{{ transaction().category }}</span>
    <span
      class="amount"
      [class.credit]="isCredit()"
      [class.debit]="!isCredit()">
      {{ formattedSign() }} {{ transaction().amount | currency }}
    </span>
    <span class="date">{{ transaction().date | date:'mediumDate' }}</span>
  </div>

  <div class="card-footer">
    <ng-content />
    <button (click)="onFlag()" class="flag-button">Flag for Review</button>
  </div>
</div>
```

Notice `<ng-content />` in the footer area -- this is a content projection slot that we will use shortly.

### Binding to a Sub-Component

In the parent template, replace the `<li>` markup with the new card component. First, add `TransactionCardComponent` to the parent's `imports` array, then update the template:

```html
<!-- transaction-search.component.html (updated) -->

@for (tx of filteredTransactions(); track tx.id) {
  <app-transaction-card
    [transaction]="tx"
    [highlighted]="tx.amount > 1000"
    (flagged)="onTransactionFlagged($event)" />
}
```

The parent component needs a handler for the `flagged` event:

```typescript
flaggedTransactions = signal<Transaction[]>([]);

onTransactionFlagged(tx: Transaction): void {
  this.flaggedTransactions.update(list => [...list, tx]);
}
```

The data flow is clear and unidirectional: `Transaction` objects flow *down* through the `[transaction]` input, and flagged events flow *up* through the `(flagged)` output.

### Content Projection

Content projection lets the parent inject arbitrary content into a designated slot in the child template. The `<ng-content />` tag in `TransactionCard` acts as that slot.

From the parent:

```html
<app-transaction-card
  [transaction]="tx"
  [highlighted]="tx.amount > 1000"
  (flagged)="onTransactionFlagged($event)">

  @if (tx.pending) {
    <span class="pending-badge">Pending</span>
  }
</app-transaction-card>
```

The `<span class="pending-badge">` is projected into the card's `<ng-content />` slot. This pattern keeps the card component generic: it does not need to know about every possible badge or annotation. The parent decides what to project based on its own context.

For components that need multiple projection slots, use named slots with the `select` attribute:

```html
<!-- child template -->
<div class="header"><ng-content select="[card-header]" /></div>
<div class="body"><ng-content /></div>

<!-- parent usage -->
<app-transaction-card [transaction]="tx">
  <span card-header>Custom Header</span>
  <p>This goes into the default slot.</p>
</app-transaction-card>
```

### Writable Inputs with ModelSignals

Sometimes a child component needs to *write back* to a value owned by the parent. The `model()` function creates a two-way-bindable signal:

```typescript
// transaction-card.component.ts (additions)

import { Component, input, output, model, computed } from '@angular/core';

export class TransactionCardComponent {
  transaction = input.required<Transaction>();
  highlighted = input(false);
  notes = model('');

  flagged = output<Transaction>();

  // ... existing computed signals and methods
}
```

Unlike `input()`, `model()` creates a writable signal. The child can call `this.notes.set('...')` or `this.notes.update(...)`, and the new value propagates back to the parent through Angular's two-way binding mechanism.

### Two-way Bindings

The `model()` signal is designed for two-way binding with the banana-in-a-box syntax `[(property)]`:

```html
<!-- parent template -->
<app-transaction-card
  [transaction]="tx"
  [(notes)]="transactionNotes">
</app-transaction-card>
```

In the child template, bind the model to an input element:

```html
<!-- transaction-card.component.html -->
<textarea
  [value]="notes()"
  (input)="notes.set($event.target.value)"
  placeholder="Add a note...">
</textarea>
```

When the user types in the textarea, `notes.set()` updates the model signal. Because the parent used `[(notes)]`, the parent's `transactionNotes` signal receives the new value automatically. The flow is bidirectional: either side can initiate a change, and the other side stays in sync.

This replaces the older pattern of combining `@Input` with `@Output` and the `Change` suffix (`notesChange`). The `model()` function handles both directions in a single declaration.

> **When to use `model()` vs. `input()` + `output()`**: Reserve `model()` for cases where the child legitimately co-owns the state -- form controls, toggles, and inline-editable fields. For most parent-to-child data flow, `input()` with a separate `output()` event provides clearer data ownership. We revisit this trade-off in [Chapter 3](ch03-reactive-signals.md).

---

## Summary

This chapter covered the core mechanics of building Angular components with the signal-based API:

- **`signal()`** creates writable, reactive state within a component.
- **`computed()`** derives values that update automatically when their upstream signals change.
- **`input()`** and **`input.required()`** define component inputs as read-only signals, replacing the `@Input` decorator.
- **`output()`** defines component outputs via `OutputEmitterRef`, replacing the `@Output` decorator and `EventEmitter`.
- **`model()`** creates writable, two-way-bindable signals for cases where parent and child share ownership of a value.
- **`inject()`** replaces constructor injection with a function call that works anywhere in the injection context.
- **Built-in control flow** (`@if`, `@for`, `@switch`) replaces structural directives with a more concise, compile-time-checked syntax.
- **`httpResource`** (experimental) wraps HTTP requests in a signal-based resource, eliminating manual loading and error state management.
- **Content projection** with `<ng-content>` lets parent components inject arbitrary content into child component slots.

The `TransactionSearch` and `TransactionCard` components demonstrate a pattern you will use throughout the FinancialApp: a container component that manages state and data access, delegating presentation to smaller sub-components that communicate exclusively through inputs and outputs. In [Chapter 3](ch03-reactive-signals.md) we extend this foundation with reactive signal patterns including `effect()`, `linkedSignal()`, and `resource()`. In [Chapter 7](ch07-testing-vitest.md) we return to these components to write unit tests with Vitest.
