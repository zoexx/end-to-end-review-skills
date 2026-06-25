# CI/CD Pipelines

Deep-dive for reviewing the pipeline that builds, tests, and ships your code. A CI/CD system is a privileged execution environment with access to your secrets, your registry, and your production credentials — it is one of the highest-value targets in your whole stack. A flaw here doesn't break one request; it lets an attacker exfiltrate every secret you have, or push a poisoned artifact straight to prod. Each section pairs a ❌ anti-pattern with a ✅ fix and names the incident it causes.

Examples are GitHub Actions-first with GitLab CI / Jenkins notes, but the threats are platform-agnostic. (Secret-handling depth → **security-review**; this guide is about pipeline shape and trust boundaries.)

---

## 1. The fork pwn-request — `pull_request_target` running untrusted code

The single most dangerous CI misconfiguration. `pull_request_target` runs in the context of the **base** repo — with **secrets and a write token** — but if you check out the PR's head, you're executing attacker-controlled code with that access.

❌ Untrusted fork code with secrets in scope:
```yaml
on: pull_request_target          # ❌ runs with base-repo secrets + write token
jobs:
  test:
    steps:
      - uses: actions/checkout@v4
        with:
          ref: ${{ github.event.pull_request.head.sha }}   # ❌ checks out the fork's code
      - run: npm install && npm test    # ❌ fork's package.json scripts run with secrets
```
✅ Build untrusted PRs with no secrets; gate privileged steps:
```yaml
on: pull_request                 # ✅ fork PRs get NO secrets, read-only token
jobs:
  test:
    permissions: { contents: read }
    steps:
      - uses: actions/checkout@v4   # checks out fork code, but nothing secret is reachable
      - run: npm ci && npm test
# privileged follow-up (e.g. preview deploy) runs in a separate workflow_run / approved environment
```
> Incident: a stranger opens a PR from a fork. Your `pull_request_target` workflow checks out their branch and runs `npm install`, whose `postinstall` script reads `${{ secrets.* }}` from the environment and POSTs every key — cloud creds, npm token, signing key — to their server. You now have to rotate every secret and assume prod was breached. The fix is to never run untrusted code in a context that can read secrets.

If you must use `pull_request_target` (e.g. to label PRs), do **not** check out the head, and keep the job to trusted code only.

---

## 2. Unpinned actions / third-party steps — pin to a SHA

A tag like `@v4` or `@main` is a moving pointer. Whoever controls the upstream action (or its tag) controls code that runs in your pipeline with your secrets.

❌ Mutable refs:
```yaml
- uses: actions/checkout@v4          # ❌ tag can be re-pointed
- uses: some-org/deploy-action@main  # ❌ runs whatever main is today
```
✅ Pin to a full commit SHA:
```yaml
- uses: actions/checkout@b4ffde65f46336ab88eb53be808477a3936bae11   # v4.1.1
- uses: some-org/deploy-action@9f1e2d3c…  # pinned; renovate/dependabot bumps it deliberately
```
> Incident: the `tj-actions/changed-files` supply-chain compromise (2025) re-pointed a widely used tag to malicious code that dumped CI secrets into build logs for thousands of repos overnight. Anyone pinned to a SHA was unaffected; anyone on a floating tag ran the malware on their next build. Pin third-party actions to a SHA and let a bot bump them through review.

Same rule for the container images your jobs run in (see §4) and for reusable workflows (§9).

---

## 3. `:latest` and unpinned build images

A pipeline that pulls `:latest` (or an unpinned base/tool image) isn't reproducible — the build that passed yesterday can fail or ship different bytes today, and you can't reproduce an old release to debug it.

❌ Floating image:
```yaml
container: node:latest          # ❌ which Node? changes under you
# Dockerfile
FROM python:3                   # ❌ 3.x minor floats
```
✅ Pin to a digest or exact version:
```yaml
container: node:20.11.1-bookworm@sha256:abc123…   # exact + digest
# Dockerfile
FROM python:3.12.4-slim@sha256:def456…
```
> Incident: a release built green on Monday; on Wednesday the same pipeline fails because `node:latest` rolled to a new major with a breaking change — and you can't rebuild last week's passing artifact to bisect, because that tag no longer points at the same image. Pinning makes every build reproducible and every old release rebuildable.

---

## 4. Secrets echoed into logs or build output

Build logs are widely readable (often by anyone with repo read, sometimes public). A secret printed once is a secret leaked forever.

