# Chapter 14: Monorepos & Reusable Libraries

In [Chapter 8](ch08-architecture.md) we drew boundaries inside the FinancialApp -- vertical slices, barrel exports, Sheriff rules. Those boundaries lived within a single project. As the application matures, some of that code wants to escape. The shared UI components could serve a second application. The domain models belong in a package that backend-for-frontend services also consume.

This chapter covers the two primary strategies for managing multiple projects and reusable libraries in the Angular ecosystem: the Angular CLI's built-in workspace support and Nx. We will work through both using the FinancialApp, which is already structured as an Nx monorepo in the companion code (`financial-app/`).

## Angular CLI-based Repos

### Creating a Monorepo

The Angular CLI supports multi-project workspaces out of the box. A standard `ng new` creates a single application, but you can add libraries alongside it:

```bash
ng new financial-workspace
cd financial-workspace
ng generate library shared-models
ng generate library shared-ui
ng generate library shared-data-access
```

Each `ng generate library` command creates a directory under `projects/` with its own `package.json`, `tsconfig`, and `ng-package.json` (the ng-packagr configuration). The CLI also updates `angular.json` to register each library as a project and adds TypeScript path mappings to the root `tsconfig.json`:

```json
{
  "compilerOptions": {
    "paths": {
      "shared-models": ["projects/shared-models/src/public-api.ts"],
      "shared-ui": ["projects/shared-ui/src/public-api.ts"],
      "shared-data-access": ["projects/shared-data-access/src/public-api.ts"]
    }
  }
}
```

These path mappings mean your application code imports from libraries as though they were installed npm packages -- no relative paths reaching into sibling directories.

### Structure of Libraries

An Angular library project has a specific anatomy that differs from an application. The key file is `ng-package.json`, which tells ng-packagr how to compile and package the library:

```json
{
  "$schema": "../../node_modules/ng-packagr/ng-package.schema.json",
  "dest": "../../dist/shared-models",
  "lib": {
    "entryFile": "src/public-api.ts"
  }
}
```

The `public-api.ts` file (the entry point) is the library's barrel export. It defines the public surface area -- everything a consumer is allowed to import:

```typescript
// projects/shared-models/src/public-api.ts
export { Account, AccountStatus } from './lib/account.model';
export { Transaction, TransactionType } from './lib/transaction.model';
export { Portfolio, Holding } from './lib/portfolio.model';
export { Client, ClientRef } from './lib/client.model';
```

Anything not exported through `public-api.ts` is internal to the library. This mirrors the information hiding pattern from [Chapter 8](ch08-architecture.md), but now with build-level enforcement -- consumers literally cannot import unexported symbols when the library is built and distributed as a package.

Libraries can also define **secondary entry points** for tree-shakable sub-packages. Each sub-directory with its own `ng-package.json` becomes an importable path (e.g., `shared-ui/data-table`), and the bundler tree-shakes everything the consumer does not reference.

### Trying Out the Library in the Monorepo

During development, the TypeScript path mappings let you consume library code without building it first. The dev server compiles library source alongside the application. A feature component imports directly from the shared library:

```typescript
import { Account } from '@financial-app/shared/models';
import { AccountCardComponent } from '@financial-app/shared/ui';

@Component({
  selector: 'app-account-overview',
  imports: [AccountCardComponent],
  template: `
    @for (account of accounts(); track account.id) {
      <lib-account-card [account]="account" />
    }
  `,
})
export class AccountOverviewComponent {
  accounts = input.required<Account[]>();
}
```

No build step, no linking -- just TypeScript path resolution. You edit a shared model, save, and see the change reflected in the running application immediately.

### Building and Publishing the npm Package

When you are ready to publish a library, ng-packagr compiles it into a format compatible with the Angular Package Format (APF):

```bash
ng build shared-models
```

The output lands in `dist/shared-models/` and includes ESM bundles, TypeScript declaration files, and a `package.json` with proper `exports` fields. To publish, use a scoped name in the library's `package.json`:

```json
{
  "name": "@financial-app/shared-models",
  "version": "1.0.0",
  "peerDependencies": {
    "@angular/core": ">=21.0.0"
  }
}
```

Declaring Angular as a `peerDependency` rather than a direct dependency prevents version duplication in the consumer's bundle. Then publish from the dist output:

```bash
cd dist/shared-models
npm publish --registry https://registry.your-org.com
```

### Consuming the npm Package

Once published, any Angular application installs and imports the library like any other npm package:

```bash
npm install @financial-app/shared-models
```

```typescript
import { Account, AccountStatus } from '@financial-app/shared-models';
```

