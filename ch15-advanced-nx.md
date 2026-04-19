# Chapter 37: Advanced Monorepo & Nx Features

In [Chapter 14](ch14-monorepos-libraries.md) we set up the FinancialApp as an Nx monorepo, drew library boundaries with Sheriff, and watched computation caching turn 14-second builds into 42-millisecond replays. That chapter covered the foundation: workspace creation, project graphs, caching, affected commands, and Nx Cloud's distributed task execution. This chapter goes deeper. We will build custom code generators that enforce FinancialApp conventions in every new feature, connect AI assistants to the workspace through Nx's MCP server, fine-tune task pipelines for complex build orchestration, automate library versioning with `nx release`, and establish a repeatable workflow for keeping every dependency in the monorepo current.

The unifying theme is **automation at the workspace level**. A well-configured monorepo should not just organize code -- it should generate, validate, version, and upgrade it with minimal manual intervention.

---

## Nx Generators for Code Scaffolding

The Angular CLI's `ng generate` command creates components, services, and pipes from built-in schematics. In an Nx workspace, the same capability is available through Nx generators, with broader coverage and tighter integration with the project graph.

### Built-in Generators

The `@nx/angular` plugin ships generators for every common Angular artifact:

```bash
nx generate @nx/angular:component --name=account-detail --project=financial-app
nx generate @nx/angular:service --name=portfolio --project=shared-data-access
nx generate @nx/angular:pipe --name=currency-format --project=shared-ui
nx generate @nx/angular:library --name=compliance --directory=libs/features
```

Each generator respects the project's configuration -- output paths, selector prefixes, style preprocessor settings -- so the generated files land in the right place with the right conventions. The `--dry-run` flag previews what will be created without writing anything to disk. To see every generator available from a plugin, run `nx list @nx/angular`. Third-party plugins like `@nx/js`, `@nx/eslint`, and `@ngrx/signal-store` register their own generators in the same registry.

### Why Built-in Generators Are Not Enough

Built-in generators produce structurally correct Angular code. They do not produce *your* Angular code. A `@nx/angular:component` knows nothing about the FinancialApp's design system wrappers from [Chapter 32](ch22-material-design-system.md), the Vitest testing patterns from [Chapter 7](ch07-testing-vitest.md), or the member ordering conventions from [Chapter 31](ch16-style-guide-governance.md). Every generated component needs manual adjustment -- and that manual work is exactly the kind of repetitive, error-prone task that generators should eliminate.

---

## Custom Workspace Generators

A custom generator encodes your team's conventions into a command. Instead of generating a generic component and hand-editing it to match the style guide, you run a single command that produces a complete, convention-compliant feature scaffold.

### Creating the Generator

Nx workspace generators live in the `tools/` directory. Create a generator for feature scaffolding:

```
financial-app/
  tools/
    generators/
      feature/
        schema.json
        index.ts
        files/
          __name__.component.ts.template
          __name__.component.html.template
          __name__.component.spec.ts.template
          index.ts.template
```

The `schema.json` file defines the generator's inputs -- the questions it asks when invoked:

```json
{
  "$schema": "https://json-schema.org/schema",
  "id": "feature",
  "title": "Feature Component Generator",
  "type": "object",
  "properties": {
    "name": {
      "type": "string",
      "description": "The name of the feature (e.g., transfers, statements)",
      "x-prompt": "What is the feature name?"
    },
    "domain": {
      "type": "string",
      "description": "The business domain this feature belongs to",
      "x-prompt": "Which domain? (e.g., transactions, accounts, portfolios)",
      "enum": ["transactions", "accounts", "portfolios", "clients"]
    }
  },
  "required": ["name", "domain"]
}
```

The `x-prompt` fields enable interactive mode -- Nx will ask for values not provided on the command line. The `enum` on `domain` restricts choices to the FinancialApp's established business areas, preventing typos and ad-hoc domain names.

### The Generator Implementation

The `index.ts` file is a function that receives the schema inputs and manipulates the file system through Nx's `Tree` abstraction:

```typescript
// tools/generators/feature/index.ts
import {
  Tree,
  formatFiles,
  generateFiles,
  names,
  joinPathFragments,
} from '@nx/devkit';

interface FeatureGeneratorSchema {
  name: string;
  domain: string;
}

export default async function featureGenerator(
  tree: Tree,
  schema: FeatureGeneratorSchema
) {
  const nameVariants = names(schema.name);
  const targetDir = joinPathFragments(
    'apps/financial-app/src/app/features',
    schema.domain,
    nameVariants.fileName
  );

  generateFiles(tree, joinPathFragments(__dirname, 'files'), targetDir, {
    ...nameVariants,
    domain: schema.domain,
    domainClass: names(schema.domain).className,
    template: '',
  });

  await formatFiles(tree);
}
```

