# Chapter 45: Multi-Step Forms & Wizards

> **Extends:** [Chapter 6](ch06-signal-forms.md) introduced Signal Forms. This chapter builds a wizard pattern on top of that foundation.

Some forms cannot fit on one screen. FinancialApp's client onboarding needs six distinct sections before an advisor can open a brokerage account: **identity** (legal name, date of birth, tax ID), **address**, **KYC documents** (passport, proof of address), **risk profile** (experience, tolerance, horizon), **beneficiaries** (up to four with allocation percentages), and a **review** step where client and advisor sign off before submission.

Rendering all of that on a single page works technically -- Signal Forms can compose arbitrarily deep trees -- but the UX is brutal. A 200-field scroll discourages completion, hides validation errors off-screen, and turns mobile into a scrolling marathon. Worse, onboarding is often paused and resumed across sessions: a client starts on their phone at lunch and finishes on a desktop that evening. A single-page form has nowhere to put a progress cue or a "resume where you left off" prompt.

A **wizard** turns one overwhelming page into a manageable journey. The user sees one section at a time, validates each before moving on, and can resume from where they left off. This chapter builds a generic wizard that wraps Signal Forms, then refactors the existing `ClientOnboardingComponent` from [Chapter 6](ch06-signal-forms.md) to use it.

---

## When to Wizard vs. Single Page

The overhead -- navigation state, progress tracking, autosave -- is only worth paying when one of these conditions holds:

- **Complexity is high.** Rule of thumb: more than ~20 fields or more than 4 logical sections.
- **Fields depend on earlier answers.** If step 2's fields are determined by step 1 (e.g., different KYC documents based on citizenship), a staged reveal is clearer than conditionally-hidden blocks.
- **Abandonment risk is measurable.** If analytics show more than 15-20% of users who start the form never submit, splitting into steps with autosave usually recovers a meaningful fraction.
- **Mobile is a first-class target.** One question per screen adapts naturally to narrow viewports.
- **The workflow is inherently sequential.** Review-and-sign-off patterns map cleanly to "you cannot proceed until this is correct."

Stick with a single page when users frequently revise earlier fields based on later ones, when every field is independent, or when the total count fits above the fold. Power users prefer one-screen forms -- a wizard helps novices but adds clicks for experts.

---

## Wizard Anatomy

The wizard library has three pieces:

- **`FinWizard`** -- the orchestrator. Manages the active step index, tracks which steps are complete, gates navigation on per-step validity, and emits `complete` with the aggregated value at the end. Renders the progress indicator and Next/Back buttons; step content comes from projected children.
- **`FinWizardStep`** -- a thin wrapper around a Signal Forms subtree, typically one `formGroup` per step. Owns its own fields, validators, and template. Exposes its form group through a signal-valued input so the parent can ask "are you valid?" without reaching into private state.
- **Progress indicator** -- a linear bar for 2-4 steps, a step list with check marks for 5+. Built on design-system tokens ([Chapter 27](ch27-material-design-system.md)) so it inherits the current theme automatically.

---

## State Management

### One `formGroup` Per Step

Rather than one monolithic form tree, each step owns its own `formGroup`. The wizard stitches them together at submit time:

```typescript
// libs/shared/ui/wizard/onboarding-forms.ts
import { formGroup, formField, formArray } from '@angular/forms/signals';

export function createOnboardingForms() {
  return {
    identity: formGroup({
      firstName: formField(''), lastName: formField(''),
      dateOfBirth: formField(''), taxId: formField(''),
    }),
    address: formGroup({
      street: formField(''), city: formField(''), state: formField(''),
      zip: formField(''), country: formField('US'),
    }),
    documents: formGroup({
      passportUrl: formField<string | null>(null),
      proofOfAddressUrl: formField<string | null>(null),
    }),
    risk: formGroup({
      experience: formField<'none' | 'some' | 'extensive'>('none'),
      tolerance: formField<number>(5),
      horizon: formField<'short' | 'medium' | 'long'>('medium'),
    }),
    beneficiaries: formArray([createBeneficiary()]),
  };
}

function createBeneficiary() {
  return formGroup({
    name: formField(''), relationship: formField(''), allocation: formField<number>(0),
  });
}
```

