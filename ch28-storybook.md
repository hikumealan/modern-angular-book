# Chapter 28: Storybook for Component Libraries

A design system that lives only in production is a design system nobody trusts. Developers copy-paste from old screens because they cannot see what components exist. Designers file bugs because the spacing "looks wrong" but have no isolated view to prove it. QA tests every permutation manually because there is no gallery of states. Storybook solves all three problems by giving shared components an independent development environment -- one that doubles as a testing platform and documentation site.

This chapter integrates Storybook into the FinancialApp workspace we have been building since [Chapter 8](ch08-architecture.md). We will write stories for the design-system primitives introduced in [Chapter 27](ch27-material-design-system.md), wire up interaction and accessibility testing, connect the Storybook MCP addon to AI coding agents, and publish the result as living documentation that stays current with every merge.

> **Companion code:** `.storybook/` config and stories for all shared components live in `financial-app/`. The `@storybook/addon-a11y` and `@storybook/addon-mcp` addons are configured in `.storybook/main.ts`.

---

## Setting Up Storybook in an Nx Workspace

Storybook ships a framework-aware initializer that detects Angular and configures the build pipeline automatically:

```bash
npx storybook@latest init
```

In an Nx workspace you can also use the dedicated generator, which wires Storybook targets into `project.json` for each library:

```bash
nx g @nx/storybook:configuration shared-ui --uiFramework=@storybook/angular
```

Either approach produces a `.storybook/` directory at the workspace root containing two files: `main.ts` for build configuration and `preview.ts` for global decorators and parameters.

### Configuring for Angular

The generated `main.ts` needs adjustment for a design-system workspace. Set the framework, point the story glob at shared libraries, and register the addons we will use throughout this chapter:

```typescript
// .storybook/main.ts
import type { StorybookConfig } from '@storybook/angular';

const config: StorybookConfig = {
  stories: [
    '../libs/shared/ui/src/**/*.stories.ts',
    '../libs/shared/design-system/src/**/*.stories.ts',
  ],
  addons: [
    '@storybook/addon-essentials',
    '@storybook/addon-a11y',
    '@storybook/addon-mcp',
  ],
  framework: {
    name: '@storybook/angular',
    options: {},
  },
};

export default config;
```

The `preview.ts` file imports your design-system stylesheet so that tokens, typography, and spacing apply to every story without per-story configuration:

```typescript
// .storybook/preview.ts
import type { Preview } from '@storybook/angular';
import '../libs/shared/design-system/src/styles/tokens.scss';

const preview: Preview = {
  parameters: {
    controls: {
      matchers: {
        color: /(background|color)$/i,
        date: /Date$/i,
      },
    },
  },
};

export default preview;
```

With the Nx target in place, launch the dev server with `nx storybook shared-ui`. Storybook starts on `localhost:6006` with hot reload, and every story file matching the glob appears in the sidebar.

---

## Writing Stories

### Story Format

Storybook uses Component Story Format (CSF), where each `.stories.ts` file exports a default `Meta` object describing the component and named exports for individual states:

```typescript
// libs/shared/ui/src/fin-button/fin-button.stories.ts
import type { Meta, StoryObj } from '@storybook/angular';
import { FinButtonComponent } from './fin-button.component';

const meta: Meta<FinButtonComponent> = {
  title: 'Design System/FinButton',
  component: FinButtonComponent,
  tags: ['autodocs'],
};

export default meta;
type Story = StoryObj<FinButtonComponent>;

export const Primary: Story = {
  args: { variant: 'primary', label: 'Submit' },
};
```

The `tags: ['autodocs']` entry tells Storybook to generate a documentation page from the component's inputs, outputs, and JSDoc comments. The `title` property controls sidebar organization -- slashes create nested groups.

### Design-System Primitives

Before documenting domain wrappers, establish stories for the raw design tokens. These stories serve as the single source of truth for the visual language of FinancialApp.

**Icon gallery.** `FinIcon` renders icons from the design system's SVG sprite. A gallery story iterates over the icon registry and renders every available icon at multiple sizes:

```typescript
// libs/shared/design-system/src/fin-icon/fin-icon.stories.ts
import type { Meta, StoryObj } from '@storybook/angular';
import { FinIconComponent } from './fin-icon.component';
import { ICON_REGISTRY } from '../tokens/icon-registry';

const meta: Meta<FinIconComponent> = {
  title: 'Design System/Primitives/FinIcon',
  component: FinIconComponent,
};

export default meta;

export const AllIcons: StoryObj<FinIconComponent> = {
  render: () => ({
    template: `
      <div style="display: flex; flex-wrap: wrap; gap: 24px;">
        @for (name of icons; track name) {
          <div style="text-align: center; width: 80px;">
            <fin-icon [name]="name" size="md" />
            <small>{{ name }}</small>
          </div>
        }
      </div>
    `,
    props: { icons: Object.keys(ICON_REGISTRY) },
  }),
};
```

