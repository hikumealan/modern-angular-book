# Appendix C: Reactive Forms Compatibility

Signal Forms is the v21 default and what the rest of this book uses, but the `FormControl` / `FormGroup` / `FormBuilder` trio from `@angular/forms` still powers an enormous amount of existing code. This appendix is the compact reference for readers whose day job includes reading, maintaining, or incrementally modernizing that code. Every pattern is grounded in the same FinancialApp domain (`Account`, `Transaction`, `Portfolio`, `Holding`, `Client`) used throughout the book, so what you learn here transfers directly.

---

## Why This Appendix Exists

[Chapter 6](ch06-signal-forms.md) teaches Signal Forms as the Angular v21 default, and every form built in the companion FinancialApp uses `formField`, `formGroup`, and `formArray`. The reality outside the book is messier: classic Reactive Forms is still the dominant forms API in production, and will remain so for several years. A new hire onto an existing Angular team needs enough Reactive Forms fluency to read the existing code, fix bugs in it, and -- when the timing is right -- migrate it. This appendix covers exactly that: just enough Reactive Forms to be dangerous, a side-by-side translation guide to Signal Forms, and a pragmatic playbook for incremental migration when you decide the effort is worth it.

---

## A Minimal Reactive Form Example

A Reactive Form lives in the component class as an object graph of `FormControl`, `FormGroup`, and `FormArray` instances. Angular's `ReactiveFormsModule` supplies the `[formGroup]` and `formControlName` directives that bind this graph to the template. The model is the source of truth; the template is a view over it.

The component below edits a FinancialApp `Account` with three fields -- `name`, `type`, and `balance`:

```typescript
// account-edit.component.ts
import { Component, inject } from '@angular/core';
import {
  FormBuilder,
  ReactiveFormsModule,
  Validators,
} from '@angular/forms';

@Component({
  selector: 'app-account-edit',
  imports: [ReactiveFormsModule],
  templateUrl: './account-edit.component.html',
})
export class AccountEditComponent {
  private fb = inject(FormBuilder);

  form = this.fb.nonNullable.group({
    name: ['', [Validators.required, Validators.minLength(2)]],
    type: [
      'checking' as 'checking' | 'savings' | 'investment',
      Validators.required,
    ],
    balance: [0, [Validators.required, Validators.min(0)]],
  });

  save() {
    if (this.form.invalid) {
      this.form.markAllAsTouched();
      return;
    }
    const value = this.form.getRawValue();
    console.log(value);
  }
}
```

```html
<form [formGroup]="form" (ngSubmit)="save()">
  <label>
    Name
    <input type="text" formControlName="name" />
    @if (form.controls.name.touched && form.controls.name.invalid) {
      <span class="error">Name must be at least 2 characters.</span>
    }
  </label>

  <label>
    Type
    <select formControlName="type">
      <option value="checking">Checking</option>
      <option value="savings">Savings</option>
      <option value="investment">Investment</option>
    </select>
  </label>

  <label>
    Balance
    <input type="number" formControlName="balance" />
  </label>

  <button type="submit" [disabled]="form.invalid">Save</button>
</form>
```

`FormBuilder.nonNullable.group()` produces a strongly typed `FormGroup` whose `.value` excludes `undefined` after `reset()` -- the default nullable behaviour is a long-standing source of bugs, and `nonNullable` is the recommended starting point for new Reactive Forms code. Validators run on every value change and populate `form.controls.name.errors`, which the template reads via control lookups (for example, `form.controls.balance.errors?.['min']`).

---

## Common Reactive Forms Patterns

Four patterns account for the overwhelming majority of production Reactive Forms code. Most legacy forms you encounter are a composition of these.

### Cross-Field Validation with `AbstractControlOptions`

Validators attached at the group level receive the whole group as their argument and can compare siblings. Pass them through the `AbstractControlOptions` second argument to the group constructor:

```typescript
import { AbstractControl, ValidationErrors, ValidatorFn } from '@angular/forms';

const dateRange: ValidatorFn = (
  group: AbstractControl,
): ValidationErrors | null => {
  const start = group.get('startDate')?.value as string | null;
  const end = group.get('endDate')?.value as string | null;
  if (!start || !end) return null;
  return start <= end ? null : { dateRange: true };
};

this.form = this.fb.nonNullable.group(
  {
    startDate: ['', Validators.required],
    endDate: ['', Validators.required],
  },
  { validators: [dateRange] },
);
```

Group-level errors appear on the group itself, not on any single control: read them with `form.errors?.['dateRange']`. This is the same mental model as Signal Forms' `groupValidator()`, just with a nullable untyped surface.

### Dynamic `FormArray` for N-of-Something

A `FormArray` holds a variable-length list of controls or groups, which maps naturally to "add another line item" UIs. A portfolio editor that lets a client manage an arbitrary number of holdings:

```typescript
export class PortfolioEditorComponent {
  private fb = inject(FormBuilder);

  form = this.fb.nonNullable.group({
    name: ['', Validators.required],
    holdings: this.fb.array([this.createHoldingGroup()]),
  });

  get holdings() {
    return this.form.controls.holdings;
  }

  private createHoldingGroup() {
    return this.fb.nonNullable.group({
      ticker: ['', Validators.required],
      shares: [0, [Validators.required, Validators.min(1)]],
    });
  }

  addHolding() {
    this.holdings.push(this.createHoldingGroup());
  }

  removeHolding(index: number) {
    this.holdings.removeAt(index);
  }
}
```

The template iterates with `formArrayName` on the container and `formGroupName` on each item:

```html
<div formArrayName="holdings">
  @for (group of holdings.controls; track $index) {
    <fieldset [formGroupName]="$index">
      <input formControlName="ticker" placeholder="Ticker" />
      <input type="number" formControlName="shares" placeholder="Shares" />
      <button type="button" (click)="removeHolding($index)">Remove</button>
    </fieldset>
  }
</div>
<button type="button" (click)="addHolding()">Add holding</button>
```

File-upload rows inside a `FormArray` follow the same shape; see [Chapter 48](ch29-file-handling.md) for the `ControlValueAccessor` pattern that lets a custom file-input component slot into any Reactive or Signal Form.

### Async Validators Returning `Observable<ValidationErrors | null>`

Async validators resolve to `ValidationErrors | null` through an `Observable` or `Promise`. They plug into the third argument of `new FormControl` or the `asyncValidators` key of `fb.group()`:

```typescript
import { AsyncValidatorFn } from '@angular/forms';
import { catchError, debounceTime, map, of, switchMap } from 'rxjs';

export function uniqueAccountName(
  accounts: AccountsService,
): AsyncValidatorFn {
  return (control) =>
    of(control.value as string).pipe(
      debounceTime(300),
      switchMap((name) => accounts.nameExists(name)),
      map((exists) => (exists ? { taken: true } : null)),
      catchError(() => of(null)),
    );
}

this.form = this.fb.nonNullable.group({
  name: [
    '',
    { validators: [Validators.required], asyncValidators: [uniqueAccountName(inject(AccountsService))] },
  ],
});
```

While an async validator is pending, `control.status === 'PENDING'` and `control.pending` is `true`. The template typically shows a small spinner next to the field while that state holds.

### Subscribing to `valueChanges` (and the RxJS Pitfalls)

`AbstractControl.valueChanges` is an `Observable<T>` that emits every change. It is the standard way to react to user input in Reactive Forms -- and the source of a family of recurring bugs:

```typescript
import { DestroyRef, inject } from '@angular/core';
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';
import { debounceTime, distinctUntilChanged, startWith } from 'rxjs/operators';

export class AccountSearchComponent {
  private destroyRef = inject(DestroyRef);

  form = inject(FormBuilder).nonNullable.group({
    query: [''],
  });

  ngOnInit() {
    this.form.controls.query.valueChanges
      .pipe(
        startWith(this.form.controls.query.value),
        debounceTime(200),
        distinctUntilChanged(),
        takeUntilDestroyed(this.destroyRef),
      )
      .subscribe((value) => this.search(value));
  }

  private search(value: string) {
    /* ... */
  }
}
```

