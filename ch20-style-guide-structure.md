# Chapter 20: Coding Style Guide & Project Structure

An architecture diagram tells you where modules live and how they communicate. It says nothing about whether the team names files consistently, formats code the same way, or agrees on where to put a `computed()` versus a method. Style guides fill that gap. They are the low-level complement to the high-level patterns established in [Chapter 8](ch08-architecture.md) -- and in many ways, they matter more for day-to-day productivity.

A team without a style guide wastes time on formatting debates in code reviews, produces inconsistent code that confuses new hires, and gradually drifts into a state where every file looks like it was written by a different team -- because it was. A team with an enforced style guide eliminates an entire class of review friction, onboards developers faster (they read one document instead of absorbing conventions through osmosis), and keeps the codebase feeling cohesive even as it grows.

This chapter codifies the conventions we recommend for modern Angular projects, explains the rationale behind each one, and shows how to enforce them automatically with ESLint, Prettier, and EditorConfig. The examples continue with the FinancialApp, the personal finance management application we have been building since [Chapter 1](ch01-getting-started.md).

---

## Angular Official Style Guide Conventions

The Angular team maintains an official style guide that has evolved alongside the framework. Many of its recommendations are simple enough that teams adopt them without thinking -- until someone doesn't, and a naming collision or an ambiguous file purpose costs an afternoon of debugging. The conventions below are the ones worth codifying explicitly.

### Naming

Angular's naming conventions follow a consistent formula: **hyphenated file names that mirror the TypeScript identifier**, suffixed with the artifact type. A component class named `TransactionListComponent` lives in `transaction-list.component.ts`. Its template lives in `transaction-list.component.html`, its styles in `transaction-list.component.scss`. The one-to-one mapping between class name and file name means a developer can find the source file for any symbol by converting PascalCase to kebab-case.

The FinancialApp follows this pattern throughout:

```
src/app/features/portfolios/
  portfolio-dashboard.component.ts
  portfolio-dashboard.component.html
  portfolio-dashboard.component.scss
  portfolio-detail.component.ts
  portfolio-detail.component.html
  portfolio-detail.component.scss
  portfolio.service.ts
  portfolio.model.ts
```

Services drop the `Component` suffix and use `.service.ts`. Models use `.model.ts`. Guards use `.guard.ts`. The pattern is mechanical, which is the point -- naming should never require a judgment call.

### One Concept Per File

Every file should export a single primary concept. A component file exports one component. A service file exports one service. This rule is easy to follow for components and services but tempting to violate for models. When the `Account` and `Transaction` interfaces are small, putting them in a shared `models.ts` file feels efficient. It becomes a problem when the file grows to 200 lines and a developer searching for `HoldingSummary` has to scroll past twelve unrelated types.

In the FinancialApp, each domain model gets its own file:

```typescript
// src/app/features/accounts/account.model.ts
export interface Account {
  id: number;
  name: string;
  type: 'checking' | 'savings' | 'investment';
  balance: number;
  currency: string;
  ownerId: number;
}
```

When two types are genuinely coupled -- an interface and its factory function, or a type and its type guard -- they can share a file. The test is whether a consumer would ever import one without the other.

### Project Layout

All application code lives under `src/`. The entry point is `main.ts`, which bootstraps the application and provides the root configuration. This is Angular's convention since standalone components replaced `NgModule`, and it keeps the boot sequence in a single, readable location.

Within `src/app/`, code is grouped by feature area. The FinancialApp's top-level structure reflects its business domains:

```
src/app/
  features/
    accounts/
    transactions/
    portfolios/
    clients/
  shared/
    guards/
    interceptors/
    pipes/
    ui/
  app.component.ts
  app.config.ts
  app.routes.ts
```

[Chapter 8](ch08-architecture.md) covers vertical slicing, boundary enforcement, and the deeper architectural reasoning behind this layout. This chapter is concerned with the simpler question: given that structure, how should the code inside each folder be written?

---

## Dependency Injection Preferences

Modern Angular provides two ways to inject dependencies: the `inject()` function and constructor parameter injection. Both work. We recommend `inject()` exclusively for new code, and here is why.

