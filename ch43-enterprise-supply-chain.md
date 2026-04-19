# Chapter 18: Enterprise Governance & Supply Chain

Every Angular application inherits a software supply chain it did not write. A freshly generated `ng new` project resolves hundreds of direct and transitive packages before a developer types a single line of code. For FinancialApp, production dependencies expand to four-digit numbers once TypeScript tooling, build plugins, and test harnesses are counted. Each of those packages was authored by someone your organization has no contract with, published to a registry you do not run, and pinned by a lockfile that a single typo in a PR description can rewrite. The application you ship to a regulated end user is, mathematically, mostly other people's code.

[Chapter 35](ch18-security-owasp.md) treats the supply chain as one of ten OWASP categories -- a few paragraphs on `npm audit`, lockfile integrity, and install scripts. That coverage is correct, but for an enterprise financial product it is the beginning of the conversation, not the end. Regulators now expect a Software Bill of Materials for every production release. Procurement offices demand a VPAT and SOC 2 evidence before signing the contract. A single leaked API key in a compiled `environment.ts` can trigger a mandatory breach disclosure under SEC cybersecurity rules. These obligations are organizational, not purely technical, and they require the engineering team to treat governance artifacts -- SBOMs, signed releases, dependency review workflows, license reports -- as shipping deliverables with the same rigor as features.

This chapter maps the governance surface for an enterprise Angular project: what artifacts to produce, which tools produce them, where they live after generation, and how CI enforces the policies that keep them honest. Most of the tooling is generic npm and GitHub infrastructure rather than Angular-specific, but the integration points -- what goes into the bundle, what leaks through `environment.ts`, what an auditor can verify -- are frontend engineering concerns.

> **Prerequisites:** [Chapter 35](ch18-security-owasp.md) (application-level OWASP), [Chapter 34](ch36-cicd-deployment.md) (CI/CD fundamentals), [Chapter 16](ch42-compliance.md) (regulatory context).

> **Companion code:** `financial-app/.github/workflows/supply-chain.yml` (added in Phase 4), alongside the existing `deploy.yml` from [Chapter 34](ch36-cicd-deployment.md) and the compliance libraries from [Chapter 16](ch42-compliance.md).

---

## Why Supply Chain Matters for Financial Apps

The modern supply-chain threat model was set by SolarWinds in 2020 -- a trusted build system, signed updates, and a nation-state adversary who turned a monitoring tool into a backdoor inside thousands of customer networks. npm has had its own sequence of incidents since: `event-stream` injecting wallet-stealing code through a transitive dependency, `ua-parser-js` shipping a cryptominer in a hijacked release, `node-ipc` sabotaging users in certain regions via a postinstall payload.

The Node ecosystem's breadth -- hundreds of thousands of packages, most with a single maintainer -- is what makes Angular development productive and what makes the blast radius of any compromise enormous. A single compromised dependency at the leaf of the tree executes inside the same Node process as the build tooling, which usually has filesystem access to `~/.aws`, `~/.npmrc`, and the CI runner's temporary token for pushing images. From that foothold, exfiltrating credentials or back-dooring the output bundle is a matter of minutes.

Regulators have caught up. The SEC's 2023 cybersecurity disclosure rule requires public US companies to report material cybersecurity incidents within four business days, and supply-chain compromises now qualify. PCI-DSS v4.0, which became mandatory on March 31, 2025, requires documented inventories of third-party software and active monitoring for known vulnerabilities -- in practice, an SBOM and a dependency-scanning process. The EU Cyber Resilience Act imposes similar obligations on any product with "digital elements" sold in the EU. For a financial application serving consumer accounts, these regimes stack rather than substitute.

Against this backdrop, `npm audit` is a floor, not a ceiling. It flags packages with advisories in the public database, ignores advisories that have not yet been published, and says nothing about license compliance, secret leakage, or artifact integrity. An Angular team that treats a green `npm audit` as sufficient diligence will fail an SOC 2 audit the first time someone asks "how do you know what's in your production bundle?"

---

## SBOM Generation (Software Bill of Materials)

