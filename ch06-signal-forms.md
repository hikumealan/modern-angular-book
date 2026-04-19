# Chapter 6: Signal Forms

> **WARNING -- Experimental API.** Signal Forms shipped as a developer preview in Angular v21. The API has already been renamed multiple times (from `Control` to `Field` to the current `formField`), and further breaking changes are likely before the feature stabilizes. Every example in this chapter uses the latest known naming at the time of writing (`formField`, `formGroup`, `formRecord`, `formArray`). If you are reading this after additional releases, consult the official Angular documentation for the current API surface. Pin your Angular version and check the changelog before upgrading.

Forms are where theory meets production. Signals give components reactive state; the router gives them URLs; services give them shared data -- but none of that matters until a user can *enter* information and the application can validate, transform, and submit it. Angular's original forms modules (`FormsModule` and `ReactiveFormsModule`) served this purpose for a decade, but they were designed before signals existed and carry significant baggage: deep `Observable` hierarchies, loosely typed `FormControl<any>`, and a tight coupling between validation and the RxJS scheduling model.

Signal Forms reimagine the entire forms layer on top of the signal primitive introduced in [Chapter 3](ch03-reactive-signals.md). A form field is a `WritableSignal` with validation and metadata bolted on. A form group is a flat record of fields. Validation is synchronous by default and schema-driven when you want it. The result is a forms API that is smaller, faster to render, and fully type-safe from the model definition to the template binding.

This chapter builds two forms for the FinancialApp: a **transaction entry form** for recording debits and credits, and a **client onboarding form** with nested groups, arrays, and conditional validation. By the end you will be comfortable with every layer of the Signal Forms API.

> **Companion code:** The transaction entry form lives in
> `financial-app/apps/financial-app/src/app/domains/transactions/transaction-entry.component.ts`.
> The client onboarding form lives in
> `financial-app/apps/financial-app/src/app/domains/clients/client-onboarding.component.ts`.

---

## A First Signal Form

### Setting up the Component

Start with a simple transaction entry form. The component needs `formField` from `@angular/forms/signals` and a few standard imports:

```typescript
// transaction-entry.component.ts
import { Component } from '@angular/core';
import { formField, SignalFormDirective } from '@angular/forms/signals';
import { FormsModule } from '@angular/forms';

@Component({
  selector: 'app-transaction-entry',
  imports: [FormsModule, SignalFormDirective],
  templateUrl: './transaction-entry.component.html',
})
export class TransactionEntryComponent {
  readonly description = formField<string>('');
  readonly amount = formField<number>(0);
  readonly date = formField<string>(new Date().toISOString().slice(0, 10));
}
```

Each call to `formField()` creates a writable signal that tracks a single form value. The argument is the initial value, and the generic parameter locks the type. Unlike the legacy `FormControl`, there is no `any` escape hatch -- if you pass a `string` initializer, the field is `FormField<string>` and the compiler will reject a numeric assignment.

### Understanding the FieldTree Type

Signal Forms introduce the concept of a **field tree**. A `FieldTree` is a recursive type that describes the shape of a form at compile time:

```typescript
type FieldTree =
  | FormField<unknown>
  | { [key: string]: FieldTree }
  | FieldTree[];
```

A single `formField()` is a leaf. A plain object of fields is a group. An array of fields is -- predictably -- a form array. The entire form model is a tree of these three building blocks, and every utility function in the API is generic over `FieldTree`. This is why a group of fields does not need a special wrapper class in simple cases: any object literal whose values are `FormField` instances already satisfies the type.

### Binding to the Template

Signal Forms bind to the template with the `[formField]` directive, which replaces the legacy `[formControl]`:

```html
<!-- transaction-entry.component.html -->
<form signalForm>
  <label for="description">Description</label>
  <input id="description" [formField]="description" />

  <label for="amount">Amount</label>
  <input id="amount" type="number" [formField]="amount" />

  <label for="date">Date</label>
  <input id="date" type="date" [formField]="date" />

  <button type="submit">Record Transaction</button>
</form>
```