The `names()` utility converts a single input string into every casing variant a template might need: `fileName` (`account-detail`), `className` (`AccountDetail`), `propertyName` (`accountDetail`), and `constantName` (`ACCOUNT_DETAIL`). The `generateFiles()` function processes every file in the `files/` directory, substituting EJS template expressions and replacing `__name__` in file names with the actual name.

### Template Files

Templates use EJS syntax (`<%= %>`) to inject the name variants. The component template produces a file that already uses the FinancialApp's design system wrappers and follows the member ordering from [Chapter 31](ch16-style-guide-governance.md):

```typescript
// tools/generators/feature/files/__name__.component.ts.template
import { ChangeDetectionStrategy, Component, computed, inject, input } from '@angular/core';
import { FinFormFieldComponent } from '@financial-app/shared/ui/form-field';
import { FinButtonComponent } from '@financial-app/shared/ui/button';
import { FinDataTableComponent } from '@financial-app/shared/ui/data-table';
import { <%= domainClass %>Service } from '@financial-app/shared/data-access';

@Component({
  selector: 'fa-<%= fileName %>',
  templateUrl: './<%= fileName %>.component.html',
  changeDetection: ChangeDetectionStrategy.OnPush,
  imports: [FinFormFieldComponent, FinButtonComponent, FinDataTableComponent],
})
export class <%= className %>Component {
  readonly id = input.required<string>();

  private readonly <%= domain %>Service = inject(<%= domainClass %>Service);

  protected readonly data = computed(() =>
    this.<%= domain %>Service.getById(this.id())
  );
}
```

The test template follows the Vitest pattern established in [Chapter 7](ch07-testing-vitest.md):

```typescript
// tools/generators/feature/files/__name__.component.spec.ts.template
import { describe, it, expect } from 'vitest';
import { render, screen } from '@testing-library/angular';
import { <%= className %>Component } from './<%= fileName %>.component';

describe('<%= className %>Component', () => {
  it('should render', async () => {
    await render(<%= className %>Component, {
      inputs: { id: '1' },
    });
    expect(screen.getByRole('main')).toBeTruthy();
  });
});
```

The barrel export template ensures the new feature is immediately importable:

```typescript
// tools/generators/feature/files/index.ts.template
export { <%= className %>Component } from './<%= fileName %>.component';
```

### Running the Generator

With the generator in place, scaffolding a new feature is a single command:

```bash
nx generate @financial-app/tools:feature --name=transfers --domain=transactions
```

This creates four files under `apps/financial-app/src/app/features/transactions/transfers/`:

```
transfers.component.ts        -- component with design system imports
transfers.component.html       -- empty template ready for markup
transfers.component.spec.ts    -- Vitest test skeleton
index.ts                       -- barrel export
```

Every file follows the team's conventions from day one. No manual cleanup, no review comments about missing `ChangeDetectionStrategy.OnPush`, no forgotten barrel exports. The generator is the style guide made executable.

---

## Nx MCP Server

[Chapter 49](ch47-ai-in-angular.md) introduced the Angular CLI's MCP server, which gives AI coding assistants access to project structure, documentation, best practices, and migration tools. The Nx MCP server operates at a different layer: it exposes **workspace-level** intelligence -- project graphs, task orchestration, generator execution, and CI pipeline analytics.

### Setup

If you use the Nx Console extension in VS Code or Cursor, the MCP server is configured automatically. For manual setup, add it to your `.cursor/mcp.json` alongside the Angular CLI MCP from [Chapter 49](ch47-ai-in-angular.md):

```json
{
  "mcpServers": {
    "angular-cli-mcp": {
      "command": "npx",
      "args": ["-y", "@anthropic/angular-mcp@latest"]
    },
    "nx-mcp": {
      "command": "npx",
      "args": [
        "nx-mcp@latest",
        "--workspace-root", "${workspaceFolder}",
        "--no-minimal"
      ]
    }
  }
}
```

The `--no-minimal` flag enables the full set of tools. In minimal mode, Nx MCP provides only documentation and connectivity tools -- useful when AI skills handle workspace exploration independently. Full mode unlocks the complete toolset.

