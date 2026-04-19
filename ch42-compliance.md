# Chapter 16: Compliance for Financial Apps

Compliance in a financial application is not a problem you hand off to the legal team with a promise to revisit it next quarter. It is frontend code. The consent banner that appears before the first analytics script fires, the masking that hides account numbers in a server log, the audit event emitted when a user views a client's portfolio -- these are Angular components, services, and interceptors that the engineering team builds, tests, and ships. When a regulator asks how FinancialApp obtained consent from a German user before setting a marketing cookie, the answer lives in the git history of the `ConsentBanner` component.

The cost of getting it wrong is measured in regulator fines, not customer complaints. GDPR penalties scale to four percent of global annual revenue. A single PCI DSS finding can revoke a merchant account. A SOC 2 auditor who discovers that audit logs are written only on the client can fail an entire control. These are existential outcomes, and avoiding them requires treating compliance like any other non-functional requirement: design it into the architecture, codify it in libraries, enforce it in CI. This chapter is not legal advice -- the specifics depend on your jurisdiction, your data, and your counsel -- but a frontend engineering pattern library: the code and architectural decisions that make compliance enforceable, testable, and auditable from inside an Angular application.

> **Companion code:** `financial-app/libs/shared/compliance/` with `ConsentBanner`, `ConsentService`, `AuditLogger`, `PiiMasker`; integrated into the app shell and transaction flows.

---

## The Regulatory Landscape at a Glance

Financial applications that serve users across borders inherit a patchwork of overlapping regulations. A handful of regimes set the shape of most compliance work:

| Regime | Jurisdiction | Frontend impact |
|---|---|---|
| **GDPR / UK GDPR** | EU, United Kingdom | Consent before non-essential cookies and tracking; right to access, export, and delete personal data |
| **CCPA / CPRA** | California | "Do Not Sell or Share My Personal Information" link; opt-out flows |
| **LGPD** | Brazil | Consent and data subject rights similar to GDPR, with Portuguese-language disclosures |
| **PCI DSS** | Global, card networks | Card data never touches your application; scope reduction via hosted fields |
| **SOC 2 Type II** | Contractual, US-centric | Operational controls over a sustained window; audit trails and access review |
| **PSD2 / UK Open Banking** | EU, UK | Strong Customer Authentication (SCA) flows; consent for account information access |

Most of these regulations are written in terms of outcomes -- "the user has consented," "sensitive data is protected," "access is logged" -- not in terms of specific technologies. That is both freedom and burden. You have latitude in how to implement controls, but the controls must actually work. An auditor does not care whether your consent banner uses signals or NgRx; they care that consent was captured, stored, respected, and retrievable.

For FinancialApp the baseline is that any user anywhere might visit, and the strictest applicable rule wins. If a Californian opts out of selling their data and a Brazilian revokes consent, the runtime must honor both without regional forks of the codebase.

---

## Consent: GDPR and CCPA in Practice

Consent is the most visible piece of compliance. Before a single analytics pixel fires, a first-time visitor from a GDPR jurisdiction must see a banner, choose what to allow, and have that choice respected on every subsequent page load. GDPR's requirements are the strictest in common use, so designing to GDPR and relaxing for other regions is the pragmatic default.

### Consent Banner Types

Three consent models cover the majority of real deployments:

- **Notice-only.** A banner informs the user that cookies are in use. Legal in the United States for most purposes, insufficient under GDPR.
- **Opt-in.** No non-essential processing happens until the user affirms. Required under GDPR for marketing and analytics cookies. The absence of a choice is treated as refusal.
- **Granular categories.** The user accepts, rejects, or customizes by category: strictly necessary, functional, analytics, marketing. The most auditable model and the only one that satisfies GDPR's "freely given, specific, informed, and unambiguous" standard.

FinancialApp uses the granular model. The extra UI complexity is outweighed by the audit advantage -- every consent state maps to a specific choice the user made, and the record of that choice survives in storage.