The key decision is **when to publish versus when to keep code in the monorepo**. Libraries shared across independently deployed applications need to be published. Libraries used only within the monorepo are better consumed through path mappings -- publishing adds versioning overhead without benefit.

## Faster Builds and More Convenience with Nx

The Angular CLI's monorepo support works, but it does not scale gracefully. Every `ng build` and `ng test` operates on one project at a time with no awareness of what has changed. Nx extends the CLI with a project graph, computation caching, and task orchestration. The companion FinancialApp is already structured as an Nx workspace.

### Creating an Nx Workspace

You can create a new Nx workspace with Angular in a single command:

```bash
npx create-nx-workspace@latest financial-app \
  --preset=angular-monorepo \
  --appName=financial-app \
  --style=scss \
  --e2eTestRunner=none
```

Or add Nx to an existing Angular CLI workspace:

```bash
npx nx@latest init
```

The `init` command adds `nx.json` and installs the Nx CLI. Existing `angular.json` projects continue to work unchanged. The FinancialApp's workspace separates applications and libraries into dedicated directories:

```
financial-app/
  apps/
    financial-app/            <-- the main application
      project.json
      src/
        app/
          features/
            accounts/
            transactions/
            portfolios/
            clients/
  libs/
    shared/
      models/                 <-- @financial-app/shared/models
        project.json
        src/
          account.model.ts
          transaction.model.ts
          portfolio.model.ts
          client.model.ts
          index.ts
      ui/                     <-- @financial-app/shared/ui
        project.json
        src/
          account-card/
          transaction-card/
          index.ts
      data-access/            <-- @financial-app/shared/data-access
        project.json
        src/
          account.service.ts
          transaction.service.ts
          portfolio.service.ts
          index.ts
  nx.json
  tsconfig.base.json
  sheriff.config.ts
```

Each project has its own `project.json` instead of being registered in a monolithic `angular.json`, so teams can modify project configuration without merge conflicts. The path mappings in `tsconfig.base.json` wire the `@financial-app/shared/*` imports to their source locations.

### Module Boundaries

Nx provides built-in module boundary enforcement through project tags and ESLint rules. For applications using the vertical slice architecture from [Chapter 8](ch08-architecture.md), **Sheriff** offers a more ergonomic alternative -- it works at the file and directory level, enforcing boundaries both within and across Nx libraries.

The FinancialApp's `sheriff.config.ts` defines a tagging scheme based on directory structure:

```typescript
import { noDependencies, SheriffConfig } from '@softarc/sheriff-core';

export const sheriffConfig: SheriffConfig = {
  version: 1,
  tagging: {
    'apps/<app>': 'app:<app>',
    'libs/shared/<lib>': 'shared:<lib>',
  },
  depRules: {
    'app:*': ['shared:*'],
    'shared:models': noDependencies,
    'shared:ui': ['shared:models'],
    'shared:data-access': ['shared:models'],
  },
};
```

This configuration enforces three rules:

1. Applications can depend on any shared library.
2. The `shared/models` library has no dependencies -- it is the leaf of the dependency tree.
3. Both `shared/ui` and `shared/data-access` can depend on `shared/models`, but not on each other.

If a developer in `shared/ui` tries to import from `shared/data-access`, Sheriff flags it in the editor immediately.

### Nx with Sheriff and Detective

Nx and Sheriff complement each other: Nx manages the build graph and task orchestration; Sheriff enforces the architectural rules. The dependency graph is both **optimized** (Nx only rebuilds what changed) and **correct** (Sheriff prevents illegal dependencies from forming). To integrate Sheriff into an Nx workspace:

```bash
npm install -D @softarc/sheriff-core @softarc/eslint-plugin-sheriff
```

Then configure your `eslint.config.js`:

```javascript
import sheriff from '@softarc/eslint-plugin-sheriff';

export default [
  // ... other configs
  {
    plugins: { '@softarc/sheriff': sheriff },
    rules: {
      '@softarc/sheriff/dependency-rule': 'error',
      '@softarc/sheriff/deep-import': 'error',
    },
  },
];
```

The `deep-import` rule prevents consumers from reaching past a library's barrel export. The `dependency-rule` enforces the relationships defined in `sheriff.config.ts`. For visualizing the dependency graph, **Detective** renders an interactive diagram from the same Sheriff configuration -- use it to verify that reality matches intent.

### Incremental Builds with Nx

The single most impactful feature Nx brings to a monorepo is **computation caching**. When you run a target -- build, test, lint -- Nx hashes the inputs (source files, dependencies, configuration) and caches the output. If the inputs have not changed, Nx replays the cached result instead of re-executing the task.

