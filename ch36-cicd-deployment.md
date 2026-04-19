# Chapter 34: CI/CD & Deployment

A feature is not shipped when the pull request is merged. It is shipped when the code is running on production infrastructure, serving real users, with a reliable way to roll it back if something goes wrong. Everything between "green tests locally" and "bytes in a user's browser" is the delivery pipeline -- and a well-tuned pipeline quietly compounds across a team's output. Fast feedback on every commit. A preview URL attached to every pull request. Staged rollouts that let you ship on Friday without flinching. The alternative -- a flaky, slow, manual deploy -- taxes every change the team makes, whether anyone measures it or not.

This chapter walks through the pipeline for FinancialApp end to end: the GitHub Actions workflow that runs Nx affected targets on every push, the Docker image that serves SSR in production, the Kubernetes manifests that schedule it, and the release strategy that makes production changes boring again. The companion code referenced throughout is already wired into the monorepo from [Chapter 14](ch14-monorepos-libraries.md) and [Chapter 37](ch15-advanced-nx.md) -- the pipeline here is the outer layer that those chapters were building toward.

> **Companion code:** `financial-app/.github/workflows/deploy.yml`, `financial-app/docker/Dockerfile`, `financial-app/docker/nginx.conf`, environment files in `financial-app/apps/financial-app/src/environments/`.

---

## GitHub Actions for Nx

CI for an Nx monorepo has one defining characteristic: most commits touch only a fraction of the workspace. `nx affected` computes the subset of projects impacted by a change and runs lint, test, build, and e2e only on those projects. A full workspace run on every commit would turn a thirty-second change into an eight-minute pipeline; affected runs keep feedback close to the diff that produced it. [Chapter 37](ch15-advanced-nx.md) covers `nx affected` in depth -- here we wire it into GitHub Actions.

The minimal `ci.yml` workflow has four responsibilities: check out the code with enough history to compute `affected`, install dependencies with a warm cache, run `nx affected` with parallelism, and upload the reports so a failed run is actually debuggable.

```yaml
# .github/workflows/ci.yml
name: CI

on:
  pull_request:
  push:
    branches: [main]

jobs:
  main:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - uses: actions/setup-node@v4
        with:
          node-version: '22'
          cache: 'npm'

      - name: Derive affected base SHA
        uses: nrwl/nx-set-shas@v4

      - name: Install dependencies
        run: npm ci

      - name: Lint, test, build affected
        run: npx nx affected --target=lint,test,build --parallel=3

      - name: E2E affected
        run: npx nx affected --target=e2e --parallel=1

      - name: Upload test reports
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: test-reports
          path: |
            coverage/
            apps/financial-app-e2e/playwright-report/
            apps/financial-app-e2e/test-results/
          retention-days: 14
```

Three details in this workflow do more work than they look like they do. `fetch-depth: 0` tells `actions/checkout` to fetch the full commit history, which `nx affected` needs to walk the graph from the merge base. `nrwl/nx-set-shas` exports the `NX_BASE` and `NX_HEAD` environment variables so affected knows which range to diff against -- on a PR it picks the branch point, on a push to `main` it picks the previous successful CI run. `--parallel=3` lets Nx run three projects' targets concurrently on a standard runner; bump it higher on larger machines, keep it at `1` for e2e to avoid fighting over the port the Playwright server binds to ([Chapter 19](ch37-e2e-playwright.md)).

The `if: always()` on the artifact upload matters: when a build fails, you want the Playwright trace and the coverage report *more* than when it passes. Without `always()`, a red build throws away the evidence of what went wrong.

---

## Caching: Making Commits Cheap

CI caching has two layers, and it is worth understanding which one solves which problem. The `cache: 'npm'` option on `setup-node` caches the npm registry download -- it turns `npm ci` from "fetch and unpack 300 MB" into "verify the lockfile hash and restore." This is installation caching.

**Computation caching** is the second, bigger win. When you run `nx build financial-app`, Nx hashes the inputs (source files, dependencies, build configuration) and checks whether it has seen that exact combination before. If it has, it replays the cached output instantly instead of rebuilding. [Chapter 14](ch14-monorepos-libraries.md) introduced this locally; on CI, it matters even more -- most PRs touch a single library, and every unchanged project should restore from cache in milliseconds.