### The ConsentService

The `ConsentService` is the source of truth for what the user has agreed to. Every downstream consumer -- the telemetry service from [Chapter 17](ch20-observability.md), the marketing tag manager, the optional personalization features -- reads its state from here.

```typescript
// libs/shared/compliance/src/lib/consent.service.ts
import { computed, effect, inject, Injectable, signal } from '@angular/core';
import { HttpClient } from '@angular/common/http';

export type ConsentCategory =
  | 'necessary'
  | 'functional'
  | 'analytics'
  | 'marketing';

export interface ConsentState {
  necessary: true;
  functional: boolean;
  analytics: boolean;
  marketing: boolean;
  version: number;
  recordedAt: string;
}

const STORAGE_KEY = 'fa.consent.v1';
const CONSENT_VERSION = 2;

@Injectable({ providedIn: 'root' })
export class ConsentService {
  private readonly http = inject(HttpClient);
  private readonly state = signal<ConsentState | null>(this.load());

  readonly current = this.state.asReadonly();
  readonly hasDecided = computed(() => this.state() !== null);
  readonly analyticsAllowed = computed(() => this.state()?.analytics === true);
  readonly marketingAllowed = computed(() => this.state()?.marketing === true);

  constructor() {
    effect(() => {
      const current = this.state();
      if (current) {
        localStorage.setItem(STORAGE_KEY, JSON.stringify(current));
        this.http.post('/api/compliance/consent', current).subscribe({ error: () => {} });
      }
    });
  }

  accept(choice: Omit<ConsentState, 'necessary' | 'version' | 'recordedAt'>) {
    this.state.set({
      necessary: true,
      ...choice,
      version: CONSENT_VERSION,
      recordedAt: new Date().toISOString(),
    });
  }

  acceptAll() { this.accept({ functional: true, analytics: true, marketing: true }); }
  rejectAll() { this.accept({ functional: false, analytics: false, marketing: false }); }

  revoke() {
    localStorage.removeItem(STORAGE_KEY);
    this.state.set(null);
  }

  private load(): ConsentState | null {
    try {
      const raw = localStorage.getItem(STORAGE_KEY);
      const parsed = raw ? (JSON.parse(raw) as ConsentState) : null;
      return parsed?.version === CONSENT_VERSION ? parsed : null;
    } catch {
      return null;
    }
  }
}
```

A few details matter. The `version` field lets you invalidate prior consent when the categories change -- if you add a new "personalization" category, bumping `CONSENT_VERSION` forces every user to re-consent rather than inferring agreement from an outdated choice. The server sync is fire-and-forget, but the state still persists in `localStorage` so a transient network error does not cost the user's choice. The authoritative record is the server copy, which is what an auditor will ask for.

### The ConsentBanner Component

The banner is a standalone component rendered near the root of the application shell. It reads `hasDecided` from the service and renders nothing once the user has made a choice.

```typescript
// libs/shared/compliance/src/lib/consent-banner.component.ts
@Component({
  selector: 'app-consent-banner',
  standalone: true,
  imports: [FinButton, FinCard, FinCheckbox],
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    @if (!consent.hasDecided()) {
      <fin-card role="dialog" aria-modal="true"
        aria-labelledby="consent-title" aria-describedby="consent-body">
        <h2 id="consent-title" i18n="@@consentTitle">Your privacy choices</h2>
        <p id="consent-body" i18n="@@consentBody">
          FinancialApp uses cookies to operate this site, analyze usage,
          and personalize marketing. Strictly necessary cookies are always
          active. You can change your choices at any time from Settings.
        </p>

        @if (showDetails()) {
          <fieldset>
            <fin-checkbox checked disabled i18n>Strictly necessary</fin-checkbox>
            <fin-checkbox [(checked)]="functional" i18n>Functional</fin-checkbox>
            <fin-checkbox [(checked)]="analytics" i18n>Analytics</fin-checkbox>
            <fin-checkbox [(checked)]="marketing" i18n>Marketing</fin-checkbox>
          </fieldset>
          <fin-button variant="primary" (click)="saveChoices()" i18n>Save choices</fin-button>
          <fin-button variant="text" (click)="consent.rejectAll()" i18n>Reject all</fin-button>
        } @else {
          <fin-button variant="primary" (click)="consent.acceptAll()" i18n>Accept all</fin-button>
          <fin-button variant="secondary" (click)="consent.rejectAll()" i18n>Reject all</fin-button>
          <fin-button variant="text" (click)="showDetails.set(true)" i18n>Customize</fin-button>
        }
      </fin-card>
    }
  `,
})
export class ConsentBannerComponent {
  protected readonly consent = inject(ConsentService);
  protected readonly showDetails = signal(false);
  protected readonly functional = signal(true);
  protected readonly analytics = signal(false);
  protected readonly marketing = signal(false);