Constructor injection buries dependencies inside a parameter list that grows unwieldy as a component matures:

```typescript
// Avoid: constructor injection
@Component({ /* ... */ })
export class PortfolioDashboardComponent implements OnInit {
  constructor(
    private readonly portfolioService: PortfolioService,
    private readonly route: ActivatedRoute,
    private readonly router: Router,
    private readonly snackBar: MatSnackBar,
    private readonly destroyRef: DestroyRef
  ) {}

  ngOnInit(): void {
    this.loadPortfolios();
  }
}
```

The `inject()` function lets you declare dependencies as class fields, alongside the code that uses them. You can add comments, group related injections, and let TypeScript infer the type:

```typescript
// Prefer: inject() function
@Component({ /* ... */ })
export class PortfolioDashboardComponent implements OnInit {
  private readonly portfolioService = inject(PortfolioService);
  private readonly route = inject(ActivatedRoute);
  private readonly router = inject(Router);
  private readonly snackBar = inject(MatSnackBar);
  private readonly destroyRef = inject(DestroyRef);

  ngOnInit(): void {
    this.loadPortfolios();
  }
}
```

The practical benefits go beyond aesthetics. With `inject()`, you can initialize derived state inline -- a pattern that becomes especially natural with signals:

```typescript
// src/app/features/accounts/account-detail.component.ts
@Component({ /* ... */ })
export class AccountDetailComponent {
  private readonly accountService = inject(AccountService);
  private readonly route = inject(ActivatedRoute);

  readonly accountId = input.required<string>();
  readonly account = computed(() =>
    this.accountService.getById(this.accountId())
  );
}
```

Constructor injection also interacts poorly with `useDefineForClassFields`, a TypeScript compiler option that Angular v21 enables by default. With this option, class fields are defined on the instance before the constructor body runs. If you mix constructor parameters with field initializers that reference `this`, the behavior can be surprising. Using `inject()` for everything avoids this category of bugs entirely.

---

## Component and Directive Conventions

### Ordering Class Members

A consistent ordering of class members makes components scannable. The convention we follow in the FinancialApp places Angular-specific declarations first, followed by derived state, then lifecycle hooks, and finally methods:

```typescript
// src/app/features/clients/client-card.component.ts
@Component({
  selector: 'fa-client-card',
  templateUrl: './client-card.component.html',
  styleUrl: './client-card.component.scss',
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class ClientCardComponent implements OnInit {
  // 1. Inputs, outputs, models, queries
  readonly client = input.required<Client>();
  readonly selected = model(false);
  readonly expanded = output<boolean>();
  private readonly avatar = viewChild<ElementRef>('avatar');

  // 2. Injected dependencies
  private readonly clientService = inject(ClientService);

  // 3. Derived / computed state
  protected readonly fullName = computed(
    () => `${this.client().firstName} ${this.client().lastName}`
  );
  protected readonly riskLabel = computed(
    () => this.client().riskProfile.toUpperCase()
  );

  // 4. Lifecycle hooks
  ngOnInit(): void {
    this.loadAvatar();
  }

  // 5. Public / protected methods
  protected toggleExpanded(): void {
    this.expanded.emit(!this.selected());
  }

  // 6. Private methods
  private loadAvatar(): void {
    this.clientService.getAvatar(this.client().id);
  }
}
```

This ordering is not arbitrary. A reviewer scanning the file sees the component's API surface (inputs, outputs) before its implementation details (private methods). The pattern holds regardless of component size.

### Access Modifiers: `protected` and `readonly`

Members accessed only from the component's template should be `protected`. This signals intent -- the member is part of the template's contract but not part of the component's public API for parent components. In the example above, `fullName` and `riskLabel` are `protected` because they are consumed by the template but should not be referenced from outside the component.

Properties initialized by Angular's reactive primitives -- `input()`, `output()`, `model()`, and signal queries -- should be `readonly`. Angular sets these up during component initialization and they should never be reassigned:

```typescript
readonly client = input.required<Client>();
readonly selected = model(false);
readonly expanded = output<boolean>();
```

The `readonly` modifier catches accidental reassignment at compile time, which is especially valuable when refactoring.