The `signalForm` directive on the `<form>` element opts the subtree into Signal Forms mode. Inside, each `[formField]` directive creates a two-way binding: user input writes to the signal, and programmatic signal updates write back to the DOM. Because the value is a signal, any `computed()` or `effect()` that reads `this.amount()` will re-evaluate whenever the user types.

---

## Working with Schemas

### Using Separate Schemas

Declaring fields inline works for small forms, but larger forms benefit from a schema object that separates the shape definition from the component class:

```typescript
// transaction.schema.ts
import { formField } from '@angular/forms/signals';

export function createTransactionSchema() {
  return {
    description: formField<string>(''),
    amount: formField<number>(0),
    date: formField<string>(new Date().toISOString().slice(0, 10)),
    type: formField<'debit' | 'credit'>('debit'),
    accountId: formField<string>(''),
  };
}
```

The component consumes the schema by calling the factory:

```typescript
@Component({ /* ... */ })
export class TransactionEntryComponent {
  readonly form = createTransactionSchema();
}
```

Template bindings reference the fields through the `form` object: `[formField]="form.description"`. This pattern keeps the component lean and makes the schema reusable across components and tests.

### Controlling Behavior

`formField()` accepts an options object as its second argument. The most common options control when the signal updates:

```typescript
const amount = formField<number>(0, {
  updateOn: 'blur',
});
```

The `updateOn` option mirrors the legacy API: `'change'` (default) updates on every keystroke, `'blur'` waits until the field loses focus, and `'submit'` defers until the form is submitted. Because signal updates are synchronous, choosing `'blur'` reduces the number of downstream recomputations for fields where intermediate values are meaningless -- a currency amount, for instance, is not useful until the user finishes typing.

### Debouncing

For fields where you want incremental feedback but not on every keystroke, Signal Forms support a `debounce` option:

```typescript
const description = formField<string>('', {
  debounce: 300,
});
```

The signal will not update until 300 milliseconds have elapsed since the last input event. This is particularly useful for search-as-you-type fields that trigger HTTP calls -- a pattern explored in [Chapter 3](ch03-reactive-signals.md) with raw signals.

### Validating Against Zod and Standard Schema