  saveChoices() {
    this.consent.accept({
      functional: this.functional(),
      analytics: this.analytics(),
      marketing: this.marketing(),
    });
  }
}
```

The component composes the design-system primitives (`FinButton`, `FinCard`, `FinCheckbox`) described in [Chapter 32](ch22-material-design-system.md), which means it inherits the system's focus management, contrast guarantees, and RTL behavior without bespoke styles. The `role="dialog"` and `aria-modal="true"` combination is what makes the banner acceptable to both GDPR and the ADA/EAA -- a consent banner that cannot be operated by a screen reader fails accessibility and the regulations that piggyback on it.

### Respecting Consent

Capturing a choice is the easy half. The hard half is ensuring that every downstream script, tag, and feature checks that choice before acting. The pattern used throughout FinancialApp is to gate initialization on a signal read:

```typescript
// libs/shared/compliance/src/lib/telemetry-gate.ts
import { effect, inject, Injectable } from '@angular/core';
import { TelemetryService } from '@financial-app/shared/observability';
import { ConsentService } from './consent.service';

@Injectable({ providedIn: 'root' })
export class TelemetryGate {
  private readonly consent = inject(ConsentService);
  private readonly telemetry = inject(TelemetryService);

  constructor() {
    effect(() => {
      if (this.consent.analyticsAllowed()) {
        this.telemetry.enable();
      } else {
        this.telemetry.disable();
      }
    });
  }
}
```

`TelemetryService` from [Chapter 17](ch20-observability.md) exposes `enable()` and `disable()` methods that start and stop the underlying SDK, flush buffered events on disable, and no-op on subsequent calls. The `effect` reacts the moment consent changes, so a user who revokes analytics from the settings page stops being tracked immediately -- not on the next page load.

Marketing tags follow the same pattern. The tag manager snippet is not included in `index.html`; instead, a service injects the script tag dynamically when `marketingAllowed()` becomes true and removes it when consent is revoked. If the script is never loaded, it cannot set cookies or make outbound calls, which is what "respect consent" means in practice.

### Right to Be Forgotten and Data Export

GDPR Article 17 grants users the right to have their personal data erased. Article 15 grants them the right to a copy. Both must be available through reasonable means, and "reasonable" in 2026 means a button in the account settings. The deletion flow is a confirmation dialog followed by a POST to the backend:

```typescript
// libs/shared/compliance/src/lib/account-deletion.component.ts
async deleteAccount() {
  const confirmed = await this.dialog.confirm({
    title: $localize`:@@deleteTitle:Delete your account?`,
    body: $localize`:@@deleteBody:This permanently removes your profile,
      transactions, and saved preferences. This cannot be undone.`,
    requireTypedConfirmation: 'DELETE',
  });
  if (!confirmed) return;
  this.audit.log({ type: 'account_deletion_requested' });
  await firstValueFrom(this.http.post('/api/compliance/delete', {}));
  this.auth.signOut();
  this.router.navigateByUrl('/goodbye');
}
```

The `requireTypedConfirmation` pattern forces the user to type a specific string, which blocks accidental confirmations. The audit event fires before the request, so even if the backend processes the deletion and fails to respond, the intent is recorded. Data export works the same way with one twist: export is asynchronous. Producing a complete dump of a user's transactions, portfolios, and profile can take minutes, so the frontend kicks off a job and trusts the backend to email a signed download link when ready.

---

## PCI DSS: Never Touch the Card

PCI DSS governs the handling of payment card data. The single most important frontend rule is this: your Angular code must never see a card number. Not in a signal, not in a form field, not in a `HttpClient` request body, not in memory for a microsecond. The moment a card number enters your JavaScript heap, your application is in PCI scope, and the scope determines how much auditing, pen testing, and quarterly certification you owe.

The way to stay out of scope is tokenization through a payment provider's hosted fields. Stripe Elements, Braintree Hosted Fields, and Adyen Web Components all implement the same pattern: they embed an iframe that they control, the user types their card number into the iframe, and the iframe posts the card directly to the provider. What returns to your Angular app is a token -- a short opaque string that represents the card but cannot be used to reconstruct it.

```typescript
// libs/shared/compliance/src/lib/stripe-card-element.component.ts
@Component({
  selector: 'app-stripe-card-element',
  standalone: true,
  template: `
    <label for="card-element" i18n>Card details</label>
    <div #cardElement id="card-element" class="stripe-mount"></div>
    <fin-button (click)="submit()" i18n>Pay</fin-button>
  `,
})
export class StripeCardElementComponent implements AfterViewInit {
  private readonly loader = inject(StripeLoader);
  private readonly mount = viewChild.required<ElementRef<HTMLElement>>('cardElement');
  readonly tokenized = output<string>();