### Documentation and Guidance Tools

The `nx_docs` tool retrieves Nx-specific documentation for configuration questions and best practices. Where the Angular CLI MCP answers "how do I set up lazy loading?", the Nx MCP answers "how do I configure task pipelines?" or "what is the correct input configuration for caching?" The two servers complement rather than overlap.

### Task Monitoring

During long-running builds or test suites, the Nx MCP provides real-time visibility:

- `nx_current_running_tasks_details` -- lists all currently executing tasks with their status
- `nx_current_running_task_output` -- streams the output of a specific running task

An AI assistant can use these tools to diagnose a stuck build, identify which task in a pipeline is slow, or report progress on a multi-project rebuild without the developer switching to a terminal.

### Visualization

The `nx_visualize_graph` tool renders the project dependency graph or task dependency graph as an interactive diagram. This is the programmatic equivalent of `nx graph`, accessible to AI assistants that need to understand workspace topology before making architectural suggestions. When a developer asks "what would be affected if I change the shared models library?", the assistant can inspect the graph rather than guessing.

### Extended Workspace Tools

In full mode (`--no-minimal`), additional tools unlock deeper workspace interaction: `nx_workspace` and `nx_project_details` for inspecting configuration, `nx_available_plugins` for discovering plugins, and `nx_generators` / `nx_generator_schema` / `nx_run_generator` for listing, inspecting, and executing generators -- including the custom feature generator we built earlier. With these tools, an AI assistant can scaffold code, inspect build configurations, and modify project settings without the developer typing a single CLI command.

### CI and Cloud Analytics Tools

For teams using Nx Cloud, the MCP server bridges local development with CI intelligence. The `ci_information` tool retrieves pipeline status, failed tasks, and error details from recent CI runs. The `update_self_healing_fix` tool reviews and applies (or rejects) AI-generated fixes for CI failures -- when a pipeline fails, Nx Cloud analyzes the failure, generates a proposed fix, and surfaces it through the MCP. The developer's AI assistant can review the fix, apply it locally, and verify it passes before pushing.

Cloud analytics tools -- `cloud_analytics_pipeline_executions_search`, `cloud_analytics_run_details`, and `cloud_analytics_tasks_search` -- provide deeper insight into pipeline performance, useful for tracking flaky tests or identifying tasks that consistently slow down CI.

### Minimal Mode vs. Full Mode

The `--no-minimal` flag is a design decision. When your AI skills (as described in [Chapter 49](ch47-ai-in-angular.md)) already encode workspace exploration knowledge, the MCP server does not need to duplicate that capability. Minimal mode focuses on what skills cannot provide: live documentation, CI connectivity, and cloud analytics. Full mode is appropriate when the MCP server is the primary interface between the AI assistant and the workspace.

---

## Task Pipeline Optimization

[Chapter 14](ch14-monorepos-libraries.md) showed the basics of `dependsOn` and caching in `nx.json`. As the workspace grows, pipeline configuration becomes a first-class architectural concern. The goal is to express the correct dependency relationships between tasks so that Nx can parallelize everything that does not have a strict ordering constraint.

### Configuring Task Dependencies

The `targetDefaults` section in `nx.json` defines how tasks relate to each other across projects:

```json
{
  "targetDefaults": {
    "build": {
      "dependsOn": ["^build"],
      "inputs": ["production", "^production"],
      "outputs": ["{options.outputPath}"],
      "cache": true
    },
    "test": {
      "dependsOn": ["build"],
      "inputs": ["default", "^production"],
      "cache": true
    },
    "lint": {
      "inputs": [
        "default",
        "{workspaceRoot}/eslint.config.js",
        "{workspaceRoot}/.prettierrc"
      ],
      "cache": true
    },
    "serve": {
      "dependsOn": ["^build"],
      "cache": false
    },
    "e2e": {
      "dependsOn": ["serve"],
      "cache": true
    }
  },
  "namedInputs": {
    "default": ["{projectRoot}/**/*", "sharedGlobals"],
    "production": [
      "default",
      "!{projectRoot}/**/*.spec.ts",
      "!{projectRoot}/tsconfig.spec.json",
      "!{projectRoot}/vitest.config.ts"
    ],
    "sharedGlobals": [
      "{workspaceRoot}/tsconfig.base.json",
      "{workspaceRoot}/nx.json"
    ]
  }
}
```