Each group validates independently. The wizard only reads `.valid()` on the active group to decide whether navigation is allowed; it never needs to know a step's internals.

### Signal-Based Wizard State

The orchestrator tracks three signals: the active step index, the set of completed step indices, and the projected step components.

```typescript
// libs/shared/ui/wizard/fin-wizard.component.ts
import { Component, computed, contentChildren, inject, signal } from '@angular/core';
import { LiveAnnouncer } from '@angular/cdk/a11y';
import { FinWizardStepComponent } from './fin-wizard-step.component';

@Component({
  selector: 'fin-wizard',
  imports: [FinProgressIndicatorComponent, FinButtonComponent],
  templateUrl: './fin-wizard.component.html',
})
export class FinWizardComponent {
  private announcer = inject(LiveAnnouncer);

  readonly steps = contentChildren(FinWizardStepComponent);
  readonly activeStepIndex = signal(0);
  readonly completedSteps = signal<Set<number>>(new Set());

  protected currentStep = computed(() => this.steps()[this.activeStepIndex()]);
  protected totalSteps = computed(() => this.steps().length);
  protected isFirstStep = computed(() => this.activeStepIndex() === 0);
  protected isLastStep = computed(() => this.activeStepIndex() === this.totalSteps() - 1);
  protected canAdvance = computed(() => this.currentStep()?.isValid() ?? false);

  next(): void {
    if (!this.canAdvance()) { this.currentStep()?.markAllAsTouched(); return; }
    const i = this.activeStepIndex();
    this.completedSteps.update((s) => new Set(s).add(i));
    if (i < this.totalSteps() - 1) {
      this.activeStepIndex.set(i + 1);
      this.announceStepChange();
    }
  }

  back(): void {
    if (this.isFirstStep()) return;
    this.activeStepIndex.update((i) => i - 1);
    this.announceStepChange();
  }

  jumpTo(index: number): void {
    if (!this.completedSteps().has(index) && index !== this.activeStepIndex()) return;
    this.activeStepIndex.set(index);
    this.announceStepChange();
  }

  private announceStepChange(): void {
    const step = this.currentStep();
    if (!step) return;
    this.announcer.announce(
      `Step ${this.activeStepIndex() + 1} of ${this.totalSteps()}: ${step.label()}`,
      'polite'
    );
    queueMicrotask(() => step.focusFirstField());
  }
}
```

`contentChildren()` from [Chapter 10](ch10-signal-queries.md) collects the projected `FinWizardStep` instances as a signal, so adding or removing steps at runtime just works. Because `canAdvance` is a `computed()`, the Next button's disabled state updates automatically when the active step's validity changes -- no manual wiring.

---

## Navigation

### Next and Back

The template renders two buttons and consults the signals above:

```html
<!-- libs/shared/ui/wizard/fin-wizard.component.html -->
<fin-progress-indicator
  [steps]="steps()" [activeIndex]="activeStepIndex()"
  [completed]="completedSteps()" (stepSelected)="jumpTo($event)" />

<div class="wizard-body"><ng-content /></div>

<div class="wizard-actions">
  <fin-button variant="secondary" [disabled]="isFirstStep()" (click)="back()">Back</fin-button>
  @if (!isLastStep()) {
    <fin-button variant="primary" [disabled]="!canAdvance()" (click)="next()">Next</fin-button>
  } @else {
    <fin-button variant="primary" [disabled]="!canAdvance()" (click)="submit()">Submit</fin-button>
  }
</div>
```