  private card!: stripe.elements.Element;

  async ngAfterViewInit() {
    const stripe = await this.loader.get();
    this.card = stripe.elements().create('card');
    this.card.mount(this.mount().nativeElement);
  }

  async submit() {
    const stripe = await this.loader.get();
    const result = await stripe.createToken(this.card);
    if (result.token) this.tokenized.emit(result.token.id);
  }
}
```

The parent component submits the transaction with only the token -- never the card number:

```typescript
this.http.post('/api/transactions', {
  amount: this.amount(),
  currency: this.currency(),
  source: this.cardToken(),
}).subscribe(/* ... */);
```

The card number crossed from the user's browser to Stripe's servers and never entered FinancialApp's runtime. This is the difference between PCI SAQ-A (the shortest self-assessment questionnaire, roughly two dozen controls) and SAQ-D (the full questionnaire with several hundred controls). Choosing SAQ-A is a frontend architectural decision made once, early, and defended forever.

The corollary is that your team never accepts a feature request that reintroduces card handling -- no "custom card input with our own branding," no "accept the card in chat and forward it." The moment you type the first character of a card number into something you control, you owe the long questionnaire.

---

## Audit Trails

SOC 2 and most banking regulations require a record of compliance-relevant user actions: who logged in, what they viewed, what they changed, when. The frontend contributes to this record by emitting structured events, but with a critical caveat -- **client-side audit logs are untrusted**. A determined attacker can pause the JavaScript debugger, delete events, or call the logger with forged data. The authoritative audit log lives on the server, constructed from server-side observations of API calls.

So what is the frontend logger for? It fills in the context the server cannot see: which page the user was on, which button they clicked, which filter they applied. It produces a stream of events that correlates with the server's stream via a shared correlation ID, giving auditors a richer picture than either side alone.

```typescript
// libs/shared/compliance/src/lib/audit-logger.service.ts
export type AuditEventType =
  | 'transaction_submitted' | 'transaction_viewed' | 'account_viewed'
  | 'consent_changed' | 'pii_unmasked'
  | 'account_deletion_requested' | 'data_export_requested';