The local cache lives in `.nx/cache`, which a CI runner discards between jobs. To share computation across runs and across developers, point Nx at a remote cache. **Nx Cloud** is the managed option -- `npx nx connect-to-nx-cloud` adds an `nxCloudAccessToken` to `nx.json` and registers the workspace. Once connected, every CI run (and every developer's machine) reads from and writes to the same cache bucket. A PR that changes a single component no longer rebuilds the other fourteen libraries -- it pulls them straight from the cache.

For teams that cannot send source hashes to a managed service, Nx also supports self-hosted remote caches backed by S3-compatible storage or a filesystem. The tradeoff is operational: you own the cache bucket, its eviction policy, and its credentials.

A fallback that works without any remote cache is `actions/cache` pointed at the Nx cache directory:

```yaml
- uses: actions/cache@v4
  with:
    path: .nx/cache
    key: nx-${{ runner.os }}-${{ github.sha }}
    restore-keys: |
      nx-${{ runner.os }}-
```

This is coarser than a real remote cache -- a `restore-keys` match can pull in stale entries -- but it costs nothing and buys back most of the wins on a sequential branch.

---

## Branch Strategy: Main, Tags, and PRs

A three-environment deployment model maps naturally to Git:

- **Pull requests** get ephemeral preview deploys -- a unique URL for every branch, torn down when the PR closes.
- **`main`** deploys to **staging** -- every merge immediately appears on the internal staging environment so the team dogfoods what is about to ship.
- **Semantic version tags** deploy to **production** -- cutting a release is creating a tag, and only tags trigger production deploys.

The workflow for this strategy lives in `.github/workflows/deploy.yml`, with separate jobs keyed off the event that triggered the run:

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches: [main]
    tags: ['v*']
  pull_request:
    types: [opened, synchronize, reopened, closed]

jobs:
  deploy-staging:
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '22', cache: 'npm' }
      - run: npm ci && npx nx build financial-app --configuration=staging
      - uses: docker/build-push-action@v5
        with:
          context: .
          file: docker/Dockerfile
          tags: ghcr.io/${{ github.repository }}:staging-${{ github.sha }}
          push: true
      - run: kubectl set image deployment/financial-app app=ghcr.io/${{ github.repository }}:staging-${{ github.sha }}
        env: { KUBECONFIG: ${{ secrets.KUBECONFIG_STAGING }} }

  deploy-production:
    if: github.event_name == 'push' && startsWith(github.ref, 'refs/tags/v')
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '22', cache: 'npm' }
      - run: npm ci && npx nx build financial-app --configuration=production
      - uses: docker/build-push-action@v5
        with:
          context: .
          file: docker/Dockerfile
          tags: |
            ghcr.io/${{ github.repository }}:${{ github.ref_name }}
            ghcr.io/${{ github.repository }}:latest
          push: true
      - run: kubectl set image deployment/financial-app app=ghcr.io/${{ github.repository }}:${{ github.ref_name }}
        env: { KUBECONFIG: ${{ secrets.KUBECONFIG_PRODUCTION }} }
```

GitHub Actions **environments** (the `environment:` keys) are worth calling out. An environment carries its own secrets -- a staging `KUBECONFIG` separate from production -- and can require manual approval before a job runs. Configuring production to require approval means a tag push queues the deploy but waits for a human to click "approve" -- a small gate that catches "oh no, I tagged the wrong commit" before it matters.

---

## Preview Deploys per PR

A preview deploy is a fully functional version of the branch, hosted at a unique URL. Reviewers click through the actual UI instead of reading a screenshot. Designers flag visual regressions without pulling down the branch. QA runs exploratory testing before merge.

Vercel, Netlify, and Cloudflare Pages all ship preview deploys as a first-class feature. Connect the repository once, and every PR gets a URL like `financial-app-pr-482.vercel.app` automatically. The provider also comments back on the PR with the URL, so reviewers do not need to dig for it.

For self-hosted infrastructure, the equivalent is a workflow step that builds the branch, pushes to a namespaced path in object storage (or a Kubernetes namespace), and posts the URL back as a PR comment using `actions/github-script`:

```yaml
- uses: actions/github-script@v7
  with:
    script: |
      github.rest.issues.createComment({
        issue_number: context.issue.number,
        owner: context.repo.owner,
        repo: context.repo.repo,
        body: `Preview: https://pr-${context.issue.number}.preview.financialapp.internal`
      });
