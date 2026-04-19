# Appendix A: TypeScript Essentials for Angular

Angular is written in TypeScript, and every example in this book assumes a working knowledge of the language. If you arrive from vanilla JavaScript, another typed language, or simply want a refresher on the features the rest of the book relies on, this appendix is the place to start. It is not a complete TypeScript reference -- it is the subset you need to read, write, and debug Angular v21 code comfortably. When you need more depth on advanced features (branded types, mapped types, template literal types, generics in generic service patterns), see [Chapter 24](ch27-advanced-typescript-openapi.md).

The examples throughout use the FinancialApp domain types (`Account`, `Transaction`, `Portfolio`, `Holding`, `Client`) so what you learn here transfers immediately to the rest of the book.

---

## Why Angular Is Built in TypeScript

TypeScript is a strict superset of JavaScript that adds a static type system, compile-time checks, and modern language features (decorators, private fields, optional chaining) that are either missing or unstable in raw JavaScript. Every valid JavaScript program is a valid TypeScript program, but TypeScript rejects programs whose types do not agree -- catching bugs before they reach the browser.

Angular uses TypeScript for three reasons that matter in day-to-day development:

- **Autocompletion and refactoring in editors.** Because types are known at edit time, VS Code can rename symbols across files, suggest properties on a value, and warn you when a template binds to a field that does not exist.
- **Template type checking.** Angular's compiler (`ngc`) type-checks expressions inside HTML templates using the same TypeScript engine. A typo like `{{ account.blance }}` becomes a compile error, not a silent `undefined`.
- **Safer dependency injection and signals.** `inject(HttpClient)` returns `HttpClient`. `input<Account>()` produces an `InputSignal<Account>`. The types flow through decorators, generics, and signal utilities so that wrong usage is caught immediately.

TypeScript compiles to plain JavaScript -- the runtime knows nothing about types. The cost is entirely paid at build time.

---

## Types

TypeScript's type system annotates values with information about their shape. You can declare types explicitly, or let the compiler infer them.

### Primitives

```typescript
const balance: number = 1_234.56;
const accountName: string = 'Primary Checking';
const isPending: boolean = true;
const ticker: symbol = Symbol('AAPL');
const missing: null = null;
const notSet: undefined = undefined;
```

Most of the time you omit the annotation and let inference do the work:

```typescript
const balance = 1_234.56;
const accountName = 'Primary Checking';
```

The compiler infers `number` and `string` from the initializers. Use explicit annotations when inference is wrong or ambiguous, or when the value is declared without an initializer.

### Arrays and Tuples

Arrays have two interchangeable syntaxes:

```typescript
const tickers: string[] = ['AAPL', 'MSFT', 'GOOG'];
const amounts: Array<number> = [1000, 2500, 750];
```

Prefer `T[]` for simple element types and `Array<T>` when the element type itself contains complex syntax. Both produce identical types.

Tuples are fixed-length arrays with known element types at each position:

```typescript
type AccountBalance = [number, 'USD' | 'EUR' | 'GBP'];

const primary: AccountBalance = [5432.10, 'USD'];
const savings: AccountBalance = [15_000, 'EUR'];
```

### Literal Types and Union Types

A literal type accepts only one specific value. Combined with `|` (union), literal types express enumerations without the `enum` keyword:

```typescript
type AccountType = 'checking' | 'savings' | 'investment';

const type: AccountType = 'checking';
```

TypeScript's control-flow analysis narrows unions inside conditional branches:

```typescript
function describe(type: AccountType): string {
  if (type === 'checking') {
    return 'Primary spending account';
  }
  if (type === 'savings') {
    return 'Emergency or goal savings';
  }
  return 'Investment portfolio';
}
```

If you add a new member to `AccountType`, TypeScript flags every `switch` or `if` chain that does not handle it -- provided you use the `never` exhaustiveness trick covered in Chapter 24.

### `unknown` vs `any`

`any` disables type checking for a value. `unknown` keeps it, but requires an explicit narrowing before use. Always prefer `unknown` when you truly do not know the shape (JSON payloads, event handlers, untyped third-party APIs).

```typescript
function parseAccount(raw: unknown): Account {
  if (
    typeof raw === 'object' &&
    raw !== null &&
    'id' in raw &&
    typeof (raw as Record<string, unknown>).id === 'number'
  ) {
    return raw as Account;
  }
  throw new Error('Invalid account payload');
}
```

Using `any` here would silence the compiler entirely; using `unknown` forces you to prove the shape first. Zod (covered in [Chapter 24](ch27-advanced-typescript-openapi.md)) automates this kind of runtime validation.