The Next button is disabled until the current step validates. Pressing Enter inside the form calls `next()`, which calls `markAllAsTouched()` on an invalid step so every field's error surfaces at once -- the user instantly sees what still needs attention.

### Optional Steps and Direct Jumps

Some sections are genuinely optional. `FinWizardStep` accepts an `optional` flag; an optional step counts as completable with an empty value:

```typescript
@Component({ selector: 'fin-wizard-step', templateUrl: './fin-wizard-step.component.html' })
export class FinWizardStepComponent {
  readonly label = input.required<string>();
  readonly optional = input(false);
  readonly form = input.required<FieldTree>();
  isValid = computed(() => this.optional() || this.form().valid());
}
```

Clicking a completed step in the progress indicator jumps the user directly back. Forward jumps are disallowed by `jumpTo()` -- you can revisit any completed step, but cannot skip to one you have not reached legitimately.

### Unsaved-Changes Guard

Wizards accumulate real work. Losing a half-completed onboarding to an accidental browser-back click is unacceptable. The wizard integrates with [Chapter 12](ch12-initialization-routes.md)'s `CanDeactivateFn`:

```typescript
// apps/financial-app/src/app/features/clients/client-onboarding.component.ts
export class ClientOnboardingComponent implements HasUnsavedChanges {
  readonly wizardRef = viewChild.required(FinWizardComponent);

  hasUnsavedChanges(): boolean {
    const w = this.wizardRef();
    return w.completedSteps().size > 0 && !w.isSubmitted();
  }
}
```

A production variant replaces `window.confirm` with the `FinConfirmDialog` from [Chapter 27](ch27-material-design-system.md).

---

## Cross-Step Validation

Some rules cannot be checked inside a single step. Beneficiary allocations must total 100 -- a constraint within one array -- but the review step additionally requires *all* previous groups to be valid.

Cross-step validators are plain `computed()` signals on the wizard host that read the relevant form groups:

```typescript
protected allocationError = computed<string | null>(() => {
  const total = this.forms.beneficiaries.fields()
    .reduce((sum, b) => sum + b.fields.allocation(), 0);
  return total === 100 ? null : `Allocations must total 100% (currently ${total}%)`;
});

protected readyToSubmit = computed(() =>
  this.forms.identity.valid() && this.forms.address.valid() &&
  this.forms.documents.valid() && this.forms.risk.valid() &&
  this.allocationError() === null
);
```

These computeds drive the review step. If `allocationError()` is non-null, the review page shows an error banner pointing back to step 5; clicking the banner jumps the wizard there. This is **deferred validation** -- the rule only surfaces at submit time, not during data entry where it would frustrate users still typing.

For validators that must run on submit of the *entire* wizard (server cross-entity checks, document expiration), expose a `submitValidators` input on `FinWizard` and run them before emitting `complete`.

---

## Draft Autosave and Restore

Long wizards are abandoned without autosave. FinancialApp writes the in-progress form to IndexedDB on every field change, debounced to avoid write amplification, and prompts the user to resume on reload.

### Debounced Save Service

Building on the offline sync utilities from [Chapter 44](ch44-optimistic-ui.md) and the error-handling patterns in [Chapter 23](ch23-error-handling.md):