```

The teardown step -- triggered when the PR is closed -- matters as much as the deploy. Without it, preview deploys accumulate forever, and the object storage bill or namespace quota eventually forces a manual cleanup sprint nobody wants.

---

## Deployment Targets: What Do We Actually Run On?

There is no universal "best" deployment target for an Angular app. The right answer depends on whether the app is a SPA or uses SSR ([Chapter 22](ch31-defer-ssr-hydration.md)), how much control you need, and what infrastructure your organization already runs.

**Vercel, Netlify, Firebase Hosting.** Serverless-style hosting with zero infrastructure to manage. Push a git branch; they run the build, serve the static assets from a CDN, and run any SSR code as serverless functions. Preview deploys, SSL, and global CDN come for free. The tradeoff is flexibility -- you configure what the platform exposes, not what the OS exposes. For FinancialApp's marketing pages or a SPA dashboard they are nearly frictionless; for heavy SSR workloads with long-lived connections they can become constraining.

**AWS Amplify Hosting.** Similar model, tighter integration with the rest of AWS. A good fit if you are already running your backend in AWS and want IAM-scoped deploys, CloudFront for the CDN, and CodeBuild for the pipeline. The developer experience is slightly rougher than Vercel's, but the AWS integration pays back when you need it.

**Docker + Kubernetes.** Full control. You build the image, you own the cluster, you decide how SSR scales. This is the right target when you need runtime control (custom middleware, streaming SSR, background jobs), when compliance requires on-prem hosting, or when the organization already runs Kubernetes for everything else. The operational cost is real -- someone owns the cluster -- but the ceiling is unlimited.

**Cloud Run.** Google Cloud's "serverless containers" offering -- a middle ground. Bring a Docker image, get autoscale-to-zero and pay-per-request billing without managing a cluster. For an SSR Angular app with spiky traffic it is often the sweet spot. AWS App Runner and Azure Container Apps occupy a similar niche on their respective clouds.

A common hybrid pattern runs the static SPA on a CDN-backed host and the SSR rendering on containers. Angular's hybrid routing from [Chapter 22](ch31-defer-ssr-hydration.md) -- prerendered routes as static HTML, server-rendered routes as SSR -- fits this split naturally.

---

## Docker for SSR Angular

When you do run in a container, the Dockerfile has one job: produce the smallest, most secure runtime image possible. That means a multi-stage build -- a fat build stage with Node and the full toolchain, and a slim runtime stage with only what the server needs to execute.

```dockerfile
# financial-app/docker/Dockerfile
FROM node:22-alpine AS build
WORKDIR /app

COPY package.json package-lock.json ./
RUN npm ci

COPY . .
RUN npx nx build financial-app --configuration=production && \
    npx nx build financial-app --configuration=production --ssr

FROM gcr.io/distroless/nodejs22-debian12 AS runtime
WORKDIR /app

COPY --from=build /app/dist/financial-app/server ./server
COPY --from=build /app/dist/financial-app/browser ./browser

ENV NODE_ENV=production
ENV PORT=4000
EXPOSE 4000