### Class and Style Bindings

Angular provides multiple ways to apply dynamic classes and styles. For single bindings, prefer the native `[class.active]` and `[style.width.px]` syntax over `ngClass` and `ngStyle`:

```html
<!-- Prefer: direct bindings -->
<div [class.active]="isActive()" [style.opacity]="opacity()">
  {{ label() }}
</div>

<!-- Avoid: ngClass / ngStyle for simple cases -->
<div [ngClass]="{ 'active': isActive() }" [ngStyle]="{ 'opacity': opacity() }">
  {{ label() }}
</div>
```

Direct bindings are easier to read and produce less overhead. Reserve `ngClass` and `ngStyle` for cases where the set of classes or styles is genuinely dynamic -- for example, when a component receives a map of class names from a parent.

### Event Handler Naming

Name event handlers for what they **do**, not the event that triggers them. A button click that saves a transaction should call `saveTransaction()`, not `handleClick()` or `onClick()`. The name should make the template self-documenting:

```html
<!-- Prefer: descriptive action -->
<button (click)="saveTransaction()">Save</button>
<button (click)="cancelEditing()">Cancel</button>

<!-- Avoid: generic event names -->
<button (click)="handleClick()">Save</button>
<button (click)="onButtonClick()">Cancel</button>
```

When a component emits multiple events, generic names force the reader to trace through the handler's implementation to understand what it does. Descriptive names eliminate that step.

### Lifecycle Hooks

Keep lifecycle hook methods small. When `ngOnInit` grows beyond a few lines, extract the logic into well-named private methods:

```typescript
// src/app/features/portfolios/portfolio-detail.component.ts
@Component({ /* ... */ })
export class PortfolioDetailComponent implements OnInit, OnDestroy {
  ngOnInit(): void {
    this.subscribeToRouteParams();
    this.initializeChartDefaults();
  }

  ngOnDestroy(): void {
    this.disconnectMarketData();
  }

  private subscribeToRouteParams(): void { /* ... */ }
  private initializeChartDefaults(): void { /* ... */ }
  private disconnectMarketData(): void { /* ... */ }
}
```

Always implement the corresponding interface (`OnInit`, `OnDestroy`, `AfterViewInit`) when using a lifecycle hook. The interface does not change runtime behavior -- Angular calls the hook method regardless -- but it communicates intent to readers and enables compile-time checking if the method name is misspelled.

---

## Template Conventions

### Keep Templates Simple

A template that contains complex boolean logic, nested ternary expressions, or arithmetic is a template that should be refactored. Move the logic into a `computed()` signal and bind to the result:

```typescript
// src/app/features/transactions/transaction-row.component.ts
@Component({ /* ... */ })
export class TransactionRowComponent {
  readonly transaction = input.required<Transaction>();

  protected readonly isHighValue = computed(
    () => Math.abs(this.transaction().amount) > 10_000
  );

  protected readonly displayAmount = computed(() => {
    const t = this.transaction();
    return t.type === 'debit' ? -t.amount : t.amount;
  });
}
```

```html
<!-- Clean template that reads like a specification -->
<tr [class.high-value]="isHighValue()">
  <td>{{ transaction().date }}</td>
  <td>{{ transaction().description }}</td>
  <td>{{ displayAmount() | currency }}</td>
</tr>
```

The template becomes a thin binding layer between the user and the component's state. Every conditional, every transformation, every derived value lives in TypeScript where it can be tested, refactored, and type-checked.

### Native Control Flow

Use Angular's built-in control flow syntax -- `@if`, `@for`, `@switch` -- in all new templates. The structural directive equivalents (`*ngIf`, `*ngFor`, `*ngSwitch`) are still supported but are legacy API. The built-in syntax is not just cosmetically different; it enables better type narrowing, integrates with `@empty` blocks in `@for`, and is optimized by the Angular compiler:

```html
<!-- src/app/features/accounts/account-list.component.html -->
@if (accounts().length > 0) {
  <ul>
    @for (account of accounts(); track account.id) {
      <li>
        <fa-account-card [account]="account" />
      </li>
    } @empty {
      <li class="empty-state">No accounts found.</li>
    }
  </ul>
} @else {
  <p>Loading accounts...</p>
}
```