A Software Bill of Materials is the ingredient list for your shipped artifact. It enumerates every package, version, license, and supplier that contributes to the build, in a format that machines and auditors can both consume. Two formats dominate: **SPDX** (a Linux Foundation standard widely used in government procurement) and **CycloneDX** (an OWASP project with richer metadata for vulnerability correlation). Both are JSON-serializable, and most mature tools emit either on request. For a financial application, CycloneDX is the pragmatic default -- it integrates cleanly with vulnerability databases, and the US federal executive order on software supply-chain security (EO 14028) explicitly recognizes both.

Two tools cover most needs. The official npm-aware generator is **`@cyclonedx/cyclonedx-npm`**, which walks the `package-lock.json`, resolves the actual installed tree, and emits a CycloneDX JSON document:

```bash
npx @cyclonedx/cyclonedx-npm --output-format json \
  --output-file sbom.cdx.json
```

The cross-ecosystem alternative is **`syft`**, an Anchore-authored scanner that produces SBOMs from filesystem trees, container images, or package manifests. For the FinancialApp Docker image built in [Chapter 34](ch36-cicd-deployment.md), `syft` is the better fit because it inspects the final runtime layer (including system packages that `cyclonedx-npm` does not see):

```bash
syft ghcr.io/acme/financial-app:v1.4.2 \
  -o cyclonedx-json=sbom.cdx.json
```

The choice of tool also affects what ends up in the file. `cyclonedx-npm` describes the JavaScript dependency graph with high fidelity -- every package, its version, its declared license, its integrity hash -- but sees nothing below the language runtime. `syft` against a container image sees the Node.js binary, the base image's `libc`, any system tools baked in, and ignores TypeScript source. Mature pipelines produce both: a JavaScript SBOM for dependency review and a container SBOM for image scanning.

Store the generated SBOM alongside the artifact it describes. For FinancialApp, each production release uploads the SBOM to the same S3 bucket that holds the Docker image and the sourcemaps, keyed by the release tag. The SBOM is then signed with **`cosign`** (part of the **sigstore** project) so an auditor can verify that the inventory they are reading is the one CI produced, not one a developer regenerated on a laptop weeks later.

A GitHub Actions step to generate and sign the CycloneDX JSON during the production release:

```yaml
- name: Generate CycloneDX SBOM
  run: |
    npx @cyclonedx/cyclonedx-npm \
      --output-format json \
      --output-file sbom.cdx.json \
      --omit dev

- name: Install cosign
  uses: sigstore/cosign-installer@v3

- name: Sign SBOM with cosign
  env:
    COSIGN_EXPERIMENTAL: '1'
  run: |
    cosign sign-blob --yes \
      --output-signature sbom.cdx.json.sig \
      --output-certificate sbom.cdx.json.crt \
      sbom.cdx.json

- name: Upload SBOM artifacts
  uses: actions/upload-artifact@v4
  with:
    name: sbom-${{ github.ref_name }}
    path: |
      sbom.cdx.json
      sbom.cdx.json.sig
      sbom.cdx.json.crt
    retention-days: 365
```

The `--omit dev` flag is not cosmetic. A production SBOM should describe what ships to users, not the build-time toolchain; mixing the two inflates the inventory and dilutes the vulnerability signal. A year of retention is the minimum for most audit regimes -- older SBOMs answer the "which customers received code that used this CVE-affected version?" question after a compromise is disclosed, and that question cannot be answered without historical evidence.

One operational subtlety: `cosign sign-blob` in keyless mode uses ephemeral keys provisioned through OIDC from the CI runner's identity. The signing certificate binds the signature to the GitHub Actions workflow identity -- `https://github.com/acme/financial-app/.github/workflows/release.yml@refs/tags/v1.4.2` -- which means a verifier can cryptographically prove that the SBOM was produced by the release workflow running on a signed tag, not by a developer's laptop. That provenance is what turns an SBOM from "a JSON file someone handed us" into "an attested artifact we can trust."

---

## Dependency Review and Automated Updates

Three tools contend for the dependency-update slot in an npm repository, each with a different philosophy.