Three pitfalls appear in nearly every bug report against a Reactive Form:

- `valueChanges` does **not** emit the initial value. Subscribers stay silent until the first real change. Pipe through `startWith(control.value)` when you need an initial emission.
- Every subscriber runs its callback on every keystroke, including side-effect-heavy work. `debounceTime` and `distinctUntilChanged` are the minimum viable defence against a runaway HTTP storm.
- Subscriptions leak without `takeUntilDestroyed` or equivalent teardown, keeping the component alive long after the user navigated away. Prefer `takeUntilDestroyed(this.destroyRef)` over manual `Subscription.unsubscribe()` bookkeeping.

Signal Forms eliminate all three: a signal has a current value by definition (no `startWith` needed), `computed()` automatically dedupes, and there is no subscription to leak.

---

## Mapping Reactive Forms to Signal Forms

The conceptual translation is cleaner than the API surfaces suggest. The table below lists the most common pairings; the code snippets that follow walk through the trickier rows.

| Reactive Forms | Signal Forms |
|---|---|
| `new FormControl(value, validators)` | `formField(value, { validators })` |
| `new FormGroup({ ... })` / `fb.group(...)` | `formGroup({ ... })` |
| `new FormArray([...])` / `fb.array(...)` | `formArray([...])` |
| `control.valueChanges` (Observable) | `field.value()` (Signal) |
| `control.statusChanges` | `field.status()` |
| `form.value` / `form.getRawValue()` | `form.value()` |
| `control.valid` / `control.errors` | `field.valid()` / `field.errors()` |
| `Validators.required` | `validators.required()` or Zod `.min(1)` |
| `control.updateValueAndValidity()` | implicit -- signals re-evaluate automatically |
| `form.markAllAsTouched()` | `form.markAllAsTouched()` |
| `[formGroup]` / `formControlName` | `[formField]="form.field"` |

### Single Field

```typescript
// Reactive
import { FormControl, Validators } from '@angular/forms';

const amount = new FormControl<number>(0, {
  validators: [Validators.required, Validators.min(0.01)],
  nonNullable: true,
});

// Signal
import { formField, ValidatorFn } from '@angular/forms/signals';

const positive: ValidatorFn<number> = (v) =>
  v >= 0.01 ? null : { min: { minimum: 0.01 } };

const amount = formField<number>(0, { validators: [positive] });
```

### Cross-Field Validation

```typescript
// Reactive
const form = this.fb.nonNullable.group(
  { a: [0], b: [0] },
  {
    validators: [
      (g) => (g.get('a')!.value <= g.get('b')!.value ? null : { range: true }),
    ],
  },
);

// Signal
import { formField, groupValidator } from '@angular/forms/signals';

const form = groupValidator(
  { a: formField(0), b: formField(0) },
  {
    validators: [
      (group) => (group.a() <= group.b() ? null : { range: true }),
    ],
  },
);
```

### Reactive Subscription to Computed Signal

```typescript
// Reactive
this.form.controls.name.valueChanges
  .pipe(takeUntilDestroyed(this.destroyRef))
  .subscribe((value) => (this.lengthHint = (value ?? '').length));

// Signal
readonly lengthHint = computed(() => this.form.name.value().length);
```

The signal version has no subscription to manage, no lifecycle-scoped teardown, and no risk of a stale snapshot -- `lengthHint()` is always derived from the latest value. For the full range of Signal Forms primitives in the same order as this mapping, see [Chapter 6](ch06-signal-forms.md).

---

## Running Both Systems in One App

Angular has no restriction against mixing forms APIs inside the same application. One component imports `ReactiveFormsModule` for its legacy form; the sibling component imports `SignalFormDirective` for its new signal-based form; both compile and run in the same bundle. The two APIs share the DOM and the Angular change-detection pipeline but not their internal state, so there is no risk of one overwriting the other at runtime.

Three observations simplify the transition period:

- **Shared validation logic.** Pure predicates -- `isValidEmail(s)`, `isPositiveAmount(n)`, `isFutureDate(d)` -- live in a plain TypeScript module and are consumed by both a Reactive `ValidatorFn` wrapper and a Signal Forms `ValidatorFn<T>`. Zod schemas are likewise framework-agnostic: parse the draft value with the Zod schema inside a Reactive validator, and hand the same schema to `withSchema()` inside a signal form. One source of truth, two adapters.
- **Custom inputs keep working.** A component that implements `ControlValueAccessor` -- a currency input, a date picker, a file dropzone from [Chapter 48](ch29-file-handling.md) -- works with both systems without code changes. `formControlName`, `[formControl]`, and `[formField]` all detect the `ControlValueAccessor` and wire it the same way.
- **At the DOM level there is no difference.** Both APIs ultimately set `.value` and dispatch `input`/`change` events on the same `<input>`, `<select>`, and `<textarea>` elements. Browser autofill, password managers, accessibility tooling, and CSS selectors behave identically in both worlds.

The practical result: a feature team can migrate at its own pace while the rest of the application keeps shipping from the same main branch.

---

## Incremental Migration Strategy

Migration is a series of small, reviewable pull requests, not a rewrite branch that never lands. A playbook that consistently works in practice:

1. **Inventory the forms.** List every component that imports `ReactiveFormsModule` or uses `FormBuilder`, then grade each by size and test coverage. The target order is smallest-first, best-tested-first.
2. **Extract validators as pure functions.** Move the body of every custom validator out of its `ValidatorFn` wrapper into a plain function that takes a value and returns an error object (or `null`). The Reactive validator becomes a one-line adapter; the eventual Signal Forms validator is a different one-line adapter around the same pure function. This step is a non-breaking refactor that makes every subsequent step easier.
3. **Start with leaf components.** A form with no children (a settings pane, a note editor, an email-update dialog) is safer to convert than a multi-step wizard. Replace the `FormGroup` with `formGroup`, swap `formControlName` bindings for `[formField]`, and delete the `ReactiveFormsModule` import when every binding in the component is gone.
4. **Test behaviour, not implementation.** The Vitest suite should assert "user enters an invalid amount, submit button is disabled, error message is visible" -- not "`FormControl.errors` contains `{ min: { ... } }`". Behavioural tests survive the migration unchanged; implementation-detail tests need rewriting for every converted form.
5. **Work outward.** When every leaf in a container is converted, flip the container next. Shared subforms migrate last, because they have the most callers and any regression ripples widest.
6. **Retire the module.** The final PR removes `ReactiveFormsModule` from the last component imports and `@angular/forms` tree-shakes the unused API away.

[Chapter 42](ch50-migrations.md) covers the general mechanics of large Angular migrations -- feature flags, long-running branches, team communication, rollback strategy -- which apply to a Reactive-to-Signal Forms migration as much as to any other modernization effort.

---

## When NOT to Migrate

Migration is not free and rarely visible to end users. If a Reactive Form is stable, well-tested, and not blocking any new feature work, there is no technical reason to rewrite it. Signal Forms do not reduce bundle size for an existing form meaningfully, and the runtime rendering cost of a well-written Reactive Form is already negligible -- [Chapter 9](ch32-performance.md) documents the profiling methodology to confirm this for your own code before assuming otherwise. Migrate when the form needs a new capability that Signal Forms expresses better (conditional validators, richer metadata, tighter end-to-end typing, simpler cross-field logic), or when the legacy code is genuinely blocking productivity. Fashion alone is a poor reason to touch a working form.

---

## Further Reading

- [Chapter 6](ch06-signal-forms.md) -- the canonical reference for Signal Forms; every pattern in the mapping table above is introduced there in depth.
- [Chapter 42](ch50-migrations.md) -- the general migration playbook (feature flags, staged rollouts, reversibility) that applies to a Reactive-to-Signal Forms conversion.
- [Chapter 48](ch29-file-handling.md) -- `ControlValueAccessor` patterns for file inputs, which work identically in both forms APIs.
- [Chapter 9](ch32-performance.md) -- profiling methodology you should run before assuming a form rewrite will pay for itself.
- [Angular Forms guide](https://angular.dev/guide/forms) -- the authoritative reference for the full Reactive Forms API surface.