```typescript
// libs/shared/ui/wizard/wizard-draft.service.ts
import { Injectable, ErrorHandler, inject } from '@angular/core';
import { openDB, IDBPDatabase } from 'idb';

export interface WizardDraft<T> {
  id: string; value: T; activeStep: number;
  completedSteps: number[]; savedAt: number;
}

@Injectable({ providedIn: 'root' })
export class WizardDraftService {
  private errorHandler = inject(ErrorHandler);
  private db: Promise<IDBPDatabase> = openDB('fin-wizard-drafts', 1, {
    upgrade: (db) => db.createObjectStore('drafts', { keyPath: 'id' }),
  });
  private pending = new Map<string, ReturnType<typeof setTimeout>>();

  save<T>(draft: WizardDraft<T>, debounceMs = 500): void {
    clearTimeout(this.pending.get(draft.id));
    this.pending.set(draft.id, setTimeout(async () => {
      try {
        await (await this.db).put('drafts', { ...draft, savedAt: Date.now() });
      } catch (err) {
        this.errorHandler.handleError(err);
      } finally {
        this.pending.delete(draft.id);
      }
    }, debounceMs));
  }

  async restore<T>(id: string): Promise<WizardDraft<T> | null> {
    try { return (await (await this.db).get('drafts', id)) ?? null; }
    catch (err) { this.errorHandler.handleError(err); return null; }
  }

  async clear(id: string): Promise<void> {
    try { await (await this.db).delete('drafts', id); }
    catch (err) { this.errorHandler.handleError(err); }
  }
}
```

The wizard hooks an `effect()` to the aggregate form value and calls `save()` on every change:

```typescript
protected currentValue = computed(() => ({
  identity: this.forms.identity.value(),
  address: this.forms.address.value(),
  documents: this.forms.documents.value(),
  risk: this.forms.risk.value(),
  beneficiaries: this.forms.beneficiaries.value(),
}));

constructor() {
  const drafts = inject(WizardDraftService);
  effect(() => drafts.save({
    id: this.draftId(), value: this.currentValue(),
    activeStep: this.activeStepIndex(),
    completedSteps: Array.from(this.completedSteps()),
    savedAt: Date.now(),
  }));
}
```

Because `currentValue()` reads every field's `value()` signal, the effect re-runs on any change. Debouncing collapses rapid typing into a single IndexedDB put.

### Resume Prompt and Cleanup

On component init, the wizard checks for a saved draft:

```typescript
async ngOnInit(): Promise<void> {
  const draft = await this.drafts.restore(this.draftId());
  if (!draft) return;
  const minutesAgo = Math.round((Date.now() - draft.savedAt) / 60_000);
  const resume = await this.dialog.confirm({
    title: 'Resume onboarding?',
    message: `You have an unsaved onboarding from ${minutesAgo} minutes ago.`,
    confirmLabel: 'Resume', cancelLabel: 'Start fresh',
  });
  if (resume) this.applyDraft(draft);
  else await this.drafts.clear(this.draftId());
}
```

`applyDraft()` patches each form group with `reset()` and the saved values, restores the active step index, and rehydrates the completed-step set. Once the server accepts the final submission, clear the draft -- stale drafts on shared devices mean the next user gets prompted to "resume" someone else's identity data, a privacy incident waiting to happen.

---

## Accessibility

Wizards are more accessibility-sensitive than single-page forms because a step change is a navigation event without a URL change. Assistive tech has to be told what happened.

### Progress Bar Semantics

Following the pattern from [Chapter 22](ch22-accessibility-aria.md), the indicator exposes `role="progressbar"` with live ARIA values:

```html
<div class="fin-wizard-progress" role="progressbar"
     [attr.aria-valuenow]="activeIndex() + 1"
     [attr.aria-valuemin]="1"
     [attr.aria-valuemax]="totalSteps()"
     [attr.aria-valuetext]="ariaValueText()">
  <ol class="steps">
    @for (step of steps(); track $index) {
      <li [class.active]="$index === activeIndex()"
          [class.completed]="completed().has($index)">
        <button type="button" (click)="stepSelected.emit($index)"
                [attr.aria-current]="$index === activeIndex() ? 'step' : null"
                [disabled]="!completed().has($index) && $index !== activeIndex()">
          <span class="step-number">{{ $index + 1 }}</span>
          <span class="step-label">{{ step.label() }}</span>
        </button>
      </li>
    }
  </ol>
</div>
```

`aria-valuetext` gives screen readers a human-readable alternative to the percentage: *"Step 3 of 6: KYC documents"* rather than *"50 percent"*.

### Announcing Step Changes and Focus Management