**GitHub Dependabot** is built into GitHub and requires no external service. A single `.github/dependabot.yml` file configures which ecosystems to watch, how often to open PRs, and which versions to group. It is the shortest path to a functional system and the right default for most teams.

**Renovate** is the more configurable alternative. Open-source with a hosted app and a self-hosted CLI, it supports granular schedules (only security updates on weekends, major versions in monthly groups), automerge rules that distinguish trusted ecosystems from risky ones, and dashboards that summarize pending updates. Renovate's flexibility is the reason many enterprises adopt it once Dependabot's single-knob configuration becomes limiting.

For a richer policy language, a minimal Renovate equivalent of the same setup:

```json
// renovate.json
{
  "extends": ["config:recommended", ":semanticCommits"],
  "schedule": ["before 6am on monday"],
  "timezone": "America/New_York",
  "packageRules": [
    {
      "matchPackagePatterns": ["^@angular/"],
      "groupName": "angular",
      "matchUpdateTypes": ["minor", "patch"]
    },
    {
      "matchUpdateTypes": ["patch"],
      "matchCurrentVersion": "!/^0/",
      "automerge": true,
      "automergeType": "pr"
    },
    {
      "matchDepTypes": ["devDependencies"],
      "matchUpdateTypes": ["patch", "minor"],
      "automerge": true
    }
  ],
  "vulnerabilityAlerts": { "enabled": true, "labels": ["security"] }
}
```

Renovate's `packageRules` compose into fine-grained policy: patch updates to stable runtime dependencies auto-merge, dev-dependency minor updates auto-merge, and anything that matches a `vulnerabilityAlert` is tagged so the security team sees it first. The rule order matters -- later rules override earlier ones -- which is the footgun most Renovate configs hit the first time. A `renovate --dry-run` against the repo before merging the config catches these mistakes before they create a hundred-PR tidal wave.

**GitHub Dependency Review Action** is neither of those. It does not open update PRs; it blocks PRs that *introduce* new vulnerabilities or forbidden licenses. Running as a required status check, it diffs the base branch's `package-lock.json` against the PR's and fails if a new dependency introduces a known CVE above your threshold. This is the tool that moves dependency review from "chase vulnerabilities after they land" to "prevent them from landing":

```yaml
# .github/workflows/supply-chain.yml (excerpt)
name: Supply chain

on:
  pull_request:
    branches: [main]

jobs:
  dependency-review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/dependency-review-action@v4
        with:
          fail-on-severity: high
          deny-licenses: GPL-2.0, GPL-3.0, AGPL-3.0
          comment-summary-in-pr: on-failure
```

The policy trio above -- fail on high or critical CVEs, reject copyleft licenses, comment on failures only -- is a sensible starting point for a financial app. Tightening `fail-on-severity` to `moderate` is tempting but produces enough false-positive friction that teams start bypassing the check, which is worse than a slightly looser threshold honored consistently.

A minimal `dependabot.yml` that watches npm and GitHub Actions weekly, with security updates taking priority:

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: npm
    directory: /
    schedule: { interval: weekly, day: monday }
    open-pull-requests-limit: 5
    groups:
      angular:
        patterns: ['@angular/*', '@angular-*']
      types:
        patterns: ['@types/*']
    ignore:
      - dependency-name: '*'
        update-types: ['version-update:semver-major']

  - package-ecosystem: github-actions
    directory: /
    schedule: { interval: weekly }