export interface AuditEvent {
  type: AuditEventType;
  subjectId?: string;
  detail?: Record<string, string | number | boolean>;
}

@Injectable({ providedIn: 'root' })
export class AuditLogger {
  private readonly http = inject(HttpClient);
  private readonly correlation = inject(CorrelationIdService);

  log(event: AuditEvent): void {
    this.http.post('/api/compliance/audit', {
      ...event,
      correlationId: this.correlation.current(),
      occurredAt: new Date().toISOString(),
      path: location.pathname,
    }).subscribe({ error: () => {} });
  }
}
```

The `CorrelationIdService` is the same one the observability stack uses to stitch together frontend telemetry and backend traces ([Chapter 17](ch20-observability.md)). Because every outbound HTTP request carries the correlation ID as a header, an auditor investigating a transaction can pull the server-side log, find the matching audit events, and reconstruct the sequence of clicks that led to the submission -- without trusting the client's version of events as ground truth.

Feature components inject the logger and emit events at the moments that matter:

```typescript
const result = await this.transactions.submit(form);
this.audit.log({
  type: 'transaction_submitted',
  subjectId: String(result.id),
  detail: { amount: form.amount, currency: form.currency },
});
```

Note what is not in the detail: account numbers, counterparty names, routing numbers. The audit event records that a transaction happened and the minimum needed to find it again; the sensitive fields stay out of the client log because the server already recorded them authoritatively.

---

## PII Handling and Masking

Personally Identifiable Information -- account numbers, Social Security numbers, tax IDs, dates of birth -- is the data that every compliance regime cares most about. The guiding principle is that PII should appear in the UI only when the user specifically needs it, and it should never appear in a place it might be unintentionally retained: logs, screenshots, crash reports, analytics payloads, exported screenshots of support tickets.

The `PiiMasker` utility provides the masked representation that is used almost everywhere in the UI.

```typescript
// libs/shared/compliance/src/lib/pii-masker.ts
export const PiiMasker = {
  accountNumber(value: string | null | undefined): string {
    return value ? `••••${value.slice(-4)}` : '';
  },

  ssn(value: string | null | undefined): string {
    const digits = (value ?? '').replace(/\D/g, '');
    return digits.length === 9 ? `•••-••-${digits.slice(-4)}` : '•••-••-••••';
  },

  email(value: string | null | undefined): string {
    if (!value?.includes('@')) return '';
    const [local, domain] = value.split('@');
    return `${local[0]}${'•'.repeat(Math.max(local.length - 1, 1))}@${domain}`;
  },
};
```

A companion pipe makes the masker template-friendly:

```typescript
// libs/shared/compliance/src/lib/mask.pipe.ts
@Pipe({ name: 'mask', standalone: true, pure: true })
export class MaskPipe implements PipeTransform {
  transform(value: string | null | undefined, kind: keyof typeof PiiMasker): string {
    return PiiMasker[kind](value);
  }
}
```

In a template, a holding row renders the routing account as `{{ account.number | mask:'accountNumber' }}` and the support screen displays `{{ user.ssn | mask:'ssn' }}`. The unmasked value never reaches the DOM unless the user explicitly asks for it.

When a user does need the full value -- a support agent verifying an account, a customer copying it to their clipboard -- the reveal is gated and audited:

```typescript
async revealAccountNumber(accountId: string) {
  const confirmed = await this.dialog.confirm({
    title: $localize`:@@revealTitle:Reveal account number?`,
    body: $localize`:@@revealBody:This action is recorded for compliance review.`,
  });
  if (!confirmed) return;
  this.audit.log({ type: 'pii_unmasked', subjectId: accountId });
  this.revealed.set(await this.accounts.fetchFullNumber(accountId));
  setTimeout(() => this.revealed.set(null), 10_000);
}
```

The reveal emits an audit event before the request, fetches from a dedicated endpoint (not the general account endpoint, so the server can apply extra authorization), displays briefly, and clears automatically -- a user who walks away from their desk does not leave a full SSN on screen indefinitely. The same masker also feeds the telemetry scrubber in [Chapter 17](ch20-observability.md): before any event leaves the browser, the scrubber walks the payload and replaces PII fields with their masked form.

---

## Data Retention and Storage

The frontend retains customer data longer than most developers realize. A signal holds the last fetched portfolio while the user clicks through the tabs. `localStorage` persists between sessions. Each of these is a retention decision, and together they determine whether a shared computer in a coffee shop leaks the previous user's holdings to the next one.

FinancialApp's rules:

1. **Never persist sensitive data in `localStorage`.** Account numbers, transaction lists, and anything resembling PII live in memory only. `localStorage` is reserved for user preferences (theme, language, consent state) that would be merely annoying, not harmful, to leak.
2. **Prefer `sessionStorage` over `localStorage` for anything identifying.** `sessionStorage` is scoped to the tab and cleared when it closes.
3. **Clear in-memory caches on sign-out and apply a TTL to sensitive data.** The auth module's `signOut()` calls `clear()` on every feature store; the portfolio holdings cache expires after ten minutes of inactivity so a subsequent user sees a clean slate and stale balances never linger.

The enforcement is a single lint rule (shown in the CI section below) that flags any `localStorage.setItem` whose key matches a PII pattern. Every suppression requires a reviewer to agree that the data is not actually sensitive.

---

## Cookie Management

Cookies classify into four buckets that map directly onto the `ConsentState` shape:

| Bucket | Example | Consent required |
|---|---|---|
| Strictly necessary | Session ID, anti-XSRF token | No -- exempt under GDPR |
| Functional | Language preference, theme | Yes, though often bundled with "necessary" |
| Analytics | Usage tracking, A/B assignment | Yes -- opt-in |
| Marketing | Retargeting, conversion pixels | Yes -- opt-in |

Before any non-necessary cookie is set, the code that sets it consults `consent.current()`. A third-party SDK that cannot be gated at the API level must be loaded dynamically only after consent, as shown earlier with the marketing tag manager -- if the script is never loaded, it cannot set cookies. FinancialApp's production domain sets no third-party cookies in the necessary bucket; every essential cookie is first-party, `Secure`, `HttpOnly`, and `SameSite=Strict`, aligned with the auth patterns from [Chapter 27](ch17-auth-patterns.md). A cookie that was once "temporary" and now lives forever is a compliance liability, not a feature.

---

## Jurisdictional Content Gating

Compliance rules vary by where the user is, and the UI must adapt. FinancialApp's locale awareness ([Chapter 23](ch25-internationalization.md)) doubles as a compliance vehicle: each locale carries not only translated copy but the disclosures, risk warnings, and opt-out links required by its jurisdiction.

```typescript
// libs/shared/compliance/src/lib/disclosure.service.ts
@Injectable({ providedIn: 'root' })
export class DisclosureService {
  private readonly locale = inject(LOCALE_ID);

