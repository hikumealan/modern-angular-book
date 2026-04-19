# Chapter 23: Internationalization at Scale

Financial applications operate across borders. When FinancialApp expands from a single English-speaking market to serve users in Germany, Japan, or Brazil, every label, date format, and currency display must adapt to the user's locale. A button that reads "Submit Transaction" in New York must read "Transaktion absenden" in Frankfurt -- and the transaction amount that appears as "$1,234.56" must render as "1.234,56 $" or "1.234,56 USD" depending on regional conventions.

Angular provides a complete internationalization (i18n) pipeline through the `@angular/localize` package. This chapter walks through that pipeline end to end: marking texts for translation, extracting them into industry-standard XLF files, integrating translations into builds, and handling the grammatical complexities that make i18n genuinely difficult -- plural forms, gender agreement, and locale-specific formatting.

> **Prerequisites:** This chapter assumes familiarity with Angular's template syntax and pipes covered in [Chapter 2](ch02-signal-components.md). We will use `CurrencyPipe`, `DatePipe`, and `DecimalPipe` alongside the i18n infrastructure.

---

## Overview

Angular's i18n story rests on three pillars:

1. **Marking** -- annotating templates and TypeScript code with translatable strings.
2. **Extracting** -- pulling those strings into translation files that linguists can work with.
3. **Building** -- compiling a separate application bundle per locale, or loading translations at runtime.

The framework uses the `i18n` attribute in templates and the `$localize` tagged template literal in TypeScript. Both produce translation units that the CLI extracts into XLIFF (XML Localization Interchange File Format) files -- the same format used by professional translation management systems.

For FinancialApp, the translation workflow looks like this: developers mark strings, the CLI extracts them into `messages.xlf`, translators produce locale-specific files like `messages.de.xlf` and `messages.ja.xlf`, and the build system compiles one bundle per locale or loads translations dynamically at startup.

---

## Installing @angular/localize

The `@angular/localize` package is not included in a default Angular project. Adding it requires a single schematic:

```bash
ng add @angular/localize
```

This command does two things: it installs the package and adds a polyfill import to your `polyfills` configuration. Under the hood, it patches the global scope with `$localize`, the tagged template function that powers runtime translation.

After installation, your `angular.json` includes the polyfill reference:

```json
{
  "projects": {
    "financial-app": {
      "architect": {
        "build": {
          "options": {
            "localize": true
          }
        }
      },
      "i18n": {
        "sourceLocale": "en-US",
        "locales": {
          "de": "src/locale/messages.de.xlf",
          "ja": "src/locale/messages.ja.xlf"
        }
      }
    }
  }
}
```

The `sourceLocale` property tells Angular which locale represents the original development language. The `locales` map points each target locale to its translation file.

---

## Marking Texts

In templates, you mark an element for translation by adding the `i18n` attribute:

```html
<button i18n="@@submitTransaction">Submit Transaction</button>
```

The `@@submitTransaction` syntax assigns a custom ID to the translation unit. Without it, Angular generates an ID from the text content and its location -- which breaks whenever you move the element. Custom IDs are more work up front but prevent translation churn across refactors.

You can also translate attributes using `i18n-*`:

```html
<input
  i18n-placeholder="@@searchPlaceholder"
  placeholder="Search transactions..."
/>
```

For FinancialApp's transaction list, a typical template might look like:

```html
<h2 i18n="@@transactionListTitle">Recent Transactions</h2>

<table>
  <thead>
    <tr>
      <th i18n="@@colDate">Date</th>
      <th i18n="@@colDescription">Description</th>
      <th i18n="@@colAmount">Amount</th>
      <th i18n="@@colStatus">Status</th>
    </tr>
  </thead>
</table>

<button i18n="@@loadMore">Load More</button>
```

Every `i18n` attribute adds a translation unit to the extracted file. The translator sees the text content and the custom ID, which provides stable context across releases.

---

## Marking Strings in the Component Class