**Typography ramp.** A story that renders every step of the type scale -- from `fin-display-large` through `fin-caption` -- gives designers and developers a shared vocabulary. Each paragraph uses the corresponding CSS class from the design-system tokens, so the story doubles as a visual regression test for font sizing.

**Token swatches.** Color palette stories visualize semantic tokens (primary, error, surface) alongside their hex values. When a designer asks "which blue are we using for links?", the swatch story answers definitively.

**Theme toggle.** A toolbar decorator that switches between light and dark themes lets reviewers verify both modes without leaving Storybook. We will build this decorator in the Controls section below.

### Domain Wrappers

With primitives documented, move to the components that compose them into domain-specific widgets.

**FinButton.** Primary, danger, and ghost variants with combinations of icon, loading spinner, and disabled state. Each variant is a named export with its own `args`:

```typescript
export const Danger: Story = {
  args: { variant: 'danger', label: 'Delete Account' },
};

export const WithIcon: Story = {
  args: { variant: 'primary', label: 'Add Transaction', icon: 'add' },
};

export const Loading: Story = {
  args: { variant: 'primary', label: 'Processing...', loading: true },
};
```

**FinDataTable.** The data table is the most complex shared component. Its stories cover populated data, empty state, sorting, pagination, and row selection:

```typescript
// libs/shared/ui/src/fin-data-table/fin-data-table.stories.ts
import type { Meta, StoryObj } from '@storybook/angular';
import { FinDataTableComponent } from './fin-data-table.component';
import { MOCK_TRANSACTIONS } from '../../test/mock-data';

const meta: Meta<FinDataTableComponent> = {
  title: 'Domain/FinDataTable',
  component: FinDataTableComponent,
  tags: ['autodocs'],
};

export default meta;
type Story = StoryObj<FinDataTableComponent>;

export const WithData: Story = {
  args: {
    data: MOCK_TRANSACTIONS,
    columns: ['date', 'description', 'amount', 'category'],
    sortable: true, paginate: true, pageSize: 10,
  },
};

export const EmptyState: Story = {
  args: { data: [], columns: ['date', 'description', 'amount'] },
};

export const WithRowSelection: Story = {
  args: {
    data: MOCK_TRANSACTIONS.slice(0, 5),
    columns: ['date', 'description', 'amount'],
    selectable: true,
  },
};
```

**FinFormField.** Stories for valid, invalid, required, and disabled states let developers see the error patterns from [Chapter 23](ch23-error-handling.md) in isolation. The `Invalid` story sets `errors` and `touched` to show the error message; the `Required` story shows the required marker without a value.

**FinAccountSelector.** Three stories cover the progression: empty (no accounts loaded), populated (with mock account data), and type-ahead filtering (demonstrating the search-as-you-type behavior).

**FinCurrencyInput.** Stories for USD, EUR, and JPY formatting verify locale-aware behavior -- particularly that the yen symbol renders without decimal places while the dollar and euro show two.

### Existing Feature Components

Stories are not limited to the shared library. A `TransactionCard` story renders mock credit and debit transactions, and an `AccountCard` story renders mock savings and checking accounts. An `ErrorBoundary` story renders the component in both its normal state and its error fallback, confirming that the fallback UI is styled and accessible without triggering a real error.

```typescript
// libs/shared/ui/src/transaction-card/transaction-card.stories.ts
export const CreditTransaction: Story = {
  args: {
    transaction: {
      id: 1, description: 'Salary Deposit', amount: 5200.00,
      type: 'credit', category: 'income', date: '2026-04-01',
    },
  },
};

export const DebitTransaction: Story = {
  args: {
    transaction: {
      id: 2, description: 'Office Supplies', amount: -127.43,
      type: 'debit', category: 'business', date: '2026-04-03',
    },
  },
};
```

---

## Controls, Args, and Decorators

### Interactive Controls

Storybook's controls panel auto-generates form fields from a component's inputs. Refine them with `argTypes` to provide dropdown selectors instead of free-text fields:

```typescript
const meta: Meta<FinButtonComponent> = {
  title: 'Design System/FinButton',
  component: FinButtonComponent,
  argTypes: {
    variant: { control: 'select', options: ['primary', 'danger', 'ghost'] },
    loading: { control: 'boolean' },
    disabled: { control: 'boolean' },
  },
};
```

