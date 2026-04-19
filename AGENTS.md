# FinancialApp -- Agent Conventions

<!-- See ch16-style-guide-governance.md -- the canonical human version. -->

Machine-readable architecture summary for AI assistants and automation. If any rule here conflicts with the chapter, the chapter wins -- but open an issue so this file can be updated.

## Repo Layout

```
modern-angular-book/
├── chNN-*.md                  # 56 book chapters (ch00..ch56)
├── docs/chapter-coverage.md   # latest audit report (generated)
└── financial-app/             # Nx v21 workspace, the running example
    ├── apps/                  # financial-app, -e2e, -widget, -mobile
    ├── backend/               # FastAPI contract backend (Appendix B0)
    ├── libs/shared/*          # cross-cutting libraries
    ├── tools/                 # generators + audit-chapter-coverage.mjs
    ├── sheriff.config.ts      # module-boundary rules
    └── eslint.config.js       # flat ESLint config
```

Each `libs/shared/<name>` directory is an Nx library with a public `src/index.ts` barrel. Deep imports past the barrel are a lint error.

## Sheriff Tag Categories

`@softarc/sheriff-core` enforces boundaries via `sheriff.config.ts`. Every file carries one or more of these tags:

1. **`domain:<slice>`** -- vertical-slice ownership: `domain:accounts`, `domain:transactions`, `domain:portfolios`, `domain:clients`. Defined in `libs/shared/architecture-notes/src/index.ts`.
2. **`type:api`** -- the public `index.ts` barrel of a feature slice. Other slices may import only from `type:api`.
3. **`type:internal`** -- implementation detail of a single slice. Importable only from within the same `domain:*` tag.
4. **`type:shared`** -- cross-cutting utilities under `libs/shared/*`. May be imported anywhere but may not depend on any `domain:*` or `type:internal` code.

A violation of any of these rules is a lint error, not a review comment.

## ESLint Rule Set

`financial-app/eslint.config.js` (flat config) composes:

- `@nx/eslint-plugin` -- `flat/base`, `flat/typescript`, `flat/javascript`.
- `@angular-eslint/eslint-plugin` -- `directive-selector` (prefix `app`, camelCase), `component-selector` (prefix `app`, kebab-case), `no-empty-lifecycle-method`, `prefer-standalone`.
- `@angular-eslint/eslint-plugin-template` -- `no-negated-async`, `eqeqeq` (allowNullOrUndefined), `banana-in-box`.
- `@softarc/sheriff-core`'s ESLint plugin enforces the tag rules above. Library-level configs may relax the prefix to `fin` for `shared/ui`.

## Chapter-Traceability Convention

Every hand-written source file in `financial-app/` must carry a chapter tag in the top 25 lines:

- TypeScript / JavaScript: `// See ch17-auth-patterns.md`
- HTML templates: `<!-- See ch11-signal-queries.md -->`
- SCSS / CSS: `/* See ch22-material-design-system.md */`
- Python / YAML / `.mjs`: `# See ch52-appendix-b0-backend-fastapi.md`

The tag points at the chapter in the repo root whose prose explains the file. Generated artifacts (under any `generated/` folder) are exempt.

## Running the Audit

From `financial-app/`:

```bash
npm run audit:chapters            # writes tools/chapter-coverage.json + docs/chapter-coverage.md
node tools/audit-chapter-coverage.mjs  # same thing, direct invocation
```

The audit fails CI (`exit 1`) when any of:

- a chapter references a file path that does not exist on disk (`missing`);
- a source file tags a chapter filename that no longer exists (`orphan_tags`).

Untagged source files are reported but do not fail the build -- fix them when you touch the file.

## Commands Before Proposing a Change

```bash
cd financial-app
npm run lint
npm test
npm run audit:chapters
```

All three must pass before a PR is review-ready.