```

The `groups` block is the quiet productivity win. Without it, every `@angular/*` package opens an independent PR and the review queue fragments into twenty near-identical diffs. With it, a single PR bumps the entire Angular release train, which is how the Angular team ships them and how the team should consume them. The `ignore` block defers major-version updates to a human; those updates invariably carry breaking changes that require code review no bot can perform.

For auto-merging, the principle is that **patch-level security updates** to a dependency with passing tests and a clean dependency-review result can merge themselves; **minor and major updates**, and anything that touches a transitive dependency across multiple packages, requires human review. Renovate expresses this with a `packageRules` entry that labels security updates `automerge: true` while leaving feature updates for weekly triage. Dependabot's equivalent is an `auto-merge` workflow keyed by `github.event.pull_request.user.login == 'dependabot[bot]'` and the update type reported in the PR body. Either way, the combination of required CI, branch protection, and a required code owner review is what keeps the loop safe -- the auto-merge only fires when every gate has already passed.

---

## Secret Scanning

A credential checked into Git is a credential the attacker already has. Public GitHub repositories are scraped within seconds of a push; private repositories leak via cloned mirrors, forked workflows, and build-cache archives. The defense is a pipeline of three stages: pre-commit detection on the developer's machine, server-side detection at push time, and continuous scanning of the repository history.

**GitHub Secret Scanning**, enabled by default on public repos and available in GitHub Advanced Security for private ones, recognizes token patterns from dozens of partners (AWS, Stripe, OpenAI, GitHub itself) and can notify the issuing service directly when a match appears. Matching providers will often revoke the leaked token automatically, which shortens the attacker's window to minutes.

**`truffleHog`** (Truffle Security) and **`gitleaks`** (Zricethezav) are the open-source companions. Both scan Git history and working trees for entropic strings and pattern matches. Both run as pre-commit hooks via `husky` and as CI jobs that gate every PR. `gitleaks` has a more mature rule set for common secret formats; `truffleHog` includes verifiers that attempt a low-privilege API call to confirm a match is a real credential before alerting.

A minimal `gitleaks.toml` tuned for a financial frontend:

```toml
title = "FinancialApp gitleaks policy"

[allowlist]
paths = [
  '''(?i)test/.*\.spec\.ts''',
  '''mocks/.*\.json''',
]

[[rules]]
id = "stripe-live-key"
description = "Stripe live secret key"
regex = '''sk_live_[0-9a-zA-Z]{24,}'''

[[rules]]
id = "openai-key"
description = "OpenAI API key"
regex = '''sk-[A-Za-z0-9]{32,}'''

[[rules]]
id = "aws-access-key"
description = "AWS access key"
regex = '''AKIA[0-9A-Z]{16}'''
```

The `[allowlist]` block is not optional. Without it, Playwright fixtures and mocked API responses trip the scanner constantly, teams learn to ignore the noise, and a real finding arrives buried under a hundred false positives. Treat the allowlist as policy -- review it in PRs, keep it narrow, and prefer deterministic test doubles over strings that look like credentials.

A minimal pre-commit integration with `gitleaks`:

```json
// .lintstagedrc.json
{
  "*": ["gitleaks protect --staged --redact --verbose"]
}
```

Paired with a CI job running the full-history scan on every push:

```yaml
- name: gitleaks full-history scan
  uses: gitleaks/gitleaks-action@v2
  env:
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
    GITLEAKS_LICENSE: ${{ secrets.GITLEAKS_LICENSE }}
```

The most common response when a secret leaks is the wrong one: force-push a history rewrite and hope nobody noticed. This reliably fails. By the time the developer sees the alert, the secret has been pulled by CI caches, mirrored to downstream forks, and potentially indexed by third-party search tools. **Rotate the secret first, always**, then rewrite history if you must -- but assume the rewrite is cosmetic and that any credential exposed to Git is permanently compromised.

The rotation checklist for FinancialApp is documented and drilled:

1. Invalidate the leaked credential in the issuing system (AWS IAM, Stripe dashboard, Vault token, whatever issued it).
2. Rotate the live runtime credential via the secret manager described in [Chapter 34](ch36-cicd-deployment.md) so the new value propagates to pods without a code change.
3. Audit access logs for the exposed credential's window of validity -- the `correlationId` pattern from [Chapter 17](ch20-observability.md) makes it possible to ask "what calls were made with this token in the last four hours?"
4. File an incident ticket with the timeline, even if nothing downstream was compromised. SOC 2 auditors look for the record of response, not only for the absence of incidents.

Angular-specific leak hazards concentrate in three places. Values committed to `environment.ts` or `environment.prod.ts` are compiled into the production JavaScript bundle and shipped to every user's browser -- anything in them is public by construction, as [Chapter 35](ch18-security-owasp.md) covers under A02. Firebase `firebaseConfig` objects look sensitive but are safe to expose (the API key is a rate-limit identifier, not a secret); however, service-account JSON files *are* secrets and must never be bundled. AI-integration keys for runtime LLM calls -- the Hashbrown-adjacent patterns from [Chapter 49](ch47-ai-in-angular.md) -- belong exclusively behind a backend proxy, never in the Angular bundle, because a leaked OpenAI or Anthropic key bills the organization until it is rotated.

---

## License Compliance

Every npm package ships under a license, and the licenses are not interchangeable. Permissive licenses -- MIT, Apache-2.0, BSD-2-Clause, ISC -- let you ship the package in a proprietary product without significant obligations. Copyleft licenses -- GPL, AGPL, LGPL in some configurations -- impose share-alike requirements that are incompatible with a closed-source SaaS distribution. SSPL, the license MongoDB moved to, is treated by most legal teams as copyleft for these purposes. A single AGPL package in a transitive dependency can, under an aggressive reading, obligate the entire frontend bundle to be released under AGPL. Legal teams read this aggressively because courts often do.

Two tools produce the license inventory. **`license-checker`** (`npm install -g license-checker-rseidelsohn` for the maintained fork) walks `node_modules` and emits a JSON or CSV report of every package's declared license:

```bash
license-checker-rseidelsohn --production \
  --json --out licenses.json
```

The `--production` flag restricts the scan to runtime dependencies, which is what actually ships. Devtools, test runners, and build plugins have different legal exposure and are typically tracked in a separate report.

The SBOM tools described earlier include license data alongside package identity, so if CycloneDX is already in the pipeline, the license report is a filter on the SBOM rather than a separate invocation. A small script that reads `sbom.cdx.json` and fails the build if any component's license appears on a blocklist replaces a second tool:

```yaml
- name: Enforce license blocklist
  run: |
    node scripts/check-licenses.mjs sbom.cdx.json \
      --deny GPL-2.0,GPL-3.0,AGPL-3.0,SSPL-1.0
```

The allow/deny lists grow into a table that legal and engineering share:

| License | Status | Notes |
|---|---|---|
| MIT, Apache-2.0, BSD-2, BSD-3, ISC | Allow | Permissive; no shipping obligations |
| MPL-2.0, LGPL-2.1 (dynamic link) | Allow with review | File-level copyleft, acceptable if unmodified |
| GPL-2.0, GPL-3.0, AGPL-3.0 | Deny | Copyleft; incompatible with closed distribution |
| SSPL-1.0, BUSL-1.1 | Deny | Source-available but not OSI; often restricted |
| Unlicensed, UNKNOWN | Block build | License metadata missing -- research required |

A real example from a FinancialApp pre-production audit: a chart library added in a feature branch pulled `canvas-ascii-art` as a transitive dev dependency. The parent was MIT; the transitive ancestor was AGPL-3.0, added three releases earlier when a maintainer rewrote their build tooling. The dependency-review action caught it at PR time because the license diff triggered the blocklist, and the team swapped to a different chart library before the release branched.

The story ends well only because the gate existed. Without it, the AGPL artifact would have shipped, a customer's open-source compliance review would have flagged it months later, and the fix would have been a coordinated release across every deployment. The second-order lesson is that license compliance compounds with dependency review and SBOM generation into a single continuous regime -- the same pipeline that generates the CycloneDX JSON every release can feed the same blocklist, so license policy and vulnerability policy are enforced by one system, not two.

---

## Signed Commits and Artifacts

A signed commit proves that the author's Git identity matches a cryptographic key under their control. Without signatures, a `git log` entry is trust-by-convention: the `Author:` line is whatever the committer typed into `git config user.email`, and nothing prevents an attacker with push access from attributing their commits to your tech lead. Enterprise repositories should require signed commits on protected branches; GitHub surfaces the signature status per commit and allows the `require-signed-commits` branch protection rule.

Developer signing has historically used GPG -- developers generate a key, register its public half with GitHub, and configure Git to sign with it. The sigstore project's **`gitsign`** is the modern alternative: commits are signed with short-lived keys obtained via OIDC from the developer's identity provider, eliminating the GPG key-management overhead. For teams already using SSO for everything else, `gitsign` is the lower-friction path to the same cryptographic property.

Branch protection then ties the signature to the policy. A `main` branch with required signatures, required reviews, and required status checks cannot be pushed to directly; every change arrives via a PR whose author identity is cryptographically attested:

```json
// Excerpt of branch-protection rule for `main`
{
  "required_signatures": { "enabled": true },
  "required_pull_request_reviews": {
    "required_approving_review_count": 1,
    "require_code_owner_reviews": true
  },
  "required_status_checks": {
    "strict": true,
    "contexts": [
      "ci / lint-test-build",
      "supply-chain / dependency-review",
      "supply-chain / image-scan"
    ]
  },
  "enforce_admins": true
}
```

The `enforce_admins: true` setting is the one teams most often skip and most often regret. Without it, repository administrators can bypass every other control with a click, and the audit story becomes "we had controls, except when we didn't." With it, even the CTO merges through the same PR flow as everyone else.

Artifact signing is the analogous discipline for container images. **`cosign`**, another sigstore component, signs Docker images produced by CI and stores the signature alongside the image in the registry. The Dockerfile from [Chapter 34](ch36-cicd-deployment.md) produces `ghcr.io/acme/financial-app:v1.4.2`; a follow-on step signs it and records the signature to a transparency log:

```yaml
- name: Sign image with cosign
  env:
    COSIGN_EXPERIMENTAL: '1'
  run: |
    cosign sign --yes \
      ghcr.io/acme/financial-app@${{ steps.build.outputs.digest }}

- name: Attach SBOM as an attestation
  env:
    COSIGN_EXPERIMENTAL: '1'
  run: |
    cosign attest --yes \
      --predicate sbom.cdx.json \
      --type cyclonedx \
      ghcr.io/acme/financial-app@${{ steps.build.outputs.digest }}
```

The second step turns the SBOM into a **cosign attestation** -- the SBOM is still in the release bucket, but a signed copy also lives next to the image in the registry, reachable by `cosign download attestation`. A downstream consumer can now answer "what's in this image?" by asking the registry directly rather than trusting a side channel.

Kubernetes admission controllers (Sigstore's policy-controller, Kyverno, or OPA Gatekeeper) can then reject any image without a valid signature from an approved CI identity. For a financial regulator asking "how do you prove the binary running in production was produced by your pipeline and not tampered with in the registry?" the answer is the cosign signature, the attached SBOM attestation, and the Rekor transparency log entry that records both.

---

## Governance Tooling in the FinancialApp

FinancialApp's supply-chain controls live in three files, referenced throughout this chapter but worth seeing as a whole.

**`.github/workflows/supply-chain.yml`** runs on every pull request and on each push to `main`. It wires together dependency review, SBOM generation, and vulnerability scanning of the final Docker image with **Trivy** (Aqua Security's open-source scanner):

```yaml
# financial-app/.github/workflows/supply-chain.yml
name: Supply chain

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

jobs:
  dependency-review:
    if: github.event_name == 'pull_request'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/dependency-review-action@v4
        with:
          fail-on-severity: high
          deny-licenses: GPL-2.0, GPL-3.0, AGPL-3.0, SSPL-1.0

  sbom:
    if: github.event_name == 'push'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '22', cache: 'npm' }
      - run: npm ci --ignore-scripts
      - name: Generate CycloneDX SBOM
        run: npx @cyclonedx/cyclonedx-npm --output-format json \
               --output-file sbom.cdx.json
      - uses: actions/upload-artifact@v4
        with: { name: sbom, path: sbom.cdx.json, retention-days: 365 }

  image-scan:
    if: github.event_name == 'push'
    needs: sbom
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: aquasecurity/trivy-action@master
        with:
          scan-type: fs
          scan-ref: .
          severity: HIGH,CRITICAL
          exit-code: '1'
          ignore-unfixed: true
```

The three jobs split by trigger for a reason: dependency review is PR-time (block bad code from merging), SBOM generation is post-merge (archive the inventory for the code that actually landed), and image scanning runs against the filesystem before the Docker image is even built so the same scanner catches both direct and transitive issues. A production release workflow then adds `cosign`-based image signing as described earlier.

Trivy deserves a brief note of its own. Alternatives exist -- **Snyk** (commercial, richer advisory database), **Grype** (Anchore, open-source, shares a vulnerability feed with `syft`), **Docker Scout** (Docker's built-in scanner) -- and teams often end up with two: one for PR-time fast feedback and another with broader coverage for nightly deep scans. The combination matters more than the specific tools. What any of them buys you is a CI-enforced answer to "do any of our dependencies have a published CVE today?" that does not require a human to remember to ask.

**`sheriff.config.ts`** enforces the Nx module boundaries defined in [Chapter 31](ch16-style-guide-governance.md) -- a compliance library cannot import from a feature library, a feature cannot import from another feature's internals, and the application shell can only depend on public entry points. These rules are about architectural integrity but they matter for supply-chain governance too: a lint rule that forbids importing from unauthorized paths reduces the surface where a rogue dependency could reach sensitive data, and a stable import graph is what makes code owners (required for signed reviews on critical files) tractable.

**`eslint.config.js`** codifies the security and compliance lint rules from [Chapter 35](ch18-security-owasp.md) and [Chapter 16](ch42-compliance.md) alongside the style rules from [Chapter 31](ch16-style-guide-governance.md). The supply-chain-relevant rules include `no-restricted-imports` entries that forbid importing telemetry SDKs directly (every import must go through the gated wrapper), and custom selectors that flag `localStorage.setItem` calls against PII keys. None of these rules defend against a determined adversary; collectively they eliminate the category of accidental mistakes that survives code review when reviewers are tired.

Runtime observability closes the loop. The telemetry service from [Chapter 17](ch20-observability.md) tags every error with the release version and the commit SHA that produced it. When a CVE is disclosed against a version of a dependency, the security team queries the observability store for events from builds that shipped that version -- the correlation between SBOM inventory and runtime telemetry turns a disclosed vulnerability into a concrete list of affected users and sessions. Without the SBOM the query has no index; without telemetry it has no data. Together they reduce an incident-response conference call from hours of archeology to a single dashboard filter.

---

## VPAT and Compliance Documentation

A **Voluntary Product Accessibility Template** (VPAT) is the industry document that describes how a product conforms to accessibility standards -- WCAG 2.2, Section 508, EN 301 549. [Chapter 8](ch24-accessibility.md) covers VPAT production as an accessibility governance practice; this chapter revisits it from the procurement angle, because the VPAT is typically requested alongside the SBOM and SOC 2 report in enterprise sales and public-sector RFPs. Treating accessibility evidence as supply-chain evidence reflects how auditors actually look at the artifacts: a bundle of trust signals, any of which can block a deal.

The VPAT aligns with broader compliance frameworks. **SOC 2 Type II** attestations evaluate controls over a sustained window, and an accessibility control ("the team conducts manual screen-reader testing for every major release and documents findings in a VPAT") is as valid a SOC 2 evidence source as an access-review control. **ISO 27001** Annex A includes controls on secure development and supplier relationships that map directly onto the SBOM and dependency-review practices earlier in this chapter. A control-mapping spreadsheet that ties each artifact to the frameworks it satisfies turns compliance from a scramble-before-audit into a continuous discipline:

| Artifact | SOC 2 criterion | ISO 27001 clause | PCI-DSS v4.0 | Produced by |
|---|---|---|---|---|
| CycloneDX SBOM | CC7.1, CC8.1 | A.8.8, A.8.28 | 6.3.2, 12.5.1 | `supply-chain.yml` |
| License report | CC1.4 | A.5.19 | -- | SBOM post-processor |
| Signed image digest | CC6.8, CC7.2 | A.8.30, A.8.32 | 6.3.3 | `cosign sign` step |
| Dependency review log | CC7.1 | A.8.8 | 6.3.2 | Dependency Review Action |
| Secret-scan report | CC6.1 | A.5.17 | 3.7, 8.6 | gitleaks, Advanced Security |
| VPAT | CC1.4 | A.5.31 | -- | Manual a11y walkthrough ([Ch 8](ch24-accessibility.md)) |
| Penetration test report | CC7.1 | A.8.29 | 11.4 | External security firm |

The mapping is specific to the control language each regime uses, and two different auditors may want slightly different phrasings; the table still matters because it gives the team one place to look when "we need to show an SOC 2 control for supply chain" is asked with forty minutes of warning.

The concept that holds the system together is the **evidence repository** -- a single place (a Confluence space, a Notion database, a versioned folder in a governance repo) that captures audit artifacts across releases. For FinancialApp the structure is:

```
governance/
  releases/
    v1.4.2/
      sbom.cdx.json
      sbom.cdx.json.sig
      license-report.json
      vpat-2026-q1.pdf
      pen-test-findings.pdf
      release-notes.md
  policies/
    dependency-review.md
    secret-rotation.md
    incident-response.md
  control-map.xlsx
```

Every production release writes a new `releases/v*/` directory via a CI job. The VPAT and pen-test PDFs are attached manually (they are produced on a different cadence than code releases), but the generated artifacts -- SBOM, license report, signed image digests -- are automated. When the SOC 2 auditor asks "show me the SBOM for the version that was live on March 14," the answer is a folder path, not a scavenger hunt across chat threads.

The evidence repository also disciplines release notes. A `release-notes.md` that sits alongside the SBOM has to be written in a tone that survives external scrutiny -- "bumped 23 dependencies" is not acceptable; "upgraded `@angular/core` 19.2.1 -> 19.2.3 (patch), mitigating CVE-2025-XXXX in a transitive dependency of the build toolchain" is. Over time, the notes accumulate into a product history an auditor can walk, which is itself a control: regulators treat "we can show every change we made to the production artifact over the past two years, with timestamps and signatures" as strong evidence of mature change management.

---

## Summary

Supply-chain governance is the organizational layer above the application-level security covered in [Chapter 35](ch18-security-owasp.md): the artifacts, workflows, and evidence that prove to an external party that the Angular code you shipped is the Angular code you intended to ship.

- **SBOMs are mandatory infrastructure**, not optional documentation. Generate CycloneDX JSON from `@cyclonedx/cyclonedx-npm` or `syft`, sign with `cosign`, archive per release. PCI-DSS v4, SEC cybersecurity rules, and most enterprise procurement regimes now require the artifact.
- **Dependency review blocks vulnerabilities at PR time** via `actions/dependency-review-action`; Dependabot and Renovate then drive the ongoing update flow. Auto-merge security patches, require human review for minor and major bumps.
- **Secret scanning is multi-layered.** Pre-commit hooks with `gitleaks` or `truffleHog`, GitHub Advanced Security on the server, rotate-first-rewrite-second when a leak is confirmed. Angular-specific hazards live in `environment.ts`, service-account JSONs, and AI provider keys (see [Chapter 49](ch47-ai-in-angular.md)).
- **License compliance is a blocklist, not a question**. Fail CI on GPL, AGPL, and SSPL in the production tree; run `license-checker` or derive the report from the SBOM.
- **Signed commits and signed artifacts** prove authorship and integrity. `gitsign` for developer identity, `cosign` for container images, admission controllers to enforce that only signed images run in production.
- **The FinancialApp governance stack** -- `.github/workflows/supply-chain.yml`, `sheriff.config.ts` from [Chapter 31](ch16-style-guide-governance.md), `eslint.config.js` -- enforces the policies continuously so review pressure does not erode them. Runtime observability from [Chapter 17](ch20-observability.md) connects inventory to impact when a CVE is disclosed.
- **The VPAT** ([Chapter 8](ch24-accessibility.md)) is the accessibility evidence that ships alongside the SBOM in enterprise procurement. Together with SOC 2 and ISO 27001 control maps, these artifacts live in a versioned evidence repository so an auditor's request is answered by a folder path, not a fire drill.

None of this replaces the application-level protections of [Chapter 35](ch18-security-owasp.md), the CI/CD plumbing of [Chapter 34](ch36-cicd-deployment.md), or the regulatory controls of [Chapter 16](ch42-compliance.md). Governance tooling is the connective tissue between them -- the discipline that turns a secure application, a reliable pipeline, and a compliant product into a story you can tell a regulator with evidence rather than assertions.