---

## Interfaces vs Type Aliases

You have two ways to name a structural type: `interface` and `type`.

```typescript
interface Account {
  id: number;
  name: string;
  type: 'checking' | 'savings' | 'investment';
  balance: number;
  currency: string;
  ownerId: number;
}

type Transaction = {
  id: number;
  accountId: number;
  amount: number;
  type: 'credit' | 'debit';
  category: string;
  date: string;
  description: string;
  pending: boolean;
};
```

For object shapes the two are nearly interchangeable. Use `interface` when you want to:

- Express a contract that classes can implement (`class FooService implements Greeter`).
- Declare a type that may be extended by consumers across module boundaries (declaration merging).

Use `type` aliases for everything else -- union types, tuples, intersections, and mapped types all require `type`:

```typescript
type PaidTransaction = Transaction & { pending: false };
type TransactionId = Transaction['id'];
type AnyId = Account['id'] | Transaction['id'];
```

The FinancialApp codebase uses `interface` for core entities in `libs/shared/models` and `type` for helpers and unions elsewhere.

### Optional and Readonly Properties

Mark a property optional with `?`. The resulting type includes `undefined`:

```typescript
interface TransactionDraft {
  amount: number;
  category?: string;
  note?: string;
}
```

Mark a property `readonly` when callers should not reassign it after construction:

```typescript
interface Account {
  readonly id: number;
  balance: number;
}
```

`readonly` is a compile-time check. It does not produce a runtime-immutable object (use `Object.freeze()` for that), but it is usually sufficient to prevent accidental mutations across your own codebase.

---

## Classes

TypeScript classes build on the ES2022 class syntax and add visibility modifiers, parameter properties, and `readonly` fields.

### Properties and Methods

```typescript
class PortfolioCalculator {
  private totalValue = 0;

  add(holding: Holding): void {
    this.totalValue += holding.shares * holding.currentPrice;
  }

  getTotal(): number {
    return this.totalValue;
  }
}
```

`private` means "only this class can access it." TypeScript enforces this at compile time. For true runtime privacy use the ECMAScript `#` prefix:

```typescript
class PortfolioCalculator {
  #totalValue = 0;

  add(holding: Holding): void {
    this.#totalValue += holding.shares * holding.currentPrice;
  }
}
```

Fields that Angular templates need to read must not be ECMAScript private (`#field`) -- templates cannot access them. Use TypeScript `private` plus `readonly`, or `protected` if the field is read by subclass templates, as the Angular Style Guide recommends ([Chapter 31](ch16-style-guide-governance.md)).

### Constructors and Parameter Properties

A constructor initializes instance state. TypeScript lets you declare fields inline as constructor parameters -- called *parameter properties*:

```typescript
class Account {
  constructor(
    public readonly id: number,
    public name: string,
    public balance: number,
  ) {}
}

const primary = new Account(1, 'Primary Checking', 5432.10);
```

This is equivalent to declaring the fields separately and assigning them in the constructor body. Parameter properties are useful in small value classes but are mostly superseded in Angular v21 by the `inject()` function, which does not need a constructor at all:

```typescript
@Component({ /* ... */ })
export class AccountListComponent {
  private accounts = inject(AccountsService);
}
```

### Inheritance

TypeScript supports single-class inheritance with `extends` and multiple-interface implementation with `implements`:

```typescript
interface Taxable {
  computeTax(): number;
}

abstract class Holding {
  constructor(public ticker: string, public shares: number) {}
  abstract marketValue(): number;
}

class StockHolding extends Holding implements Taxable {
  constructor(ticker: string, shares: number, public currentPrice: number) {
    super(ticker, shares);
  }

  override marketValue(): number {
    return this.shares * this.currentPrice;
  }

  computeTax(): number {
    return this.marketValue() * 0.15;
  }
}
```

The `override` keyword makes overrides explicit -- if the base class renames `marketValue()`, the compiler flags the override immediately. The Angular style guide recommends `override` on all overridden methods including lifecycle hooks.

### Composition Over Inheritance

Deep class hierarchies rarely survive contact with a real codebase. Prefer composition -- small classes or services that collaborate -- over inheritance. Angular's dependency injection makes composition effortless:

```typescript
@Injectable({ providedIn: 'root' })
export class PortfolioService {
  private prices = inject(MarketPriceService);
  private tax = inject(TaxService);

  totalAfterTax(holdings: Holding[]): number {
    const gross = holdings.reduce((sum, h) => sum + this.prices.valueOf(h), 0);
    return gross - this.tax.computeFor(gross);
  }
}
```