Signal Forms integrate with the [Standard Schema](https://github.com/standard-schema/standard-schema) specification, which means any schema library that implements the interface -- Zod, Valibot, ArkType -- works out of the box. For the FinancialApp we use Zod:

```typescript
import { z } from 'zod';

const transactionSchema = z.object({
  description: z.string().min(1, 'Description is required'),
  amount: z.number().positive('Amount must be positive'),
  date: z.string().regex(/^\d{4}-\d{2}-\d{2}$/, 'Invalid date format'),
  type: z.enum(['debit', 'credit']),
  accountId: z.string().uuid('Invalid account ID'),
});
```

Wire the Zod schema into the form fields with the `schema` option:

```typescript
import { formField, withSchema } from '@angular/forms/signals';

export function createTransactionSchema() {
  return withSchema(transactionSchema, {
    description: formField(''),
    amount: formField(0),
    date: formField(new Date().toISOString().slice(0, 10)),
    type: formField<'debit' | 'credit'>('debit'),
    accountId: formField(''),
  });
}
```

`withSchema()` walks the Zod schema and the field tree in parallel, attaching the appropriate validators to each field. When the user changes a value, validation runs synchronously against the Zod parser. Errors are available as a signal on each field: `form.amount.errors()` returns `null` when valid or an array of error objects when invalid.

---

## Submitting Forms

### Logic for Submitting Forms

Signal Forms provide a `submit()` helper that validates the entire tree and, if valid, invokes a callback with the typed form value:

```typescript
import { submit } from '@angular/forms/signals';

@Component({ /* ... */ })
export class TransactionEntryComponent {
  readonly form = createTransactionSchema();
  private transactionService = inject(TransactionService);

  onSubmit() {
    submit(this.form, {
      onValid: (value) => {
        this.transactionService.create(value).subscribe();
      },
      onInvalid: (errors) => {
        console.error('Validation failed:', errors);
      },
    });
  }
}
```

The `value` passed to `onValid` is fully typed -- its type is inferred from the field tree, so `value.amount` is `number`, `value.type` is `'debit' | 'credit'`, and so on. No casting is needed.

### Template for Submitting

The template wires the submit handler to the native form event:

```html
<form signalForm (ngSubmit)="onSubmit()">
  <label for="description">Description</label>
  <input id="description" [formField]="form.description" />

  <label for="amount">Amount</label>
  <input id="amount" type="number" [formField]="form.amount" />

  <label for="type">Type</label>
  <select id="type" [formField]="form.type">
    <option value="debit">Debit</option>
    <option value="credit">Credit</option>
  </select>

  <label for="accountId">Account</label>
  <input id="accountId" [formField]="form.accountId" />

  <label for="date">Date</label>
  <input id="date" type="date" [formField]="form.date" />

  <button type="submit">Record Transaction</button>
</form>
```

There is nothing new in the template itself -- `signalForm` and `[formField]` handle the binding, and Angular's existing `(ngSubmit)` event fires the handler.

### Further Submit Actions

After a successful submission you often need to reset the form or navigate away. Signal Forms expose a `reset()` function that restores every field to its initial value:

```typescript
import { submit, reset } from '@angular/forms/signals';

onSubmit() {
  submit(this.form, {
    onValid: (value) => {
      this.transactionService.create(value).subscribe({
        next: () => {
          reset(this.form);
          this.router.navigate(['/transactions']);
        },
      });
    },
  });
}
```

`reset()` walks the field tree and calls `.set()` on each field with the initial value captured at creation time. Validation state, touched state, and dirty state are all cleared.

---

## Custom Validators

### Implementing a First Custom Validators

A custom validator in Signal Forms is a plain function that receives the current value and returns either `null` (valid) or an error object:

```typescript
import { ValidatorFn } from '@angular/forms/signals';

const positiveAmount: ValidatorFn<number> = (value) => {
  return value > 0 ? null : { positiveAmount: 'Amount must be greater than zero' };
};
```

Attach it to a field through the `validators` option:

```typescript
const amount = formField<number>(0, {
  validators: [positiveAmount],
});
```

The `validators` array runs in order. If any validator returns an error object, the field is marked invalid and subsequent validators still run so that all errors are collected in a single pass.

### Refactoring Validators into Functions

Hardcoded thresholds rarely survive contact with production. Wrap validators in factory functions that accept configuration:

```typescript
function minAmount(min: number): ValidatorFn<number> {
  return (value) =>
    value >= min ? null : { minAmount: `Amount must be at least ${min}` };
}

function maxAmount(max: number): ValidatorFn<number> {
  return (value) =>
    value <= max ? null : { maxAmount: `Amount must not exceed ${max}` };
}
```

Usage is straightforward:

```typescript
const amount = formField<number>(0, {
  validators: [minAmount(0.01), maxAmount(1_000_000)],
});
```

This pattern is identical in spirit to the legacy `Validators.min()` and `Validators.max()`, but the types are tighter -- `ValidatorFn<number>` cannot accidentally be applied to a `string` field.

### Showing Validation Errors

Validation errors are exposed as a signal on each field. The template reads them reactively:

```html
<label for="amount">Amount</label>
<input id="amount" type="number" [formField]="form.amount" />
@if (form.amount.errors(); as errors) {
  <div class="error" role="alert">
    @for (error of errors | keyvalue; track error.key) {
      <p>{{ error.value }}</p>
    }
  </div>
}
```

Because `errors()` is a signal, the error block appears and disappears automatically as the user edits the field. No manual subscription management, no `statusChanges` observable.

### Conditional Validation

Some validators should only run when another field has a specific value. The transaction form, for example, requires a `memo` field only for credits above a threshold:

```typescript
import { formField, conditionalValidator } from '@angular/forms/signals';

export function createTransactionSchema() {
  const type = formField<'debit' | 'credit'>('debit');
  const amount = formField<number>(0, {
    validators: [minAmount(0.01)],
  });
  const memo = formField<string>('', {
    validators: [
      conditionalValidator(
        () => type() === 'credit' && amount() > 10_000,
        required('Memo is required for credits over $10,000')
      ),
    ],
  });

  return { type, amount, memo };
}
```

`conditionalValidator()` takes a predicate signal and a validator. When the predicate returns `false`, the validator is skipped and the field is treated as valid. The predicate is itself reactive -- because it reads `type()` and `amount()`, it re-evaluates whenever either changes, and the memo field's validity updates accordingly.

### Multi-Field Validators

Validators that compare two or more fields belong at the group level, not on an individual field. A date range validator ensures that `startDate` is before `endDate`:

```typescript
import { groupValidator, GroupValidatorFn } from '@angular/forms/signals';

const dateRangeValidator: GroupValidatorFn<{
  startDate: string;
  endDate: string;
}> = (group) => {
  const start = new Date(group.startDate());
  const end = new Date(group.endDate());
  return start < end ? null : { dateRange: 'Start date must precede end date' };
};
```

Apply it with `groupValidator()` when building the group:

```typescript
const dateRange = groupValidator(
  {
    startDate: formField(''),
    endDate: formField(''),
  },
  { validators: [dateRangeValidator] }
);
```

Group-level errors are separate from field-level errors. The template accesses them through the group's own `errors()` signal.

### Accessing Sibling Fields

Inside a field-level validator you sometimes need to read a sibling. Since fields are signals, you can close over them:

```typescript
function matchesField(
  sibling: FormField<string>,
  message: string
): ValidatorFn<string> {
  return (value) => (value === sibling() ? null : { mismatch: message });
}

const email = formField<string>('');
const confirmEmail = formField<string>('', {
  validators: [matchesField(email, 'Emails must match')],
});
```

Reading `sibling()` inside the validator creates a reactive dependency. When `email` changes, `confirmEmail`'s validation automatically reruns. This replaces the legacy pattern of subscribing to one control's `valueChanges` from another.

### Tree Validators

For deeply nested forms, a **tree validator** receives the entire field tree and can validate across any combination of fields:

```typescript
import { treeValidator, TreeValidatorFn } from '@angular/forms/signals';

const portfolioBalanceValidator: TreeValidatorFn = (tree) => {
  const holdings = tree.holdings as FormField<number>[];
  const total = holdings.reduce((sum, h) => sum + h(), 0);
  return total === 100
    ? null
    : { portfolioBalance: `Holdings must total 100%, currently ${total}%` };
};
```

Tree validators run after all field and group validators, giving them a consistent view of the entire form's state.

### Asynchronous Validators

Validators that need to perform I/O -- checking a database, calling an API -- return a `Promise` instead of a synchronous result:

```typescript
import { AsyncValidatorFn } from '@angular/forms/signals';

function uniqueAccountNumber(
  accountService: AccountService
): AsyncValidatorFn<string> {
  return async (value) => {
    if (!value) return null;
    const exists = await firstValueFrom(accountService.exists(value));
    return exists ? { uniqueAccount: 'Account number already in use' } : null;
  };
}
```

Attach async validators through the `asyncValidators` option:

```typescript
const accountNumber = formField<string>('', {
  asyncValidators: [uniqueAccountNumber(inject(AccountService))],
  debounce: 500,
});
```

Async validators run after all synchronous validators pass. While an async validator is pending, the field exposes a `pending()` signal that returns `true`, allowing the template to show a loading indicator:

```html
<input [formField]="form.accountNumber" />
@if (form.accountNumber.pending()) {
  <span class="spinner" aria-label="Validating..."></span>
}
```

### HTTP Validators

HTTP validators are a specific case of async validators where the I/O is an HTTP call. Wrap them in a factory that injects the HTTP client:

```typescript
function accountExists(): AsyncValidatorFn<string> {
  const http = inject(HttpClient);
  return async (value) => {
    if (value.length < 6) return null;
    const result = await firstValueFrom(
      http.get<{ exists: boolean }>(`/api/accounts/check/${value}`)
    );
    return result.exists
      ? { accountExists: 'This account number is already taken' }
      : null;
  };
}
```

Because the factory calls `inject()`, it must be invoked inside an injection context -- either in a constructor, a field initializer, or inside `runInInjectionContext()`. The `debounce` option on the field prevents flooding the server with requests on every keystroke.

---

## Large and Nested Forms

### Form Groups

The client onboarding form for the FinancialApp collects personal information, address details, and employment data. Each section is a **form group** -- a plain object of fields wrapped with `formGroup()` for validation and metadata:

```typescript
// client-onboarding.component.ts
import { Component, inject } from '@angular/core';
import { formField, formGroup, SignalFormDirective } from '@angular/forms/signals';
import { FormsModule } from '@angular/forms';
import { ClientService } from '@financial-app/shared/data-access';

@Component({
  selector: 'app-client-onboarding',
  imports: [FormsModule, SignalFormDirective],
  templateUrl: './client-onboarding.component.html',
})
export class ClientOnboardingComponent {
  private clientService = inject(ClientService);

  readonly form = {
    personal: formGroup({
      firstName: formField(''),
      lastName: formField(''),
      email: formField(''),
      dateOfBirth: formField(''),
    }),
    address: formGroup({
      street: formField(''),
      city: formField(''),
      state: formField(''),
      zip: formField(''),
      country: formField('US'),
    }),
    employment: formGroup({
      employer: formField(''),
      title: formField(''),
      annualIncome: formField<number>(0),
    }),
  };
}
```

Each call to `formGroup()` returns a typed group object. The template binds to nested fields with dot access:

```html
<fieldset>
  <legend>Personal Information</legend>
  <input [formField]="form.personal.fields.firstName" placeholder="First name" />
  <input [formField]="form.personal.fields.lastName" placeholder="Last name" />
  <input [formField]="form.personal.fields.email" type="email" placeholder="Email" />
  <input [formField]="form.personal.fields.dateOfBirth" type="date" />
</fieldset>

<fieldset>
  <legend>Address</legend>
  <input [formField]="form.address.fields.street" placeholder="Street" />
  <input [formField]="form.address.fields.city" placeholder="City" />
  <input [formField]="form.address.fields.state" placeholder="State" />
  <input [formField]="form.address.fields.zip" placeholder="ZIP" />
</fieldset>
```

Groups provide their own `valid()`, `dirty()`, and `errors()` signals, so you can conditionally enable a "Next" button for each section of a multi-step wizard without validating the entire form.

### Form Arrays

The onboarding form also collects the client's investment holdings -- a list of variable length. `formArray()` manages a dynamic list of field trees:

```typescript
import { formField, formGroup, formArray } from '@angular/forms/signals';

readonly holdings = formArray([
  this.createHolding(),
]);

private createHolding() {
  return formGroup({
    ticker: formField(''),
    shares: formField<number>(0),
    allocation: formField<number>(0),
  });
}

addHolding() {
  this.holdings.push(this.createHolding());
}

removeHolding(index: number) {
  this.holdings.removeAt(index);
}
```

The template iterates over the array with `@for`:

```html
<fieldset>
  <legend>Investment Holdings</legend>
  @for (holding of holdings.fields(); track $index) {
    <div class="holding-row">
      <input [formField]="holding.fields.ticker" placeholder="Ticker" />
      <input [formField]="holding.fields.shares" type="number" placeholder="Shares" />
      <input [formField]="holding.fields.allocation" type="number" placeholder="%" />
      <button type="button" (click)="removeHolding($index)">Remove</button>
    </div>
  }
  <button type="button" (click)="addHolding()">Add Holding</button>
</fieldset>
```

`formArray` exposes `fields()` as a signal of the current items, so adding or removing a holding triggers a reactive update in the template.

### Validating Form Arrays

Array-level validators receive the current list and can enforce constraints across all items:

```typescript
import { ArrayValidatorFn } from '@angular/forms/signals';

const allocationTotals: ArrayValidatorFn = (items) => {
  const total = items.reduce(
    (sum, item) => sum + item.fields.allocation(),
    0
  );
  return total === 100
    ? null
    : { allocationTotal: `Allocations must total 100% (currently ${total}%)` };
};

readonly holdings = formArray([this.createHolding()], {
  validators: [allocationTotals],
});
```

The template can display the array-level error below the holdings list:

```html
@if (holdings.errors(); as errors) {
  <p class="error" role="alert">{{ errors['allocationTotal'] }}</p>
}
```

### Subforms

When a form section grows complex enough to warrant its own component, extract it as a **subform**. The parent passes the field group as an input, and the child binds to it:

```typescript
// address-subform.component.ts
@Component({
  selector: 'app-address-subform',
  imports: [FormsModule, SignalFormDirective],
  template: `
    <input [formField]="group.fields.street" placeholder="Street" />
    <input [formField]="group.fields.city" placeholder="City" />
    <input [formField]="group.fields.state" placeholder="State" />
    <input [formField]="group.fields.zip" placeholder="ZIP" />
  `,
})
export class AddressSubformComponent {
  readonly group = input.required<FormGroup<AddressFields>>();
}
```

The parent passes the group:

```html
<app-address-subform [group]="form.address" />
```

Because the group is a reference to the same signal tree, edits inside the subform propagate to the parent automatically. No event emitters, no manual synchronization.

---

## Working with Form Metadata

### Reading Metadata

Every field and group in a signal form exposes metadata as signals. The most commonly used are:

- `valid()` -- `true` when all validators pass
- `invalid()` -- the inverse of `valid()`
- `dirty()` -- `true` when the value differs from the initial value
- `pristine()` -- the inverse of `dirty()`
- `touched()` -- `true` when the field has received and lost focus
- `untouched()` -- the inverse of `touched()`
- `pending()` -- `true` when an async validator is in flight
- `errors()` -- the current error object, or `null`

These are all reactive. A `computed()` that reads `form.amount.valid()` will automatically update when validity changes.

### Displaying Metadata

Use metadata signals to show contextual UI cues:

```html
<input [formField]="form.amount" [class.invalid]="form.amount.invalid()" />
@if (form.amount.touched() && form.amount.invalid()) {
  <div class="error" role="alert">
    Please enter a valid amount.
  </div>
}
```

The guard `form.amount.touched()` prevents error messages from appearing before the user has interacted with the field -- the same UX pattern as the legacy forms API, but expressed with signals instead of CSS classes and observables.

### Defining Custom Metadata

Signal Forms allow you to attach arbitrary metadata to a field through the `metadata` option:

```typescript
const amount = formField<number>(0, {
  metadata: {
    label: 'Transaction Amount',
    helpText: 'Enter the amount in USD',
    maxLength: 12,
  },
});
```

Custom metadata is inert -- Signal Forms do not interpret it. It exists to give the template (or a shared form-rendering component) access to field-specific configuration without scattering magic strings across the template.

### Reading Custom Metadata

Access custom metadata through the `metadata()` signal:

```html
<label>{{ form.amount.metadata().label }}</label>
<input [formField]="form.amount" />
<small>{{ form.amount.metadata().helpText }}</small>
```

A generic form field component can use this pattern to render any field without knowing its specific purpose:

```typescript
@Component({
  selector: 'app-form-field',
  template: `
    <label>{{ field().metadata().label }}</label>
    <input [formField]="field()" />
    @if (field().metadata().helpText) {
      <small>{{ field().metadata().helpText }}</small>
    }
    @if (field().touched() && field().invalid()) {
      <div class="error" role="alert">{{ field().errors() | json }}</div>
    }
  `,
})
export class FormFieldComponent {
  readonly field = input.required<FormField<unknown>>();
}
```

This is the foundation of a design-system-level form library: metadata-driven rendering with validation built in.

---

## Null and Undefined Values

Legacy Reactive Forms introduced `nonNullable` as an opt-in because `FormControl.reset()` returned `null` by default -- a source of countless runtime errors. Signal Forms invert this: fields are **non-nullable by default**. Calling `reset()` restores the initial value, not `null`.

If a field genuinely needs to represent the absence of a value, type it explicitly:

```typescript
const middleName = formField<string | null>(null);
```

Now `middleName()` returns `string | null`, and the template must handle the `null` case. This is a deliberate friction -- it forces you to decide at design time whether `null` is a valid state, rather than discovering it at runtime when a user clears an input.

For `undefined`, the same rule applies: `formField<string | undefined>(undefined)` is valid, but you must opt into it. In practice, prefer `null` over `undefined` for form values because JSON serialization drops `undefined` properties, which can silently change the shape of a submitted payload.

---

## Custom Fields

Not every input is a text box. Date pickers, rich text editors, and autocomplete widgets all need to participate in the signal form tree. Signal Forms support custom fields through the `ControlValueAccessor` interface, the same mechanism used by legacy forms:

```typescript
@Component({
  selector: 'app-currency-input',
  providers: [
    {
      provide: NG_VALUE_ACCESSOR,
      useExisting: CurrencyInputComponent,
      multi: true,
    },
  ],
  template: `
    <input
      type="text"
      [value]="display()"
      (input)="onInput($event)"
      (blur)="onTouched()"
    />
  `,
})
export class CurrencyInputComponent implements ControlValueAccessor {
  private rawValue = signal<number>(0);
  readonly display = computed(() =>
    new Intl.NumberFormat('en-US', { style: 'currency', currency: 'USD' })
      .format(this.rawValue())
  );

  private onChange: (value: number) => void = () => {};
  onTouched: () => void = () => {};

  writeValue(value: number) {
    this.rawValue.set(value ?? 0);
  }

  registerOnChange(fn: (value: number) => void) {
    this.onChange = fn;
  }

  registerOnTouched(fn: () => void) {
    this.onTouched = fn;
  }

  onInput(event: Event) {
    const raw = (event.target as HTMLInputElement).value.replace(/[^0-9.]/g, '');
    const numeric = parseFloat(raw) || 0;
    this.rawValue.set(numeric);
    this.onChange(numeric);
  }
}
```

Because `ControlValueAccessor` is bridge-compatible, the currency input works with both legacy Reactive Forms and Signal Forms. In the signal form template, bind it the same way:

```html
<app-currency-input [formField]="form.amount" />
```

The `[formField]` directive detects the `ControlValueAccessor` and wires the bridge automatically. Validation, metadata, and all other signal-based features work identically.

---

## Reactive Forms Compatibility

Classic Reactive Forms (`FormControl`, `FormGroup`, `FormBuilder`) still power a large amount of existing Angular code. Rather than scatter that material through this chapter, it lives in a single reference: [Appendix C: Reactive Forms Compatibility](ch56-appendix-c-reactive-forms.md). The appendix covers common patterns, a side-by-side Signal Forms mapping, running both systems in one app, and an incremental migration strategy.

---

## Summary

Signal Forms replace Angular's decade-old forms modules with a signal-native API that is smaller, fully typed, and synchronous by default. A `formField()` creates a writable signal with built-in validation. Groups and arrays compose fields into trees. Zod and Standard Schema provide declarative validation without custom code. Custom validators are plain functions. Metadata signals replace CSS-class-based state inspection. And the entire system is reactive end-to-end: change a value, and every dependent computation -- validation, error display, submit-button state -- updates in the same microtask.

The two forms built in this chapter -- transaction entry and client onboarding -- demonstrate the full range of the API: flat schemas, nested groups, dynamic arrays, conditional validators, async HTTP validators, and subform composition. These patterns will reappear throughout the FinancialApp as we add portfolio management in [Chapter 12](ch10-ngrx-signal-store.md) and authentication flows in [Chapter 27](ch17-auth-patterns.md).

Signal Forms are still in developer preview. The API *will* change. But the mental model -- forms as trees of signals, validation as pure functions, metadata as reactive state -- is stable. Invest in understanding the model, and the inevitable API churn becomes a find-and-replace exercise rather than a rewrite.