CMD ["server/server.mjs"]
```

Three decisions are worth calling out. The build stage uses the full `node:22-alpine` image because compilation needs `npm`, TypeScript, and the Angular CLI. The runtime stage uses Google's `distroless/nodejs22` image, which ships Node.js and its runtime dependencies but nothing else -- no shell, no package manager, no busybox. The attack surface is a fraction of an Alpine image, and the resulting container is usually under 150 MB.

The second decision is running *two* builds: the browser bundle and the SSR server bundle. Angular's `@angular/ssr` produces both from a single project but needs both artifacts copied into the runtime. The server bundle imports the browser bundle's HTML shell at runtime.

Third, there is no `CMD ["node", "server/server.mjs"]`. The distroless image sets `node` as its entrypoint, so the `CMD` only specifies the script. If you copy-paste from an Alpine example, the `node` prefix gives you a "node: command not found" error that is annoying to diagnose.

For a static SPA deployment -- no SSR, just the browser bundle -- swap the runtime stage for Nginx and a config file:

```nginx
# financial-app/docker/nginx.conf
server {
  listen 80;
  server_name _;
  root /usr/share/nginx/html;
  index index.html;

  location ~* \.(?:js|css|woff2?|png|jpg|svg|ico)$ {
    expires 1y;
    add_header Cache-Control "public, immutable";
    try_files $uri =404;
  }

  location / {
    try_files $uri $uri/ /index.html;
    add_header Cache-Control "no-cache";
  }

  add_header X-Content-Type-Options "nosniff" always;
  add_header X-Frame-Options "DENY" always;
  add_header Referrer-Policy "strict-origin-when-cross-origin" always;
}
```

The `try_files` fallback to `/index.html` is the critical line for any SPA: deep links like `/portfolio/holdings/42` have no matching file on disk, so Nginx must serve the shell and let the Angular router handle the path. The hashed asset files (with content hashes in their names) get a year-long `immutable` cache header; `index.html` gets `no-cache` so clients always check for a new version.

---

## Kubernetes: Deployment, Service, Ingress

A Kubernetes deploy for FinancialApp's SSR container is three YAML documents: a `Deployment` that runs the container, a `Service` that gives it a stable address inside the cluster, and an `Ingress` that exposes it to the outside world.

```yaml
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: financial-app
  labels: { app: financial-app }
spec:
  replicas: 3
  selector:
    matchLabels: { app: financial-app }
  template:
    metadata:
      labels: { app: financial-app }
    spec:
      containers:
        - name: app
          image: ghcr.io/acme/financial-app:latest
          ports:
            - containerPort: 4000
          envFrom:
            - configMapRef: { name: financial-app-config }
            - secretRef: { name: financial-app-secrets }
          resources:
            requests: { cpu: 100m, memory: 256Mi }
            limits:   { cpu: 500m, memory: 512Mi }
          readinessProbe:
            httpGet: { path: /health, port: 4000 }
            initialDelaySeconds: 5
          livenessProbe:
            httpGet: { path: /health, port: 4000 }
            initialDelaySeconds: 30
---
apiVersion: v1
kind: Service
metadata: { name: financial-app }
spec:
  selector: { app: financial-app }
  ports:
    - port: 80
      targetPort: 4000
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: financial-app
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  tls:
    - hosts: [app.financialapp.com]
      secretName: financial-app-tls
  rules:
    - host: app.financialapp.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: financial-app
                port: { number: 80 }
```

Resource requests and limits are not optional. Without `requests`, the scheduler cannot reason about where to place pods and you end up with "noisy neighbor" evictions. Without `limits`, a memory leak in one pod can starve everything else on the node. For SSR Angular, 256Mi is a sensible starting floor -- the Node.js runtime itself uses about 60 MB, and each concurrent SSR render adds incremental heap.

Environment configuration lives in a `ConfigMap` for non-secret values and a `Secret` for API keys and database credentials:

```yaml
# k8s/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata: { name: financial-app-config }
data:
  API_URL: "https://api.financialapp.com"
  ENVIRONMENT: "production"
  FEATURE_FLAG_ENDPOINT: "https://flags.financialapp.com"
  LOG_LEVEL: "info"
```

The `envFrom` block on the Deployment pulls every key in both the ConfigMap and the Secret into the container as environment variables. Changing a ConfigMap value does *not* automatically restart the pods -- you either roll the deployment (`kubectl rollout restart deployment/financial-app`) or use a tool like Reloader that watches ConfigMaps and triggers rollouts for you.

---

## Multi-Environment Strategy

FinancialApp has three environments (dev, staging, prod), and the code needs to know which one it is running in. Angular offers two mechanisms: build-time environments and runtime configuration. Both have their place.

**Build-time environments** are the traditional approach -- a file per environment, swapped at build time via `fileReplacements` in `angular.json`:

```typescript
// apps/financial-app/src/environments/environment.ts (dev)
export const environment = {
  production: false,
  apiUrl: 'http://localhost:3000',
};

// apps/financial-app/src/environments/environment.staging.ts
export const environment = {
  production: true,
  apiUrl: 'https://api-staging.financialapp.com',
};