Each responsibility lives in its own service. No inheritance required.

---

## Functions

TypeScript functions are ordinary JavaScript functions with optional parameter and return-type annotations.

### Function Declarations and Arrow Functions

```typescript
function formatAmount(value: number, currency: string): string {
  return new Intl.NumberFormat('en-US', { style: 'currency', currency }).format(value);
}

const formatAmountArrow = (value: number, currency: string): string =>
  new Intl.NumberFormat('en-US', { style: 'currency', currency }).format(value);
```

Arrow functions have two properties that matter in Angular code:

- They do not bind their own `this`. Inside an arrow callback, `this` refers to the enclosing scope.
- They are expressions, so they can be assigned to fields or passed as arguments without wrapping in `function () { ... }`.

```typescript
@Component({ /* ... */ })
export class TransactionListComponent {
  private transactions = inject(TransactionsService);

  trackById = (_: number, txn: Transaction): number => txn.id;
}
```

`trackById` is an arrow so the `this` reference -- if it used one -- would still point to the component. A regular function would lose that binding when passed through Angular's `@for` track expression.

### Optional, Default, and Rest Parameters

```typescript
function logTransfer(
  amount: number,
  currency = 'USD',
  note?: string,
  ...tags: string[]
): void {
  console.log(amount, currency, note, tags);
}

logTransfer(100);
logTransfer(250, 'EUR');
logTransfer(500, 'GBP', 'Rent');
logTransfer(500, 'GBP', 'Rent', 'housing', 'recurring');
```

Optional parameters (`note?`) must come after all required parameters. Default parameters (`currency = 'USD'`) are implicitly optional.

### Template Literals

Template literals replace string concatenation with embedded expressions:

```typescript
const account: Account = /* ... */;
const summary = `${account.name}: ${account.balance.toFixed(2)} ${account.currency}`;
```

They also support multi-line strings and *tagged templates*, which let you pre-process the interpolated parts. Tagged templates underpin some Angular utilities (sanitization helpers, HTML/CSS template strings in other frameworks) but are rarely written in application code.

---

## Generics

Generics parametrize a function, class, or type with another type. The built-in `Array<T>` is the simplest example. Angular itself uses generics extensively:

```typescript
const primary: InputSignal<Account> = input.required<Account>();
const response$: Observable<Transaction[]> = httpClient.get<Transaction[]>('/api/transactions');
```

The `<T>` syntax lets the caller pick the concrete type. Inside the generic definition you can constrain `T`:

```typescript
function sortById<T extends { id: number }>(items: T[]): T[] {
  return [...items].sort((a, b) => a.id - b.id);
}
```

`T extends { id: number }` says "`T` must have at least a numeric `id` property." The function works on `Account`, `Transaction`, `Client`, or any other entity with an `id`.

This is the quick intro. [Chapter 24](ch27-advanced-typescript-openapi.md) covers generic service patterns, mapped types, template literal types, and the `satisfies` operator.

---

## Utility Types and Type Modifiers

A handful of built-in utility types and modifiers come up constantly in Angular code.

### `readonly`, `as const`, Non-Null Assertion

`readonly` on a property or array prevents mutation at the type level:

```typescript
interface Account {
  readonly id: number;
  balance: number;
}

function credits(account: Account): readonly number[] {
  return [account.balance, account.balance * 1.05];
}
```

The caller of `credits` receives a `readonly number[]` and cannot push or splice.

`as const` freezes a literal expression into the most specific type possible:

```typescript
const accountTypes = ['checking', 'savings', 'investment'] as const;

type AccountType = (typeof accountTypes)[number];
```

Without `as const`, `accountTypes` would have type `string[]`. With it, the type is `readonly ['checking', 'savings', 'investment']`, and `AccountType` becomes the union `'checking' | 'savings' | 'investment'`. This pattern keeps a list of constants and its type in sync from a single source.

The non-null assertion `!` tells the compiler "trust me, this is not null or undefined":

```typescript
const input = document.querySelector('input')!;
input.focus();
```

Use `!` sparingly. It silences the compiler but does nothing at runtime. When Angular queries can legitimately resolve to `undefined` (during initial render, for example), prefer `viewChild.required()` over `viewChild()!.`

### Optional Chaining and Nullish Coalescing

Optional chaining (`?.`) short-circuits property access when any link in the chain is `null` or `undefined`:

```typescript
const street = account?.owner?.address?.street;
```