The orchestrator uses `LiveAnnouncer` from `@angular/cdk/a11y` ([Chapter 22](ch22-accessibility-aria.md)) to speak the new step label on every navigation. Without this, a keyboard user hears silence when the content swaps and assumes nothing happened.

After advancing, focus moves into the new step. The step exposes `focusFirstField()`, called inside `queueMicrotask()` so the new step has rendered first:

```typescript
// libs/shared/ui/wizard/fin-wizard-step.component.ts
focusFirstField(): void {
  this.host.nativeElement
    .querySelector<HTMLElement>('input, select, textarea, [tabindex="0"]')
    ?.focus();
}
```

### Keyboard Shortcuts

Enter within the form triggers `next()`. Shift+Enter as "go back" is a convenience binding for keyboard power users -- strictly optional, must not hijack Shift+Enter in multiline inputs. Tab must not trap focus outside the step body, and Escape must not silently drop the user's work.

---

## Design-System Integration

Wizard steps render their fields through `FinFormField` and submit through `FinButton` ([Chapter 27](ch27-material-design-system.md)). The progress indicator uses design tokens directly:

```scss
// libs/shared/ui/wizard/fin-wizard.component.scss
.fin-wizard-progress {
  display: flex; padding: var(--fin-space-md);
  background: var(--fin-color-surface-variant); border-radius: var(--fin-radius-md);

  .steps { display: flex; gap: var(--fin-space-md); list-style: none; margin: 0; padding: 0; }
  li.active .step-number { background: var(--fin-color-primary); color: var(--fin-color-primary-contrast); }
  li.completed .step-number::before { content: '\2713'; }
}
```

Because colors, spacing, and radii are tokens, dark mode, brand overrides, and high-contrast variants all work without additional code.

---

## Error Recovery

Every step renders validation errors next to each field. When the user clicks Next on an invalid step, the wizard prepends a per-step **error summary**:

```html
@if (currentStep()?.errorSummary()?.length) {
  <div class="step-error-summary" role="alert">
    <h3>Please fix the following:</h3>
    <ul>
      @for (error of currentStep()!.errorSummary(); track error.field) {
        <li><a [href]="'#' + error.fieldId">{{ error.label }}: {{ error.message }}</a></li>
      }
    </ul>
  </div>
}
```

The anchor links give keyboard users a single-Tab route to the first broken field. On the review step, the wizard aggregates errors from every previous step into one summary; clicking an entry jumps back to the offending step automatically.

---

## Server Submission Strategies

**All-at-once** is the simpler pattern: every field collected, final step POSTs the whole payload. One endpoint, one transaction, easy rollback. For onboarding flows that complete in a single session this is the right default -- the server never sees a half-formed client record.

```typescript
submit(): void {
  this.clientService.onboard(this.currentValue()).subscribe({
    next: async () => {
      await this.drafts.clear(this.draftId());
      this.router.navigate(['/clients', 'onboarded']);
    },
    error: (err) => this.errorHandler.handleError(err),
  });
}
```

**Per-step commit** suits long multi-session wizards. Each step commits to the server on completion; resuming loads the partially-built entity from the server rather than IndexedDB. Think 10-step enterprise onboarding with document uploads -- you want uploads persisted immediately, not lost to a tab crash.

```typescript
async onStepComplete(stepIndex: number, payload: unknown): Promise<void> {
  await firstValueFrom(this.http.patch(this.stepEndpoints[stepIndex], payload));
}
```

Per-step commit requires the backend to accept partial payloads (PATCH a draft resource), enforce validation incrementally, and finalize only on a terminal "submit" endpoint. Considerably more work on both sides, but for regulated workflows where incomplete records must survive across days, weeks, or advisors, it is the only pattern that survives production.

Hybrids are common: IndexedDB autosave for in-session recovery plus one server commit per logical phase. Pick the granularity that matches your rollback semantics.

---

## Testing Wizards