// apps/financial-app/src/environments/environment.prod.ts
export const environment = {
  production: true,
  apiUrl: 'https://api.financialapp.com',
};
```

Build-time environments are the right tool when the differences between environments are fixed at build time (the dev build enables debug UI; the prod build does not). They are the *wrong* tool when the same built image needs to behave differently in staging and production -- which is the whole premise of container-based deployment, where you build once, promote everywhere.

**Runtime configuration** solves the "build once, deploy many" problem. The image ships with a config placeholder; the container populates it at startup from environment variables; the Angular app fetches it during bootstrap.

A small shell script in the container substitutes the environment variables into a static JSON file before Node starts:

```bash
#!/bin/sh
# docker/entrypoint.sh
cat > /app/browser/assets/config.json <<EOF
{
  "apiUrl": "${API_URL}",
  "environment": "${ENVIRONMENT}",
  "featureFlagEndpoint": "${FEATURE_FLAG_ENDPOINT}"
}
EOF
exec node server/server.mjs
```

An `APP_INITIALIZER` (or the modern `provideAppInitializer` from [Chapter 48](ch13-initialization-routes.md)) fetches this file before bootstrap completes, storing the result in a `RuntimeConfigService` that the rest of the app injects. The same container image can now be promoted from staging to production by changing the ConfigMap -- no rebuild, no risk of "did the wrong bundle ship?"

**Secrets are different.** Never commit them to environment files, never bake them into images, never echo them to the build log. Three patterns cover most teams:

1. **GitHub Secrets** -- fine for CI credentials (Docker registry tokens, `KUBECONFIG`). They are scoped to a repository or environment, masked in logs, and injected as `${{ secrets.* }}` in workflows.
2. **Kubernetes Secrets** -- for runtime credentials (database passwords, signing keys). Created with `kubectl create secret` or the External Secrets Operator; injected into pods via `envFrom` or mounted volumes.
3. **HashiCorp Vault or AWS Secrets Manager** -- for the whole organization. Applications fetch secrets via a sidecar or an SDK; rotation is automated; access is audited. This is the right answer at any real scale.

The non-negotiable rule is that `.env.production` never appears in a Git commit. A `.env.example` with placeholder values is fine as documentation; the actual values belong in the secret manager.

---

## Canary Releases and Rollback

Every production deploy is a gamble that the new code works on real traffic. Canary releases shrink that gamble by sending a small percentage of traffic to the new version first and watching for regressions before shifting more.

The simplest canary mechanism is **traffic splitting at the ingress layer**. NGINX Ingress supports weighted backends:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: financial-app-canary
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "10"
spec:
  rules:
    - host: app.financialapp.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: financial-app-v2
                port: { number: 80 }
```

This routes 10% of traffic to the new `financial-app-v2` service while the other 90% continues to hit the stable version. You bump the weight from 10 to 25 to 50 to 100 over hours or days, watching error rates and latency at each step.

For teams already invested in Kubernetes, **Argo Rollouts** or **Flagger** automate this loop: they read metrics from Prometheus, bump the weight automatically if the canary stays healthy, and roll back if error rates spike. The rollout controller replaces the `Deployment` object with a `Rollout` that declares the progression explicitly:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata: { name: financial-app }
spec:
  strategy:
    canary:
      steps:
        - setWeight: 10
        - pause: { duration: 10m }
        - setWeight: 25
        - pause: { duration: 10m }
        - setWeight: 50
        - pause: { duration: 20m }
        - setWeight: 100
```

A different canary mechanism is **feature flags**. Instead of shipping a new version to a percentage of traffic, you ship the new code to 100% of users but gate it behind a flag that is enabled for a small cohort. The deployment is boring; the risky change happens at flag-toggle time, which you can reverse instantly without a redeploy. Feature flags and how to integrate them with Angular are covered in Chapter 38.

**Rollback** is the mirror image of deploy. In Kubernetes, `kubectl rollout undo deployment/financial-app` reverts to the previous ReplicaSet -- usually within seconds. In Vercel, the dashboard exposes a "Promote previous deployment" button that does the same thing at the CDN layer. The habit to build is making sure rollback is rehearsed, not just possible: if nobody on the team has rolled back this quarter, you do not actually know whether rollback works.

---

## GitLab CI: Same Workflow, Different Runner

Nothing in this chapter is GitHub-specific. GitLab CI expresses the same pipeline in `.gitlab-ci.yml`:

```yaml
stages: [install, verify, build, deploy]