❌ Secret reaches the log / build output:
```yaml
- run: echo "Deploying with key ${{ secrets.API_KEY }}"   # ❌ printed to the log
- run: curl -H "Authorization: ${{ secrets.TOKEN }}" ... -v  # ❌ -v echoes the header
- run: env                                                  # ❌ dumps all secrets in env
# Dockerfile
ARG NPM_TOKEN
RUN echo $NPM_TOKEN                                         # ❌ and bakes it into a layer
```
✅ Keep secrets out of stdout and out of image layers:
```yaml
- run: deploy.sh                       # secret read from env inside the script, never echoed
  env: { API_KEY: ${{ secrets.API_KEY }} }
# Dockerfile — use a build secret mount, not ARG (ARG values persist in history)
RUN --mount=type=secret,id=npm_token  NPM_TOKEN=$(cat /run/secrets/npm_token) npm ci
```
> Incident: a `set -x` script or a `curl -v` prints the bearer token into a build log that a contractor (or, on a public repo, the world) can read. The token has to be rotated and you assume it was used. GitHub masks values registered as secrets in logs, but only exact matches — base64'd, sliced, or derived values slip through. Never deliberately print secret-derived data.

---

## 5. Long-lived cloud keys instead of OIDC

Static cloud access keys stored in CI secrets are a standing liability: they don't expire, they're copyable, and one leak (see §1, §4) is a permanent breach until someone notices and rotates.

❌ Static, long-lived credentials:
```yaml
- uses: aws-actions/configure-aws-credentials@v4
  with:
    aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}       # ❌ never expires, copyable
    aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
```
✅ Short-lived credentials via OIDC federation:
```yaml
permissions:
  id-token: write          # ✅ lets the job mint an OIDC token
  contents: read
steps:
  - uses: aws-actions/configure-aws-credentials@v4
    with:
      role-to-assume: arn:aws:iam::123:role/ci-deploy   # role trusts this repo's OIDC subject
      aws-region: us-east-1                               # creds last minutes, scoped to this run
```
> Incident: a leaked long-lived `AWS_SECRET_ACCESS_KEY` (from a log, a fork, or a compromised laptop) gives an attacker your account until a human notices and rotates — which can be weeks. OIDC issues credentials that live for minutes and are bound to a specific repo/branch/environment, so a leak is near-worthless. Federate CI to the cloud; reserve static keys for systems that genuinely can't do OIDC.

GitLab (`id_tokens`), Azure, and GCP all support the same workload-identity-federation pattern.

---

## 6. Excessive token permissions — set `permissions:`

The default workflow token is often read/write to everything in the repo. A compromised step (or dependency) inherits all of it.

❌ Default-broad token:
```yaml
# no top-level permissions: → token can push commits, publish packages, edit issues, etc.
jobs:
  build: { steps: [ ... ] }      # ❌ inherits write-all
```
✅ Default-deny, grant the minimum per job:
```yaml
permissions: { contents: read }   # ✅ repo-wide floor
jobs:
  release:
    permissions:                   # widen only where needed
      contents: write              # to push a tag / release
      packages: write              # to publish
    steps: [ ... ]
```
> Incident: a build step pulls a compromised npm dependency; because the workflow token is write-all, the malware uses it to push a backdoor commit to `main` and publish a poisoned package — all on your authority. Scoping the token to `contents: read` would have contained the blast to a failed build. Set `permissions:` explicitly; treat write as opt-in per job.

---

## 7. Production deploy with no approval gate

If merging to `main` (or any push) deploys straight to prod with no human/automated gate, a bad merge, a compromised account, or a fat-fingered branch ships instantly and irreversibly.

❌ Auto-deploy on push, no gate:
```yaml
on: { push: { branches: [main] } }
jobs:
  deploy-prod:
    steps: [ run: ./deploy.sh prod ]    # ❌ any merge → prod, no approval, no required checks
```
✅ Gate prod behind a protected environment + required checks:
```yaml
jobs:
  deploy-prod:
    environment: production       # ✅ requires reviewers / wait timer / branch rule to proceed
    needs: [test, security-scan]  # required checks must pass first (fail-closed)
    steps: [ run: ./deploy.sh prod ]
```
> Incident: a contributor merges a PR that wasn't meant for release; with no environment gate it deploys to prod in seconds, and the only "rollback" is another panicked deploy. A required `environment: production` with reviewers (and branch protection so only `main` can deploy) turns "instant accident" into "needs one approving click." Pair with branch protection: required status checks, no force-push, no direct push to `main`.