Reviewers can now toggle between variants, flip loading on and off, and test disabled states without writing new stories. This interactive exploration catches edge cases -- what does a loading ghost button look like? -- that static stories miss.

### Decorators for Angular Services

Many shared components depend on Angular services: `ThemeService` for dark mode, `LocaleService` for currency formatting, or injected data stores. Decorators wrap every story in the necessary providers using `applicationConfig`:

```typescript
// .storybook/preview.ts
import { applicationConfig } from '@storybook/angular';
import { provideAnimations } from '@angular/platform-browser/animations';

const preview: Preview = {
  decorators: [
    applicationConfig({
      providers: [provideAnimations(), ThemeService],
    }),
  ],
};
```

For stories that need mock data, apply a decorator at the story level rather than globally:

```typescript
export const WithAccounts: Story = {
  decorators: [
    applicationConfig({
      providers: [
        { provide: AccountService, useValue: { getAccounts: () => of(MOCK_ACCOUNTS) } },
      ],
    }),
  ],
  args: { placeholder: 'Select an account...' },
};
```

Layout wrappers that center content or constrain width work as template decorators, keeping stories visually consistent across the catalog.

---

## Interaction Testing

Static stories verify appearance. Play functions verify *behavior*. A play function runs after the story renders and simulates user interactions using `@storybook/test` utilities:

```typescript
// libs/shared/ui/src/fin-data-table/fin-data-table.stories.ts
import { within, userEvent, expect } from '@storybook/test';

export const SortByAmount: Story = {
  args: {
    data: MOCK_TRANSACTIONS,
    columns: ['date', 'description', 'amount'],
    sortable: true,
  },
  play: async ({ canvasElement }) => {
    const canvas = within(canvasElement);
    const amountHeader = canvas.getByRole('columnheader', { name: /amount/i });

    await userEvent.click(amountHeader);

    const cells = canvas.getAllByRole('cell');
    const amounts = cells
      .filter((_, i) => i % 3 === 2)
      .map((cell) => parseFloat(cell.textContent!.replace(/[$,]/g, '')));

    for (let i = 1; i < amounts.length; i++) {
      await expect(amounts[i]).toBeGreaterThanOrEqual(amounts[i - 1]);
    }
  },
};
```

This play function clicks the "Amount" column header and verifies that the resulting rows are sorted in ascending order. Play functions run in the Storybook UI and can be executed headlessly via `test-storybook` in CI -- bridging the gap between unit tests from [Chapter 7](ch07-testing-vitest.md) and full E2E suites.

Play functions compose. A pagination story can call the sort play function first, then verify that pagination controls update correctly after the sort:

```typescript
export const SortThenPaginate: Story = {
  args: { ...SortByAmount.args, paginate: true, pageSize: 5 },
  play: async (context) => {
    await SortByAmount.play!(context);
    const canvas = within(context.canvasElement);
    await userEvent.click(canvas.getByRole('button', { name: /next/i }));
    await expect(canvas.getAllByRole('row').length).toBeLessThanOrEqual(6);
  },
};
```

---

## Accessibility Testing

The `@storybook/addon-a11y` addon runs axe-core against every story and reports violations directly in the Storybook panel. We added it to `main.ts` earlier -- no per-story configuration is needed. Every story becomes an accessibility test case for free.

The violations panel groups issues by severity (critical, serious, moderate, minor) and links to WCAG success criteria. A `FinButton` story that renders without visible focus styles will show a serious violation, prompting an immediate fix in the design-system stylesheet.

### Layered Accessibility Strategy

The addon-a11y audits complement -- but do not replace -- the accessibility testing strategy from [Chapter 22](ch22-accessibility-aria.md). The three layers work together:

1. **Storybook addon-a11y** catches violations in isolation, during development. Fast feedback, narrow scope.
2. **Unit-level axe-core** (from [Chapter 22](ch22-accessibility-aria.md)) validates components in their test harness, covering dynamic states that static stories might miss.
3. **E2E axe audits** validate the assembled application, catching issues that emerge from component composition -- tab order across a full page, skip links, landmark regions.

No single layer is sufficient. A component that passes in isolation may fail when composed with others. A page-level audit may miss a violation that only appears when a specific component input is set. The three layers together provide confidence.

---

## Storybook MCP Addon

The `@storybook/addon-mcp` addon exposes your component library as structured context to AI coding agents through the Model Context Protocol. When an agent needs to build a new feature, it can query Storybook to discover which components already exist, what inputs they accept, and how they are intended to be used -- instead of inventing new components that duplicate existing ones.

### Configuration