The `^` prefix in `dependsOn` means "the same target in dependency projects." When `build` depends on `^build`, building `financial-app` first builds every library it imports. Without the caret, `dependsOn: ["build"]` means "the build target in the *same* project" -- useful for tasks like `test` that should run after the project's own build completes.

### Parallel Execution

By default, Nx runs tasks in parallel up to a limit determined by available CPU cores. You can tune this explicitly:

```bash
nx run-many -t build test lint --parallel=8
nx affected -t test --parallel=5
```

Setting `--parallel` too high on memory-intensive tasks (like TypeScript compilation) causes thrashing. Setting it too low leaves cores idle. A good starting point is the number of physical cores minus one. For dedicated CI machines, raise the limit aggressively. Nx respects `dependsOn` ordering even under high parallelism -- if `shared/ui` depends on `shared/models`, the model library builds first regardless of the `--parallel` setting. Independent libraries build concurrently.

---

## Nx Release and Versioning

When shared libraries in the monorepo are consumed by external applications -- published to a private npm registry or used across independently deployed micro frontends from [Chapter 30](ch38-micro-frontends.md) -- versioning becomes necessary. `nx release` automates version bumping, changelog generation, and publishing.

### Configuration

Add release configuration to `nx.json`:

```json
{
  "release": {
    "projects": ["libs/shared/*"],
    "changelog": {
      "workspaceChangelog": {
        "file": "CHANGELOG.md",
        "createRelease": "github"
      },
      "projectChangelogs": {
        "file": "CHANGELOG.md",
        "renderOptions": {
          "authors": true,
          "commitReferences": true
        }
      }
    },
    "version": {
      "conventionalCommits": true,
      "generatorOptions": {
        "fallbackCurrentVersionResolver": "disk"
      }
    },
    "git": {
      "commit": true,
      "tag": true
    }
  }
}
```

The `projects` array scopes releases to the shared libraries -- application projects are deployed, not versioned as packages. The `conventionalCommits: true` option derives version bumps from commit message prefixes: `feat:` triggers a minor bump, `fix:` triggers a patch, and `BREAKING CHANGE:` in the commit body triggers a major bump.

### The Release Workflow

Preview version changes and changelog entries with a dry run before committing to anything:

```bash
nx release --dry-run
nx release            # when satisfied
nx release publish    # publish to registry
```

The `nx release` command analyzes commits since the last release tag, determines the version bump for each affected library based on conventional commit prefixes, updates `package.json` versions, generates `CHANGELOG.md` files (both workspace-level and per-project), and creates a git commit and tag. The `publish` step runs `npm publish` for each versioned library using the `dist/` build output. The combination of conventional commits and automated changelogs means version numbers carry semantic meaning, and stakeholders can read the changelog to understand what shipped without inspecting individual commits.

---

## Keeping the Monorepo Up-to-Date

A monorepo that pins Angular 19 while the rest of the ecosystem moves to 21 is not a monorepo -- it is a time capsule. Staying current with `@angular/*`, `@nx/*`, and third-party dependencies is ongoing work, but Nx provides tooling that makes it systematic rather than heroic.

### The Migration Workflow

Nx's migration system operates in two distinct phases, giving you a review step between analysis and execution:

```bash
nx migrate latest
```

This command does not change your source code. It updates `package.json` with the target versions and generates a `migrations.json` file listing every migration script that needs to run -- Angular schematics, Nx plugin migrations, and third-party codemods. Read through `migrations.json` before proceeding. Some migrations are purely mechanical (updating import paths); others may change behavior (switching from `NgModule` to standalone components).

When ready, apply the migrations:

```bash
npm install
nx migrate --run-migrations
```

Each migration script runs in sequence, modifying source files as needed. After the migrations complete, the critical step is verification:

```bash
nx run-many -t build test lint
nx run-many -t e2e
```

If everything passes, commit the changes as a single atomic update. If a migration breaks something, the `migrations.json` file lets you identify exactly which migration caused the issue, revert it, and apply the remaining migrations individually.

### Handling Breaking Changes

Major version upgrades across `@angular/*`, `@nx/*`, and third-party libraries often land simultaneously. An Angular major version bump typically requires a matching `@nx/angular` plugin update, which may require an Nx core update, which may update the TypeScript version requirement. Letting `nx migrate latest` resolve the full chain avoids version mismatches that produce cryptic build errors.