[Chapter 2](ch02-signal-components.md) and [Chapter 11](ch11-directives-templates.md) cover the control flow syntax in detail. The style guide concern is simpler: pick one syntax and use it everywhere.

---

## ESLint Rule Customization

[Chapter 1](ch01-getting-started.md) introduced ESLint and the `angular-eslint` plugin. Here we extend the default configuration to enforce the conventions described above. The goal is to turn style violations into lint errors that CI catches before a reviewer ever sees the code.

The FinancialApp's `eslint.config.js` adds rules for component member ordering, lifecycle interface enforcement, and `inject()` preference:

```javascript
// eslint.config.js
const angular = require('angular-eslint');
const tseslint = require('typescript-eslint');

module.exports = tseslint.config(
  {
    files: ['**/*.ts'],
    extends: [
      ...tseslint.configs.recommended,
      ...angular.configs.tsRecommended,
    ],
    rules: {
      // Enforce inject() over constructor injection
      '@angular-eslint/prefer-standalone': 'error',

      // Require lifecycle hook interfaces
      '@angular-eslint/use-lifecycle-interface': 'error',

      // Enforce component selector prefix
      '@angular-eslint/component-selector': [
        'error',
        { type: 'element', prefix: 'fa', style: 'kebab-case' },
      ],
      '@angular-eslint/directive-selector': [
        'error',
        { type: 'attribute', prefix: 'fa', style: 'camelCase' },
      ],

      // Disallow ngClass / ngStyle in favor of direct bindings
      '@angular-eslint/no-host-metadata-property': 'error',

      // Discourage empty lifecycle hooks
      '@angular-eslint/no-empty-lifecycle-method': 'error',

      // Enforce output naming (no "on" prefix)
      '@angular-eslint/no-output-on-prefix': 'error',

      // TypeScript strictness
      '@typescript-eslint/explicit-member-accessibility': [
        'error',
        {
          accessibility: 'no-public',
          overrides: { properties: 'off' },
        },
      ],
      '@typescript-eslint/no-unused-vars': [
        'error',
        { argsIgnorePattern: '^_' },
      ],
    },
  },
  {
    files: ['**/*.html'],
    extends: [...angular.configs.templateRecommended],
    rules: {
      // Enforce native control flow over structural directives
      '@angular-eslint/template/prefer-control-flow': 'error',

      // Enforce self-closing tags where applicable
      '@angular-eslint/template/prefer-self-closing-tags': 'error',
    },
  }
);
```

The `explicit-member-accessibility` rule configured with `no-public` deserves explanation. In TypeScript, class members are public by default. Writing `public` explicitly adds noise without information. The rule enforces that developers omit `public` and only annotate members that are `protected`, `private`, or `readonly` -- making the non-default access levels stand out.

These rules are not exhaustive. The full `angular-eslint` rule catalog includes dozens of additional checks. Start with the rules that address your team's most frequent review comments, then add more as conventions solidify.

---

## Prettier Integration

ESLint enforces code quality and Angular-specific conventions. Prettier enforces formatting -- indentation, line length, trailing commas, quote style. Separating these concerns avoids conflicts: ESLint never has an opinion about whitespace, and Prettier never has an opinion about unused variables.

### Configuration

The FinancialApp uses a minimal Prettier configuration that aligns with Angular community conventions:

```json
// .prettierrc
{
  "singleQuote": true,
  "trailingComma": "all",
  "printWidth": 100,
  "tabWidth": 2,
  "semi": true,
  "bracketSpacing": true,
  "arrowParens": "always",
  "endOfLine": "lf",
  "overrides": [
    {
      "files": "*.html",
      "options": {
        "parser": "angular"
      }
    }
  ]
}
```

The `angular` parser for HTML files ensures Prettier understands Angular template syntax, including control flow blocks and binding expressions.

### Avoiding Conflicts with ESLint

Install `eslint-config-prettier` to disable any ESLint rules that would conflict with Prettier's formatting decisions:

```javascript
// eslint.config.js (add to the existing config)
const prettier = require('eslint-config-prettier');

module.exports = tseslint.config(
  // ...existing configs from above...
  prettier,  // Must be last to override conflicting rules
);
```