The addon is already included in our `main.ts` addons array. The only additional step is an MCP server entry that your IDE connects to:

```json
{
  "mcpServers": {
    "storybook": {
      "command": "npx",
      "args": ["@storybook/addon-mcp", "--config-dir", ".storybook"]
    }
  }
}
```

### How Agents Use Storybook Context

With the MCP server running, an AI coding agent in Cursor, VS Code, or Claude can:

- **Discover components.** "What buttons does the design system provide?" returns `FinButton` with its variant, loading, and icon inputs -- saving the agent from grepping through source files.
- **Match existing components.** When asked to build a new account summary page, the agent sees that `FinDataTable`, `FinAccountSelector`, and `TransactionCard` already exist and composes them rather than generating markup from scratch.
- **Generate correct usage.** The agent sees real story examples showing how `FinCurrencyInput` handles JPY formatting or how `FinFormField` displays validation errors, and applies the same patterns in new code.
- **Update stories alongside components.** When the agent modifies a component's API -- adding a new input or renaming a variant -- it can update the corresponding stories in the same change, keeping documentation current.

The MCP addon turns your Storybook from a human documentation tool into a machine-readable component catalog. In a codebase with dozens of shared components, this eliminates the most common source of design-system drift: developers (and their AI assistants) not knowing what already exists.

---

## Publishing as Living Documentation

### Static Build

Storybook compiles to static HTML that can be hosted anywhere:

```bash
npx storybook build -o dist/storybook
```

The output is a self-contained site -- no Node server required. In the Nx workspace, wire this as a target in `project.json` using the `@storybook/angular:build-storybook` executor so that `nx build-storybook shared-ui` produces output alongside the app.

### Visual Review with Chromatic

Chromatic captures a screenshot of every story on every commit and diffs them against the previous baseline. When a CSS change subtly shifts the padding on `FinButton`, Chromatic flags the visual difference for review before it reaches production. This catches the kind of "it looks slightly wrong" regressions that neither unit tests nor axe audits detect.

Integrate Chromatic into your CI pipeline:

```yaml
# .github/workflows/storybook.yml
storybook:
  runs-on: ubuntu-latest
  steps:
    - uses: actions/checkout@v4
      with: { fetch-depth: 0 }
    - uses: actions/setup-node@v4
      with: { node-version: 22 }
    - run: npm ci
    - run: npx chromatic --project-token=${{ secrets.CHROMATIC_TOKEN }}
```

### CI Integration

Deploy Storybook alongside the application so that every pull request has a preview link. Combined with Chromatic's visual diffs, this creates a review workflow where design regressions are caught before code review begins:

1. Developer pushes a branch.
2. CI builds Storybook and deploys it to a preview URL.
3. Chromatic runs visual diffs and posts results to the PR.
4. Interaction tests run via `test-storybook` in CI.
5. Accessibility audits flag any new violations.
6. The reviewer opens the preview link, inspects the affected stories, and approves or requests changes.

This workflow turns Storybook from a local development tool into an integral part of the quality pipeline.

---

## Summary

Storybook gives a component library what tests give application logic: confidence that things work as intended, and early warning when they break. In this chapter we integrated Storybook into the FinancialApp Nx workspace and wrote stories across three layers:

- **Design-system primitives** -- icon galleries, typography ramps, color swatches, and theme toggles -- establish the visual vocabulary and serve as the canonical reference for tokens defined in [Chapter 27](ch27-material-design-system.md).
- **Domain wrappers** -- `FinButton`, `FinDataTable`, `FinFormField`, `FinAccountSelector`, `FinCurrencyInput` -- document every variant and edge-case state, from loading spinners to validation errors.
- **Feature components** -- `TransactionCard`, `AccountCard`, `ErrorBoundary` -- prove that business-level components render correctly in isolation.

Beyond documentation, Storybook provides three testing capabilities. **Play functions** simulate user interactions and verify behavior without a full E2E harness. **The addon-a11y** runs axe-core audits against every story, complementing the unit-level and E2E accessibility testing from [Chapter 22](ch22-accessibility-aria.md). **Chromatic** captures visual snapshots and diffs them on every commit, catching CSS regressions that no other tool detects.

The **MCP addon** bridges the gap between human documentation and AI-assisted development. By exposing the component catalog through the Model Context Protocol, it ensures that coding agents reuse existing components rather than reinventing them -- keeping the design system coherent as both humans and machines contribute to the codebase.

A design system is only as good as its documentation, and documentation is only as good as its freshness. Storybook, published to CI and wired into the review workflow, stays current by construction: every component change is a story change, and every story change is a visual diff.