Make gates **fail-closed**: if the scan/test job errors, the deploy must not proceed (don't `continue-on-error` a security gate).

---

## 8. Cache poisoning

Build caches keyed on, or populated by, untrusted input let an attacker plant malicious artifacts that later jobs (including trusted, secret-bearing ones) restore and execute.

❌ Cache writable/restorable across the trust boundary:
```yaml
# a pull_request job (untrusted fork) writes the same cache key the deploy job reads
- uses: actions/cache@v4
  with: { path: ./node_modules, key: deps-${{ hashFiles('package-lock.json') }} }
# fork edits package-lock.json → controls the key → poisons node_modules for later jobs
```
✅ Don't share caches across trust levels; key on trusted inputs:
```yaml
# untrusted PR builds: use a read-only restore or a separate key namespace, never write the prod key
- uses: actions/cache/restore@v4      # restore only; fork can't populate the trusted cache
  with: { path: ./node_modules, key: deps-${{ hashFiles('package-lock.json') }} }
# verify integrity (lockfile + npm ci) instead of trusting cached binaries
```
> Incident: a forked PR seeds the dependency cache with a tampered `node_modules`; the next trusted job restores that cache and runs the planted code with deploy credentials. Caches are an execution input — treat a cache writable by untrusted code as code writable by untrusted users. Separate cache scopes per trust level and re-verify integrity (`npm ci` from a checked-in lockfile) rather than trusting restored binaries.

---

## 9. Untrusted reusable / shared workflows

`uses: org/repo/.github/workflows/x.yml@ref` runs *their* code in *your* context, inheriting whatever secrets you pass. An unpinned or untrusted reusable workflow is the same supply-chain risk as an unpinned action.

❌ Floating reference to a shared workflow + blanket secrets:
```yaml
jobs:
  deploy:
    uses: some-org/ci/.github/workflows/deploy.yml@main   # ❌ moving ref
    secrets: inherit                                       # ❌ hands over ALL secrets
```
✅ Pin the ref and pass only the secrets that workflow needs:
```yaml
jobs:
  deploy:
    uses: some-org/ci/.github/workflows/deploy.yml@a1b2c3d   # pinned SHA
    secrets:
      DEPLOY_TOKEN: ${{ secrets.DEPLOY_TOKEN }}              # explicit, minimal
```
> Incident: `secrets: inherit` against a reusable workflow you don't control hands every secret to code that can change under you. When that upstream repo is compromised (or its `@main` re-pointed), the attacker gets your full secret set on your next run. Pin the ref and pass secrets explicitly so the blast radius is one secret, not all of them.

---

## 10. Overlapping deploys — missing `concurrency`

Two deploy runs racing to the same environment can interleave, leaving it in a half-old/half-new state, or an older run can finish *after* a newer one and roll prod backward.

❌ No concurrency control:
```yaml
on: { push: { branches: [main] } }
jobs:
  deploy: { steps: [ run: ./deploy.sh prod ] }   # ❌ two quick merges → two overlapping deploys
```
✅ Serialize per environment; cancel or queue stale runs:
```yaml
concurrency:
  group: deploy-prod          # ✅ one deploy to prod at a time
  cancel-in-progress: false   # let the running deploy finish; queue the next (don't cancel mid-apply)
jobs:
  deploy: { steps: [ run: ./deploy.sh prod ] }
```
> Incident: two PRs merge a minute apart; both deploy jobs run; the slower (older) one finishes last and overwrites prod with the *previous* artifact — a silent rollback nobody ordered. A `concurrency` group serializes deploys so this can't happen. Note: `cancel-in-progress: true` is right for CI builds, but for a deploy mid-`apply`, prefer queueing so you never kill a half-finished rollout.

---

## Quick scan checklist

- [ ] No `pull_request_target` (or equivalent) checks out and runs **fork** code with secrets/write token in scope.
- [ ] Third-party actions and reusable workflows are pinned to a **SHA**, not a moving tag.
- [ ] Job/build container images are pinned to an exact version + digest — no `:latest`.
- [ ] No secret is echoed to logs, baked into an image `ARG`/layer, or printed by `-v`/`set -x`/`env`.
- [ ] Cloud auth uses short-lived **OIDC**, not long-lived static access keys in secrets.
- [ ] `permissions:` is set explicitly with a read-only floor; write is granted per-job only where needed.
- [ ] Production deploys require an **approval/environment gate** and pass required checks (fail-closed).
- [ ] Branch protection: required checks, no direct push / force-push to the release branch.
- [ ] Caches aren't shared across trust levels; integrity is re-verified rather than trusting restored artifacts.
- [ ] Deploys to one environment are serialized with `concurrency` so they can't overlap or roll backward.