Wizard tests need a harness that mirrors user navigation:

```typescript
// libs/shared/ui/wizard/testing/fin-wizard.harness.ts
export class FinWizardHarness {
  constructor(private fixture: ComponentFixture<{ wizard: FinWizardComponent }>) {}

  async next(): Promise<void> { await this.click(this.primaryBtn()); }
  async back(): Promise<void> { await this.click(this.secondaryBtn()); }

  activeIndex(): number { return this.fixture.componentInstance.wizard.activeStepIndex(); }
  canAdvance(): boolean { return !this.primaryBtn()?.disabled; }

  private async click(btn?: HTMLButtonElement | null) {
    btn?.click();
    this.fixture.detectChanges();
    await this.fixture.whenStable();
  }
  private primaryBtn() {
    return this.fixture.nativeElement.querySelector<HTMLButtonElement>(
      'fin-button[variant="primary"] button');
  }
  private secondaryBtn() {
    return this.fixture.nativeElement.querySelector<HTMLButtonElement>(
      'fin-button[variant="secondary"] button');
  }
}
```

Representative tests assert that navigation gates correctly and that drafts restore:

```typescript
// apps/financial-app/src/app/features/clients/client-onboarding.component.spec.ts
describe('ClientOnboardingComponent wizard flow', () => {
  it('blocks advancing when the identity step is invalid', async () => {
    const fixture = TestBed.createComponent(ClientOnboardingComponent);
    fixture.detectChanges();
    const wizard = new FinWizardHarness(fixture);
    expect(wizard.canAdvance()).toBe(false);
    await wizard.next();
    expect(wizard.activeIndex()).toBe(0);
  });

  it('advances after identity is valid and restores a saved draft', async () => {
    const drafts = TestBed.inject(WizardDraftService);
    await drafts.save({
      id: 'client-onboarding',
      value: { identity: { firstName: 'Restored' } },
      activeStep: 2, completedSteps: [0, 1], savedAt: Date.now(),
    });
    const fixture = TestBed.createComponent(ClientOnboardingComponent);
    await fixture.whenStable();
    expect(fixture.componentInstance.wizardRef().activeStepIndex()).toBe(2);
  });
});
```

Pair these with an `axe-core` assertion from [Chapter 22](ch22-accessibility-aria.md) on each step's rendered markup, and a Playwright end-to-end test ([Chapter 25](ch25-e2e-playwright.md)) that walks the full six-step flow in a real browser. Unit tests catch the logic; e2e tests catch the glue.

---

## Summary

Wizards exist because attention is finite. Splitting a long form into steps reduces cognitive load, makes validation failures local and actionable, and turns an abandonable wall of inputs into a progressable journey. The cost is architectural complexity: navigation state, progress UI, autosave, focus management, and submission strategy all become explicit concerns.

The `FinWizard` pattern composes cleanly with the Signal Forms foundation from [Chapter 6](ch06-signal-forms.md): each step is a `formGroup`, step validity is a `.valid()` signal, and the wizard's `canAdvance` is a `computed()` over the current step. Cross-step rules use the same reactive primitives; draft autosave hooks into `effect()` and IndexedDB. Design-system wrappers from [Chapter 27](ch27-material-design-system.md) keep visuals consistent, `LiveAnnouncer` from [Chapter 22](ch22-accessibility-aria.md) keeps assistive tech informed, and the `CanDeactivate` guard from [Chapter 12](ch12-initialization-routes.md) keeps half-finished work safe.

Reserve wizards for forms that genuinely benefit from staging -- complexity, dependency, abandonment risk, or multi-session workflows. For shorter forms, the overhead is not worth the clicks. When you do reach for one, the patterns here cover the hard parts so each feature team only writes the step components that describe their domain.

> **Companion code:** `financial-app/libs/shared/ui/wizard/` with `FinWizard` and `FinWizardStep`; the existing client-onboarding feature is refactored to use them.