Templates are only half the story. FinancialApp also displays messages constructed in TypeScript -- toast notifications, dynamic error messages, confirmation dialogs. The `$localize` tagged template literal handles these cases:

```typescript
import { Component, signal } from '@angular/core';

@Component({ /* ... */ })
export class TransactionConfirmComponent {
  readonly confirmationMessage = signal(
    $localize`:@@txConfirmation:Transaction submitted successfully`
  );

  readonly cancelLabel = $localize`:@@cancelButton:Cancel`;

  confirm(): void {
    this.confirmationMessage.set(
      $localize`:@@txProcessing:Processing your transaction...`
    );
  }
}
```

The syntax `:@@id:` inside the tagged template assigns the custom ID, mirroring the template convention. The colon-delimited prefix can also include a meaning and description to give translators additional context:

```typescript
const label = $localize`:meaning|description@@customId:Translatable text`;
```

For example:

```typescript
const tooltip = $localize`:transaction action|tooltip for submit@@submitTooltip:Submit this transaction for approval`;
```

The meaning ("transaction action") groups related translations. The description ("tooltip for submit") tells the translator where the string appears in the UI. Both are optional but valuable for large projects with hundreds of translation units.

---

## Extracting Texts

Once your templates and TypeScript files are marked, the CLI extracts all translation units into a single file:

```bash
ng extract-i18n --output-path src/locale --format xlf2
```

This produces `src/locale/messages.xlf` in XLIFF 2.0 format. A translation unit for the submit button looks like:

```xml
<unit id="submitTransaction">
  <segment>
    <source>Submit Transaction</source>
  </segment>
</unit>
```

To create locale-specific files, copy `messages.xlf` to `messages.de.xlf` and add `<target>` elements:

```xml
<unit id="submitTransaction">
  <segment>
    <source>Submit Transaction</source>
    <target>Transaktion absenden</target>
  </segment>
</unit>
```

The companion code for this chapter lives in `financial-app/src/locale/messages.*.xlf`. In a production workflow, you would send `messages.xlf` to a translation service or TMS (Translation Management System), and they would return completed locale files. The `--format` flag also supports `xlf` (XLIFF 1.2) and `json` for teams that prefer JSON-based workflows.

Re-run `ng extract-i18n` whenever you add or change translatable strings. The command is idempotent -- it overwrites the source file without touching your locale-specific translations.

---

## Integrating Translated Texts into Builds

With translation files in place, Angular's build system compiles a separate bundle for each locale. The `angular.json` configuration from earlier tells the builder where to find each file:

```json
{
  "i18n": {
    "sourceLocale": "en-US",
    "locales": {
      "de": "src/locale/messages.de.xlf",
      "ja": "src/locale/messages.ja.xlf"
    }
  }
}
```

Running `ng build --localize` produces one output directory per locale:

```
dist/financial-app/
  en-US/
    index.html
    main.js
    ...
  de/
    index.html
    main.js
    ...
  ja/
    index.html
    main.js
    ...
```

Each bundle contains only the strings for its locale -- no runtime lookup, no dictionary loading, no fallback chains. The translated strings are compiled directly into the JavaScript output, which means zero runtime overhead for translation resolution.

Your web server routes users to the appropriate bundle based on their `Accept-Language` header, a URL prefix (`/de/`, `/ja/`), or a user preference stored in a cookie or profile. A typical nginx configuration maps URL prefixes to the corresponding output directories.

---

## Setting the Language for Development

Building all locales for every development cycle is slow. During development, you typically work with a single locale. The `--localize` flag accepts a specific locale:

```bash
ng serve --localize --configuration=de
```

Or you can configure a development-specific locale in `angular.json`:

```json
{
  "architect": {
    "serve": {
      "options": {
        "localize": ["de"]
      }
    }
  }
}
```

For the source locale, no translation file is needed -- Angular uses the original strings as-is. This means your default `ng serve` always runs in the source locale without any i18n overhead.