When Angular ships migration schematics -- like `onpush_zoneless_migration` for adopting zoneless change detection or `modernize` for converting legacy patterns to current APIs -- they are available both through the Angular CLI MCP tools described in [Chapter 49](ch47-ai-in-angular.md) and through `nx migrate`. The MCP approach is interactive: an AI assistant runs the migration, reviews the diff, and fixes edge cases. The `nx migrate` approach is batch-oriented: it applies the migration across the entire workspace in one pass. Use the MCP tools for exploratory, project-by-project migration; use `nx migrate` for workspace-wide upgrades.

### Automating Upgrade PRs

The manual workflow -- run migrate, install, run migrations, test, commit -- works for quarterly upgrades. For teams that want to stay continuously current, a scheduled CI workflow automates the process:

```yaml
# .github/workflows/nx-migrate.yml
name: Nx Migrate

on:
  schedule:
    - cron: '0 6 * * 1'  # Every Monday at 6:00 UTC
  workflow_dispatch:

permissions:
  contents: write
  pull-requests: write

jobs:
  migrate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: 'npm'

      - run: npm ci

      - name: Run nx migrate
        run: |
          npx nx migrate latest --no-interactive
          npm install --legacy-peer-deps

      - name: Check for migrations
        id: check
        run: |
          if [ -f migrations.json ]; then
            echo "has_migrations=true" >> "$GITHUB_OUTPUT"
          else
            echo "has_migrations=false" >> "$GITHUB_OUTPUT"
          fi

      - name: Run migrations
        if: steps.check.outputs.has_migrations == 'true'
        run: npx nx migrate --run-migrations

      - name: Clean up migrations.json
        if: steps.check.outputs.has_migrations == 'true'
        run: rm migrations.json

      - name: Verify workspace
        run: |
          npx nx run-many -t build test lint --parallel=5
          npx nx report

      - name: Create pull request
        if: steps.check.outputs.has_migrations == 'true'
        uses: peter-evans/create-pull-request@v7
        with:
          branch: chore/nx-migrate
          title: 'chore: update Nx and Angular dependencies'
          body: |
            Automated dependency update via `nx migrate latest`.

            ## Verification
            - [x] `nx run-many -t build test lint` passed
            - [ ] Manual smoke test of critical flows

            Review the changed files for any behavioral migrations
            that require manual verification.
          commit-message: 'chore: update Nx and Angular dependencies'
          labels: dependencies
```

The workflow runs every Monday, creates a pull request with the migration changes, and verifies that build, test, and lint pass before opening the PR. The team reviews the diff, runs any manual smoke tests, and merges. Failed migrations block the PR creation, so broken upgrades never reach the main branch unreviewed. The `workflow_dispatch` trigger allows manual runs when a critical security patch needs to be picked up immediately.

---

## Summary

[Chapter 14](ch14-monorepos-libraries.md) established the monorepo and taught Nx to cache and orchestrate. This chapter extended that foundation into the areas where mature workspaces spend most of their time: generating code, integrating AI tooling, optimizing pipelines, managing releases, and staying current.

Key takeaways:

- **Built-in generators** scaffold structurally correct Angular code. **Custom generators** scaffold *your* Angular code -- with the right design system wrappers, test patterns, and barrel exports baked in. Put them in `tools/generators/` and run them with `nx generate`.
- The **Nx MCP server** gives AI assistants workspace-level intelligence: project graphs, task monitoring, generator execution, and CI pipeline analytics. It complements the Angular CLI MCP from [Chapter 49](ch47-ai-in-angular.md) by operating at the workspace layer rather than the framework layer.
- **Task pipelines** in `nx.json` express dependency relationships between targets. The `^` prefix means "dependency projects first." Get the `dependsOn` chains right, and Nx parallelizes everything else automatically.
- **`nx release`** automates versioning, changelog generation, and publishing for shared libraries. Conventional commits drive version bumps, so the version number always reflects the nature of the change.
- **`nx migrate`** separates analysis from execution, giving you a review step before applying changes. Automate the workflow with a scheduled CI job that opens migration PRs weekly, keeping the workspace current without heroic manual effort.

The monorepo is not just a code organization strategy. With the right tooling, it becomes a force multiplier -- generating conventions-compliant code, giving AI assistants structured access to the workspace, and keeping every dependency current automatically.

> **Companion code:** Custom generators live in `financial-app/tools/generators/`. Nx MCP configuration is in `.cursor/mcp.json` alongside the Angular CLI MCP from [Chapter 49](ch47-ai-in-angular.md).