The `prettier` config must appear last in the configuration array. It does not add any rules -- it only turns off rules from other configs that Prettier handles.

### Pre-commit Enforcement with lint-staged

Style guides that are not enforced are suggestions. Enforce them at the earliest possible moment: the git commit. The combination of `husky` (git hook management) and `lint-staged` (run tools on staged files only) ensures that no unformatted or unlinted code enters the repository:

```json
// package.json (excerpt)
{
  "lint-staged": {
    "*.{ts,js}": ["eslint --fix", "prettier --write"],
    "*.html": ["prettier --write"],
    "*.{scss,css}": ["prettier --write"],
    "*.{json,md}": ["prettier --write"]
  }
}
```

Set up Husky to run lint-staged on pre-commit:

```bash
npx husky init
echo "npx lint-staged" > .husky/pre-commit
```

With this in place, a developer can write code with whatever formatting habits they prefer. The pre-commit hook normalizes everything before it reaches version control. Code reviews never include formatting nits, and the team spends its review energy on logic, naming, and architecture.

---

## EditorConfig

Prettier formats code when it runs. EditorConfig configures the editor to produce correctly formatted code *as you type*. This is a small but meaningful quality-of-life improvement -- without EditorConfig, a developer whose editor defaults to tabs will produce files that Prettier immediately reformats, creating noisy diffs.

The FinancialApp includes an `.editorconfig` at the repository root:

```ini
# .editorconfig
root = true

[*]
charset = utf-8
end_of_line = lf
indent_style = space
indent_size = 2
insert_final_newline = true
trim_trailing_whitespace = true

[*.md]
trim_trailing_whitespace = false
max_line_length = off
```

The `[*.md]` override is important. Markdown uses trailing spaces for line breaks, so trimming them silently breaks formatting. The `max_line_length = off` setting prevents editors from hard-wrapping prose, which produces unreadable diffs when a paragraph is reflowed.

EditorConfig is supported natively by most editors (VS Code, JetBrains IDEs, Vim, and others). No plugin is required in most cases. Combined with Prettier and ESLint, it forms a three-layer defense:

1. **EditorConfig** -- correct defaults as you type
2. **Prettier** -- normalize formatting on save or pre-commit
3. **ESLint** -- enforce code quality and Angular conventions in CI

---

## Summary

Style guides are infrastructure. Like a well-configured build pipeline, they do their best work invisibly -- by preventing problems that would otherwise consume review cycles, onboarding time, and debugging effort.

The conventions covered in this chapter form a coherent set of recommendations for modern Angular projects:

| Convention | Modern (recommended) | Legacy (avoid) |
|---|---|---|
| Dependency injection | `inject()` function | Constructor parameters |
| Lifecycle hooks | Implement interface (`OnInit`) | Hook method without interface |
| Template-only members | `protected` access | `public` (default) |
| Reactive primitives | `readonly` modifier | Mutable assignment |
| Class/style binding | `[class.x]`, `[style.x]` | `ngClass`, `ngStyle` |
| Event handler names | `saveTransaction()` | `handleClick()` |
| Control flow | `@if`, `@for`, `@switch` | `*ngIf`, `*ngFor` |
| Formatting | Prettier (automated) | Manual / ad-hoc |
| Lint enforcement | Pre-commit hook (lint-staged) | Manual `ng lint` runs |
| Editor defaults | `.editorconfig` at repo root | Individual editor settings |

None of these conventions are revolutionary in isolation. Their value is cumulative. A codebase that follows all of them consistently feels *authored* rather than *accumulated* -- and that consistency is what lets a team of five write code that reads like it came from one mind.

The architectural patterns from [Chapter 8](ch08-architecture.md) tell you where to put code. The enforcement rules from [Chapter 14](ch14-monorepos-libraries.md) tell you what boundaries to respect. This chapter tells you how to write the code that lives inside those boundaries -- and how to ensure the team follows through.

> **Companion code:** `.editorconfig`, `.prettierrc`, and updated `eslint.config.js` with Angular-specific style rules live in `financial-app/`.