  readonly investmentWarning = computed(() => {
    switch (this.locale) {
      case 'de': case 'de-DE': return $localize`:@@disclDE:Investitionen bergen ...`;
      case 'en-GB': return $localize`:@@disclUK:Capital at risk. Investments ...`;
      case 'pt-BR': return $localize`:@@disclBR:Investimentos envolvem riscos ...`;
      default: return $localize`:@@disclDefault:Investments involve risk ...`;
    }
  });
}
```

Some jurisdictions are not served at all. Sanctions lists (OFAC in the US, EU sanctions, UK HMT) prohibit doing business with users in certain countries. Enforcement is primarily server-side (IP filtering and sanctions screening at registration) but the frontend participates in the UX: a blocked user sees a plain page explaining that the service is unavailable in their region, served early, before any PII is collected, so a visitor from a sanctioned region never creates a record that would need to be expunged.

---

## Accessibility as a Compliance Control

Accessibility and privacy compliance share a thread that is easy to miss: a consent banner that is not operable by a screen reader has not obtained informed consent from a blind user. The ADA in the United States and the European Accessibility Act (EAA) treat inaccessible consent UI as a compliance failure in both regimes simultaneously. The consent banner shown earlier sets `role="dialog"`, `aria-modal="true"`, and the `aria-labelledby`/`aria-describedby` pair for precisely this reason; the design-system primitives from [Chapter 32](ch22-material-design-system.md) carry keyboard support and focus trapping by default. The same principle applies to the account deletion confirmation, data export, and PII reveal dialog -- every compliance-adjacent piece of UI must follow WCAG 2.2 AA, or the compliance argument falls apart with it.

---

## CI/CD Enforcement

Compliance discipline decays under release pressure unless the pipeline enforces it. FinancialApp's CI runs three compliance checks on every pull request.

**No PII in `localStorage`.** A lint rule fires on suspicious key names:

```jsonc
// .eslintrc.compliance.json
{
  "rules": {
    "no-restricted-syntax": ["error", {
      "selector": "CallExpression[callee.object.name='localStorage'][callee.property.name='setItem'] Literal[value=/account|ssn|tax|routing|pan|card|balance/i]",
      "message": "Do not persist PII or balances in localStorage."
    }]
  }
}
```

**Consent banner copy check.** Legal signs off on the exact translation units. A post-extraction script compares the XLF output against a checked-in `legal/consent-approved.json`, and any diff blocks the build until both engineering and legal review. New banner copy cannot ship without a paired update to the approval file.

**Dependency license audit.** A `license-checker` run produces a report of open-source licenses in the production bundle, flags any license outside the allowlist (permissive licenses only, no GPL in the browser bundle), and archives the report as a build artifact. When an auditor asks for a software bill of materials, the artifact is the answer.

None of these checks catch every compliance failure -- an engineer determined to leak data will find a key name the lint rule does not match. The value of the checks is that the easy mistakes become impossible and the hard mistakes become visible in code review.

---

## Summary

Compliance is a frontend engineering problem, and a financial application that treats it as such has a material competitive advantage: faster audits, lower fines, and a product that users actually trust.

- **Consent is code.** The `ConsentService` is the single source of truth; the telemetry gate from [Chapter 17](ch20-observability.md) and every marketing script read its state before acting. The `ConsentBanner` is accessible by construction.
- **PCI stays out of scope by design.** Hosted fields and tokenization keep card numbers out of your Angular heap forever; the architecture choice between SAQ-A and SAQ-D is made once and defended continuously.
- **Audit events complement, never replace, server-side logs.** The frontend logger provides context via the shared correlation ID; the server-side log remains the authoritative record.
- **PII is masked by default** via `PiiMasker`, the `mask` pipe, and the telemetry scrubber. Unmasking is a deliberate, audited event.
- **Retention is enforced in storage.** `localStorage` is for preferences only; in-memory caches clear on sign-out and expire under TTL.
- **Jurisdictional disclosures ride on top of i18n ([Chapter 23](ch25-internationalization.md)).** Each locale carries the regulatory language required by its regime.
- **Accessibility and compliance intersect** at every consent dialog, deletion flow, and PII reveal. A consent banner that fails WCAG has not actually obtained consent.
- **CI enforces the easy rules** -- PII lint, consent-copy approval, license audits -- so engineering review can focus on the hard ones.

Sensitive data, once leaked, cannot be unleaked. Fines, once levied, cannot be unlevied. The patterns in this chapter will not make your application bulletproof -- no architecture can -- but they make it defensible. When the auditor arrives with a spreadsheet of controls, the answers live in source control, signed off by code review, and reproducible in CI. The close ties to [Chapter 35](ch18-security-owasp.md) and [Chapter 40](ch19-error-handling.md) are not coincidences -- security, reliability, and compliance are facets of the same engineering discipline applied to the same code.