Nullish coalescing (`??`) provides a fallback only when the left-hand side is `null` or `undefined` -- preserving other falsy values like `0` and `''`:

```typescript
const limit = settings?.transferLimit ?? 10_000;
```

These two operators are the safest way to traverse optional data from APIs, form values, or route parameters.

### Utility Types

The standard library ships several transformations. The ones that appear most often in Angular code:

- `Partial<T>` -- all properties of `T` made optional. Useful for patch operations: `const changes: Partial<Account> = { balance: 100 };`
- `Required<T>` -- all optional properties made required.
- `Readonly<T>` -- all properties `readonly`.
- `Pick<T, K>` -- a type with only the selected keys. `type AccountSummary = Pick<Account, 'id' | 'name' | 'balance'>;`
- `Omit<T, K>` -- a type with the selected keys removed. Handy for "creating" types where `id` is assigned by the server: `type NewAccount = Omit<Account, 'id'>;`
- `Record<K, V>` -- a keyed dictionary. `const holdingsByTicker: Record<string, Holding[]> = {};`

Chapter 24 goes deeper into mapped types and domain-specific utility types built on these primitives.

---

## Enums, or How to Avoid Them

TypeScript ships an `enum` keyword that generates a runtime object in addition to a type. In Angular v21 code you will almost always prefer a union of literal types:

```typescript
type TransactionStatus = 'pending' | 'cleared' | 'failed';
```

Unions compile to nothing -- they vanish at runtime, leaving only the narrowed string constants. Enums compile to objects that take up bundle space and can produce surprising values when treated as numbers. Reach for `enum` only when you need a bidirectional string-to-number lookup, which is rare in typical application code.

If you need a constant object that also produces a union type, use `as const` instead:

```typescript
export const TRANSACTION_STATUS = {
  Pending: 'pending',
  Cleared: 'cleared',
  Failed: 'failed',
} as const;

export type TransactionStatus = (typeof TRANSACTION_STATUS)[keyof typeof TRANSACTION_STATUS];
```

You keep the object reference for runtime use while the type system produces a compact literal union.

---

## Putting It Together: A Small FinancialApp Service

Here is a condensed example that exercises most of what this appendix covered: types, interfaces, generics, arrow functions, `inject()`, optional parameters, and utility types.

```typescript
import { Injectable, inject, signal } from '@angular/core';
import { HttpClient } from '@angular/common/http';

export interface Account {
  readonly id: number;
  name: string;
  type: 'checking' | 'savings' | 'investment';
  balance: number;
  currency: string;
}

export type NewAccount = Omit<Account, 'id'>;
export type AccountPatch = Partial<Omit<Account, 'id'>>;

@Injectable({ providedIn: 'root' })
export class AccountsService {
  private http = inject(HttpClient);

  private readonly state = signal<Account[]>([]);
  readonly accounts = this.state.asReadonly();

  load(): Promise<Account[]> {
    return new Promise<Account[]>((resolve, reject) => {
      this.http.get<Account[]>('/api/accounts').subscribe({
        next: (rows) => {
          this.state.set(rows);
          resolve(rows);
        },
        error: reject,
      });
    });
  }

  create(draft: NewAccount): Promise<Account> {
    return new Promise<Account>((resolve) => {
      this.http.post<Account>('/api/accounts', draft).subscribe((created) => {
        this.state.update((current) => [...current, created]);
        resolve(created);
      });
    });
  }

  update(id: Account['id'], patch: AccountPatch): void {
    this.state.update((current) =>
      current.map((a) => (a.id === id ? { ...a, ...patch } : a)),
    );
  }
}
```

Read it from the top: generics on `signal<Account[]>` flow into `accounts`, `Omit` removes the server-assigned `id` for creation, `Partial` makes every field optional for patch updates, and `inject()` pulls in `HttpClient` without a constructor. This is the shape of most service code in the book.

---

## Where to Go Next

- [Chapter 1](ch01-getting-started.md) -- set up the Angular CLI and generate the FinancialApp project.
- [Chapter 2](ch02-signal-components.md) -- build the first signal-based components and see `input()`, `output()`, and `model()` in action.
- [Chapter 24](ch27-advanced-typescript-openapi.md) -- advanced TypeScript patterns: branded types, discriminated unions, mapped types, template literal types, the `satisfies` operator, and OpenAPI-driven code generation.
- [TypeScript Handbook](https://www.typescriptlang.org/docs/handbook/intro.html) -- the authoritative reference for anything this appendix does not cover.