```bash
# First run: builds everything
npx nx build financial-app
# ... 14s

# Second run: nothing changed, replayed from cache
npx nx build financial-app
# ... 42ms (cache hit)
```

The `nx.json` configuration defines what counts as an input for caching:

```json
{
  "targetDefaults": {
    "build": {
      "dependsOn": ["^build"],
      "inputs": ["production", "^production"],
      "cache": true
    },
    "test": {
      "inputs": ["default", "^production"],
      "cache": true
    },
    "lint": {
      "inputs": ["default", "{workspaceRoot}/eslint.config.js"],
      "cache": true
    }
  },
  "namedInputs": {
    "default": ["{projectRoot}/**/*", "sharedGlobals"],
    "production": [
      "default",
      "!{projectRoot}/**/*.spec.ts",
      "!{projectRoot}/tsconfig.spec.json"
    ]
  }
}
```

The `dependsOn: ["^build"]` entry means building the application first builds its library dependencies. Since each library's build is independently cached, Nx only rebuilds what actually changed. Modify a component in `shared/ui`, and Nx rebuilds `shared/ui` and `financial-app` but skips `shared/models` and `shared/data-access`.

The `affected` command narrows this further by analyzing the git diff:

```bash
npx nx affected -t build
npx nx affected -t test
```

On a feature branch that only touches transaction files, `nx affected -t test` skips `shared/models` and `shared/ui` entirely. In a workspace with dozens of libraries, this reduces CI time from minutes to seconds.

### Distributed Cache with Nx Cloud

Local caching accelerates individual builds, but the same task is often executed redundantly across the team. Developer A runs the tests, pushes, and CI runs them again. Developer B pulls and runs them a third time. Nx Cloud solves this with a **remote cache** -- when any team member or CI agent completes a task, the result is stored centrally and replayed by subsequent runs.

```bash
npx nx connect
```

This links your workspace to Nx Cloud. No other code changes are required -- the Nx CLI transparently checks the remote cache before executing any task. In a monorepo with 30 libraries, most CI runs complete in under a minute because the majority of libraries have not changed since the last successful build.

### Even Faster: Parallelization with Nx Cloud

Caching eliminates redundant work. Parallelization speeds up the work that remains. Nx Cloud's **Distributed Task Execution (DTE)** splits tasks across multiple CI agents automatically.

Without DTE, a typical CI pipeline looks like this:

```yaml
# .github/workflows/ci.yml (sequential)
steps:
  - run: npx nx affected -t lint
  - run: npx nx affected -t test
  - run: npx nx affected -t build
```

Each step runs on a single agent. If linting takes 2 minutes, testing takes 4 minutes, and building takes 3 minutes, the pipeline takes 9 minutes total.

With DTE, Nx Cloud distributes tasks across a pool of agents:

```yaml
# .github/workflows/ci.yml (distributed)
jobs:
  main:
    steps:
      - uses: nrwl/ci@v4
        with:
          distribute-on: '5 linux-medium-js'
          commands: |
            npx nx affected -t lint test build
```

Nx Cloud analyzes the task graph, identifies which tasks can run in parallel, and distributes them across agents. It respects dependency order -- `shared/models` builds before `shared/ui` -- while running independent tasks concurrently. The 9-minute sequential pipeline completes in under 3 minutes.

## Summary

Monorepos and reusable libraries address the organizational challenge of sharing code across projects without the overhead of publishing and versioning every change.

The Angular CLI provides the foundation: `ng generate library` creates library projects, ng-packagr compiles them into the Angular Package Format, and TypeScript path mappings allow in-repo consumption during development. For small teams with a handful of shared libraries, this is sufficient.

Nx extends this foundation with project-graph awareness, computation caching, and CI/CD acceleration. The FinancialApp's workspace -- with its `shared/models`, `shared/ui`, and `shared/data-access` libraries -- demonstrates how Nx organizes code into independently buildable and testable units. Sheriff enforces the dependency rules that keep the architecture honest, and Detective visualizes those relationships for the whole team.

The practical decision tree is straightforward:

- **Single application, few shared models**: keep libraries in the CLI workspace with path mappings.
- **Multiple applications, shared component library**: adopt Nx for caching and the project graph.
- **Large organization, CI bottlenecks**: add Nx Cloud for distributed caching and task execution.

Each step builds on the previous one. You can start with CLI libraries and adopt Nx later without restructuring your code. The path mappings, barrel exports, and Sheriff rules you set up today carry forward unchanged.

In [Chapter 8](ch08-architecture.md) we designed the boundaries. In this chapter, we gave those boundaries physical form as libraries, packages, and workspace projects. The architecture is no longer just a convention -- it is encoded in the build system.