variables:
  NX_SKIP_NX_CACHE: "false"

install:
  stage: install
  image: node:22-alpine
  cache:
    key: { files: [package-lock.json] }
    paths: [node_modules/]
  script:
    - npm ci

verify:
  stage: verify
  image: node:22-alpine
  needs: [install]
  script:
    - npx nx affected --target=lint,test,build --parallel=3 --base=origin/main
  artifacts:
    when: always
    paths: [coverage/, apps/financial-app-e2e/playwright-report/]
    expire_in: 14 days

deploy:staging:
  stage: deploy
  image: docker:24
  services: [docker:24-dind]
  only: [main]
  script:
    - docker build -f docker/Dockerfile -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA .
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
    - kubectl set image deployment/financial-app app=$CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
```

The shape is identical: install, verify affected, build an image, promote it. Whichever CI system your organization runs -- GitHub Actions, GitLab CI, CircleCI, Buildkite -- the pipeline logic translates with minor syntactic adjustments.

---

## Wiring In Observability

A deploy is an event, and that event is valuable information for the observability stack. Tag every production release in Sentry so the error tracker can correlate spikes with specific deployments:

```yaml
- name: Create Sentry release
  uses: getsentry/action-release@v1
  env:
    SENTRY_AUTH_TOKEN: ${{ secrets.SENTRY_AUTH_TOKEN }}
    SENTRY_ORG: acme
    SENTRY_PROJECT: financial-app
  with:
    environment: production
    version: ${{ github.ref_name }}
    sourcemaps: ./dist/financial-app/browser
```

Uploading source maps is the step that pays back every time. A production error stack trace shows minified function names (`a.b.c(d)`) without source maps and readable TypeScript with them. The release also shows the list of commits since the last deploy, so an error that first appears at 10:03 AM can be traced to the specific PR that introduced it.

Chapter 17 covers the full observability story -- structured logging, distributed tracing, and error budgets. The hook from CI is narrow: every deploy announces itself to the tools that watch production, so those tools can connect deploys to the behavior that follows.

---

## Summary

A reliable CI/CD pipeline is less about any single tool and more about stacking small, correct defaults:

- **`nx affected` on every commit**, not the full workspace -- feedback stays close to the diff and the pipeline scales sub-linearly as the monorepo grows ([Chapter 37](ch15-advanced-nx.md)).
- **Two layers of caching** -- `setup-node`'s install cache for `npm ci`, and Nx Cloud (or a self-hosted remote cache) for computation. Unchanged projects restore instantly; CI time tracks the size of the change, not the size of the repo.
- **A three-environment Git strategy** -- PRs get previews, `main` deploys to staging, tags deploy to production. Each environment is gated in GitHub Actions environments so secrets and manual approvals are colocated with the target they protect.
- **Deployment target matches the workload.** Serverless platforms for low-ops projects; Kubernetes when you need runtime control; often a hybrid where the static SPA lives on a CDN and SSR lives on containers.
- **A multi-stage Dockerfile with a distroless runtime.** Build with the full Node image, ship with distroless. Smaller, faster, and a fraction of the attack surface of a general-purpose base image.
- **Runtime config over build-time config.** Build once, promote the same artifact through every environment, and inject differences through ConfigMaps. Secrets live in a dedicated secret manager -- never in the image, never in a committed file.
- **Canary releases instead of binary switches.** Traffic-split rollouts with NGINX, Argo Rollouts, or Flagger; feature flags where "deploy and toggle" is safer than "deploy and hope." Rehearse rollback often enough that it is boring.
- **CI announces deploys to the observability stack** -- Sentry releases with source maps, deploy markers on dashboards, release tags on traces. When production misbehaves after 2 AM, the on-call engineer starts with the list of things that changed.

The pipeline here is the scaffolding on top of which every other quality investment in the book pays off. The Playwright suite from [Chapter 19](ch37-e2e-playwright.md) is only as valuable as how quickly its results reach the PR. The SSR architecture from [Chapter 22](ch31-defer-ssr-hydration.md) is only as useful as the container that serves it in production. The Nx monorepo from [Chapter 14](ch14-monorepos-libraries.md) only stays fast with the caching wiring in this chapter. Deployment is the seam where code meets users -- keep that seam tight, boring, and well-instrumented, and the team ships faster with less anxiety.