When testing translations, a useful pattern is to create a "pseudo-locale" file that wraps every string in brackets: `[Submit Transaction]` becomes `[Transaktion absenden]`. This makes untranslated strings immediately visible in the UI without needing to read German.

---

## Specifying Translation Texts at Runtime

The compile-time approach produces optimal bundles, but some deployment scenarios demand a single build that loads translations at startup. Angular supports this through `loadTranslations()`:

```typescript
// main.ts
import { loadTranslations } from '@angular/localize';
import { bootstrapApplication } from '@angular/platform-browser';
import { AppComponent } from './app/app.component';
import { appConfig } from './app/app.config';

async function bootstrap(): Promise<void> {
  const locale = document.documentElement.lang || 'en-US';

  if (locale !== 'en-US') {
    const translations = await fetch(`/assets/i18n/${locale}.json`)
      .then(r => r.json());
    loadTranslations(translations);
  }

  await bootstrapApplication(AppComponent, appConfig);
}

bootstrap();
```

The translation JSON file is a flat key-value map:

```json
{
  "submitTransaction": "Transaktion absenden",
  "cancelButton": "Abbrechen",
  "transactionListTitle": "Letzte Transaktionen",
  "loadMore": "Mehr laden"
}
```

`loadTranslations()` must be called **before** `bootstrapApplication()`. Once the application boots, `$localize` resolves every tagged template against the loaded translations. If a key is missing, the original source string serves as the fallback.

This approach trades the zero-overhead guarantee of compile-time i18n for deployment flexibility. For FinancialApp, it makes sense when the same build artifact is deployed to multiple regions and the locale is determined by a user profile setting rather than a URL prefix.

---

## Taking Grammatical Forms into Account

English pluralization is relatively simple: one item, multiple items. German adds genitive and dative variations. Arabic has six plural forms. ICU (International Components for Unicode) message expressions handle this complexity inside Angular templates.

### Plural Forms

FinancialApp's transaction list displays a count message that must adapt to the number:

```html
<span i18n="@@txCount">
  {count, plural,
    =0 {No transactions found}
    =1 {1 transaction found}
    other {{{count}} transactions found}
  }
</span>
```

The ICU `plural` expression selects the correct form based on the `count` value. The categories (`=0`, `=1`, `other`) follow CLDR (Common Locale Data Repository) rules. A German translation might use:

```xml
<target>
  {count, plural,
    =0 {Keine Transaktionen gefunden}
    =1 {1 Transaktion gefunden}
    other {{count} Transaktionen gefunden}
  }
</target>
```

### Select Expressions

Gender and other categorical choices use the `select` expression:

```html
<span i18n="@@txApproval">
  {approverRole, select,
    manager {Approved by the account manager}
    auditor {Reviewed by the compliance auditor}
    other {Processed automatically}
  }
</span>
```

### Nested Expressions

ICU expressions can be nested for complex grammatical scenarios. A message that varies by both count and gender is valid, though readability suffers quickly. For deeply nested cases, consider breaking the message into separate translation units or computing the appropriate key in TypeScript.

---

## Supporting Different Formats

Translating labels is half the i18n challenge. The other half is formatting dates, numbers, and currencies according to locale conventions. Angular's built-in pipes handle this using the CLDR data bundled with `@angular/common`.

### Currency Formatting

FinancialApp displays transaction amounts in the user's preferred format:

```html
<td>{{ transaction.amount | currency:transaction.currencyCode:'symbol':'1.2-2' }}</td>
```

The same value `1234.56` renders differently by locale:

| Locale  | Output          |
| ------- | --------------- |
| `en-US` | `$1,234.56`     |
| `de-DE` | `1.234,56 $`    |
| `ja-JP` | `$1,234.56`     |

To set the application-wide locale for pipe formatting, provide `LOCALE_ID`:

```typescript
import { LOCALE_ID } from '@angular/core';
import { registerLocaleData } from '@angular/common';
import localeDe from '@angular/common/locales/de';

registerLocaleData(localeDe);

export const appConfig: ApplicationConfig = {
  providers: [
    { provide: LOCALE_ID, useValue: 'de-DE' },
  ],
};
```

### Date Formatting

Transaction dates follow the same locale-aware pattern:

```html
<td>{{ transaction.date | date:'mediumDate' }}</td>
```

| Locale  | Output           |
| ------- | ---------------- |
| `en-US` | `Apr 16, 2026`   |
| `de-DE` | `16.04.2026`     |

### Number Formatting

Large portfolio values use `DecimalPipe`:

```html
<span>{{ portfolio.totalValue | number:'1.2-2' }}</span>
```

The thousands separator and decimal marker flip between locales automatically. This formatting is driven entirely by `LOCALE_ID` -- no conditional logic required.

---

## Outlook: Community Solutions

Angular's built-in i18n pipeline excels at compile-time translation with zero runtime cost, but its design makes certain workflows difficult. Switching locales without a full page reload, lazy-loading translations for specific feature modules, and storing translations in a database rather than XLF files all require workarounds or external libraries.

### Transloco

[Transloco](https://github.com/jsverse/transloco) is a modern i18n library built specifically for Angular. It offers runtime translation switching, lazy loading of translation files per route, and a plugin ecosystem for translation management. Its API integrates naturally with Angular's dependency injection:

```typescript
import { TranslocoModule, provideTransloco } from '@jsverse/transloco';

export const appConfig: ApplicationConfig = {
  providers: [
    provideTransloco({
      config: {
        availableLangs: ['en', 'de', 'ja'],
        defaultLang: 'en',
        reRenderOnLangChange: true,
      },
    }),
  ],
};
```

In templates, Transloco uses a structural directive or pipe:

```html
<h2 *transloco="let t">{{ t('transactionList.title') }}</h2>
```

### ngx-translate

[ngx-translate](https://github.com/ngx-translate/core) is the longest-standing community i18n library. While it has not adopted Angular's latest patterns (signals, standalone APIs), its large install base means extensive community support and integration guides. Teams maintaining legacy applications may find it the path of least resistance.

### Choosing Between Built-in and Community Solutions

| Criterion                      | @angular/localize | Transloco          |
| ------------------------------ | ----------------- | ------------------ |
| Runtime language switching     | No (reload)       | Yes                |
| Lazy-loaded translations       | Per build         | Per route/module   |
| Zero runtime overhead          | Yes               | Small overhead     |
| ICU expression support         | Full              | Via plugin         |
| Professional TMS integration   | XLF native        | JSON-based         |

For FinancialApp, the built-in pipeline is the right choice when each regional deployment runs its own build artifact. Transloco becomes attractive if the application serves multiple locales from a single deployment and users can switch languages in their profile settings.

---

## Runtime Locale Switching vs Build-per-Locale

Angular gives you two distinct deployment models for i18n, and the choice between them is an architecture decision -- not a feature toggle you flip later. The earlier sections covered each mechanism in isolation; this one puts them side by side so the trade-offs are explicit.

### Build-per-Locale (the Angular Default)

`ng build --localize` produces one output directory per locale, each containing a complete bundle with the translated strings already compiled into the JavaScript. The translation lookup happens at build time: every `$localize` tag is replaced with the final string, and the runtime never sees a translation table.

The upside is smaller JavaScript per locale and zero runtime translation overhead -- the user's browser downloads only the strings it needs, in the form it needs them. The downside is a larger build surface. Adding a new locale means another `ng build` configuration, another artifact in the CDN, another entry in the edge routing table, and another round of smoke tests. A six-locale financial platform is six builds per deployment.

### Runtime Locale Switching

`loadTranslations()` (covered in the [earlier section](#specifying-translation-texts-at-runtime)) inverts the model. A single bundle ships everywhere. The client fetches a translation JSON file at startup, and the locale is decided at runtime from a cookie, a URL segment, or the user's profile.

The upside is operational simplicity: one build, one artifact, one CDN path. Switching a user from English to Japanese is a profile update and a page reload, not a redeploy. The downside is a larger first-paint payload (the bundle must contain the source strings plus the translation JSON) and a brief asynchronous step before `bootstrapApplication()` can run.

```typescript
// main.ts -- runtime locale switching
import { loadTranslations } from '@angular/localize';
import { bootstrapApplication } from '@angular/platform-browser';
import { AppComponent } from './app/app.component';
import { appConfig } from './app/app.config';

async function bootstrap(): Promise<void> {
  const locale = document.cookie.match(/locale=([^;]+)/)?.[1] ?? 'en-US';

  if (locale !== 'en-US') {
    const translations = await fetch(`/i18n/${locale}.json`).then(r => r.json());
    loadTranslations(translations);
  }

  document.documentElement.lang = locale;
  await bootstrapApplication(AppComponent, appConfig);
}

bootstrap();
```

### Choosing

- **Build-per-locale** is the right choice for public, SEO-critical surfaces where milliseconds of first paint and search rankings outweigh deployment simplicity. Marketing sites, pricing pages, and any page a logged-out user might land on benefit most.
- **Runtime switching** is the right choice for authenticated applications -- the portfolio dashboard, the trading screen -- where the user has already signed in, deployment simplicity outweighs small payload differences, and user-initiated locale changes from a settings screen are a feature, not an outage.

A hybrid is common in large financial platforms: a build-per-locale marketing site plus a runtime-switching authenticated application. The two share the same XLF files but ship through different pipelines.

---

## Translation Pipeline in CI

Marking strings, extracting them, and shipping bundles is a manual workflow that does not survive a team of ten shipping three features a week. A CI pipeline turns translation into a background activity that the application team almost never thinks about.

### The Extract -> Translate -> Merge Loop

Every successful CI run on the main branch runs `ng extract-i18n` and compares the result to the checked-in source XLF. If there are new, changed, or removed translation units, the pipeline:

1. Pushes the updated source XLF to the translation management system (TMS).
2. Opens translator tasks in the TMS for every new or changed unit.
3. Pulls completed translations back into locale-specific XLF files as they finish.
4. Opens a pull request back to the main branch with the updated locale files.

Several products automate this loop end-to-end. **Phrase** and **Lokalise** offer direct Git integrations with webhook-driven sync. **Transifex** is the long-standing enterprise option with Angular-aware XLF parsers. **Crowdin** is popular with open-source projects. For smaller teams, a plain Git-based workflow works too: translators check out the repo, edit the locale XLF files, and open pull requests like any other contributor.

### The Untranslated-String CI Gate

The rule that matters most is the gate that refuses to deploy an untranslated string to production. FinancialApp's deploy pipeline runs a short script that parses each production locale's XLF and fails if it contains any `<target state="new">` elements or missing `<target>` tags. Fallback to the source string is fine during development; shipping "Submit Transaction" to a Japanese user in production is not.

The same script can emit a lighter warning for staging locales or allow a short grace window ("new strings may ship untranslated for up to 24 hours while translators catch up"). The important property is that the check is automated: a new string added in feature branch X triggers a translator task before X merges to main, not in the post-release triage meeting.

---

## Financial Formatting Edge Cases

Translating labels is the easy half of i18n. The hard half is formatting numbers, currencies, and dates correctly for each locale -- and doing so in a way that never misleads a user about how much money is in their account. The [earlier locale-aware pipes section](#supporting-different-formats) covers the basic cases; this section catalogues the edge cases a financial application must get right.

### Currency and the Multi-Currency Ambiguity Problem

The Angular `CurrencyPipe` and the underlying `Intl.NumberFormat` handle straightforward cases correctly. The problem is that `1234.56` in `USD` renders as `$1,234.56` and the same value in `CAD` also renders as `$1,234.56` -- the two strings are identical. A user who holds both currencies cannot tell the balances apart from the rendered string alone.

For any multi-currency context, always show the ISO 4217 currency code alongside the symbol. FinancialApp wraps `CurrencyPipe` in a small `FinCurrencyPipe` that enforces this:

```typescript
// fin-currency.pipe.ts
import { Pipe, PipeTransform } from '@angular/core';

@Pipe({ name: 'finCurrency', standalone: true })
export class FinCurrencyPipe implements PipeTransform {
  transform(value: number, currency: string, locale?: string): string {
    const formatted = new Intl.NumberFormat(locale ?? navigator.language, {
      style: 'currency',
      currency,
      currencyDisplay: 'narrowSymbol',
    }).format(value);
    return `${formatted} ${currency}`;
  }
}
```

Used as `{{ holding.value | finCurrency: holding.currency }}`, it renders `$1,234.56 USD` and `$1,234.56 CAD` unambiguously. The ISO code is also easier to screen-read and easier to grep in logs.

### Number Formatting Quirks

The unremarkable number `1234567.89` renders in ways that routinely trip up financial software:

| Locale  | Output              | Notes                                                 |
| ------- | ------------------- | ----------------------------------------------------- |
| `en-US` | `1,234,567.89`      | Comma thousands, dot decimal                          |
| `de-DE` | `1.234.567,89`      | Dot thousands, comma decimal -- opposite of en-US     |
| `fr-FR` | `1 234 567,89`      | Non-breaking space as thousands separator             |
| `en-IN` | `12,34,567.89`      | Indian grouping: two-digit groups after the first three |

Chinese 万 (10,000) and 億 (100,000,000) grouping is culturally expected for consumer-facing displays but is not emitted by `Intl.NumberFormat` by default. Use `notation: 'compact'` with `compactDisplay: 'short'` for round numbers, and a custom formatter for exact values where the convention matters.

### Date and Financial Week Conventions

Calendar conventions diverge too. Weeks start on Sunday in the US and Monday in most of Europe. Trading weeks run Sunday--Thursday in Saudi Arabia and Israel instead of Monday--Friday. Japan occasionally uses the Reiwa calendar era alongside the Gregorian year in official contexts. For FinancialApp's "week-to-date" performance chart, a wrong week boundary looks like a product bug.

Angular's `DatePipe` delegates to CLDR via `Intl.DateTimeFormat`, which gets the format right. For derived concepts like "week start", use `new Intl.Locale('ar-SA').weekInfo?.firstDay` rather than hard-coding a rule. Fiscal-year start dates are not in CLDR and must come from business configuration, not from the locale.

---

## Summary

Angular's i18n infrastructure provides a professional-grade translation pipeline that scales from a two-locale marketing site to a multi-region financial platform. The key steps are:

1. **Install** `@angular/localize` to enable the `$localize` runtime and extraction tooling.
2. **Mark** translatable text with the `i18n` attribute in templates and `$localize` in TypeScript.
3. **Extract** translation units with `ng extract-i18n` into XLIFF files that integrate with professional translation services.
4. **Build** locale-specific bundles with `ng build --localize` for zero-overhead translated output.
5. **Format** dates, numbers, and currencies with Angular's locale-aware pipes and `LOCALE_ID`.
6. **Handle grammar** with ICU plural and select expressions for linguistically correct messages.

For FinancialApp, the translation files live in `financial-app/src/locale/messages.*.xlf`. Each deployment region receives a pre-compiled bundle with all strings resolved at build time -- no dictionary lookups, no runtime parsing, no performance tax on the end user.

When the built-in pipeline's compile-time model does not fit your deployment architecture, Transloco and ngx-translate offer runtime alternatives with different trade-off profiles. Whichever path you choose, the discipline of marking every user-facing string from day one is what separates applications that can expand internationally from those that require a painful retrofit.
