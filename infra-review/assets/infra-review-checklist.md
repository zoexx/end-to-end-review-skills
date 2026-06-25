# Infra review checklist

Copy-pasteable into a PR. Grouped by the five pillars, plus a dedicated **Rollback & blast radius** sub-list — the question to answer for every infra change. Skip rows that don't apply; flag what does with a severity label (🔴 blocking · 🟠 important · 🟡 nit · 🔵 suggestion · 📚 learning · 🌟 praise).

For infra, a 🔴 is something that can take prod down or expose data: an irreversible destructive change, a leaked credential, a deploy with no rollback.

## Rollback & blast radius (ask first, for every change)
- [ ] What environment does this touch — **prod or staging** — and what's the blast radius?
- [ ] What does it **provision or deploy** (one service? a database? an IAM role? the whole cluster?)
- [ ] What's the **rollback path**, and has it been tested? (Not "we'll roll forward.")
- [ ] Does the diff **destroy or replace** anything stateful (database, disk, bucket)? Was the plan read for it?
- [ ] Who can **trigger** this, and with what credentials? Are they least-privilege and per-environment?
- [ ] Is the irreversible step (drop/delete/destroy) **deferred** to its own later deploy?

## CI/CD pipelines
- [ ] No `pull_request_target` (or equivalent) checks out + runs **fork** code with secrets/write token in scope
- [ ] Third-party actions and reusable workflows pinned to a **SHA**, not a moving tag
- [ ] Job/build container images pinned to an exact version + **digest** — no `:latest`
- [ ] No secret echoed to logs, baked into an image `ARG`/layer, or printed by `-v`/`set -x`/`env`
- [ ] Cloud auth uses short-lived **OIDC**, not long-lived static keys in secrets
- [ ] `permissions:` set explicitly with a read-only floor; write granted per-job only where needed
- [ ] Production deploys require an **approval/environment gate**; required checks pass (fail-closed)
- [ ] Branch protection: required checks, no direct/force push to the release branch
- [ ] Caches aren't shared across trust levels; artifact integrity re-verified, not trusted from cache
- [ ] Deploys to one environment serialized with `concurrency` — can't overlap or roll backward
- [ ] Separate deploy credentials per environment

## IaC (Terraform / CDK / Pulumi)
- [ ] State uses a **remote backend** with **locking** and **encryption at rest** — no local/committed state
- [ ] No secrets in committed `*.tfvars`; secrets from a manager; state access-controlled because it holds them
- [ ] CI runs `plan` on the PR for review; `apply` is gated and applies the **saved plan**, not `-auto-approve` on push
- [ ] Providers, modules, and `required_version` **pinned**; `.terraform.lock.hcl` committed
- [ ] Plan read for **`# forces replacement`**; stateful resources have `prevent_destroy` + deletion protection
- [ ] `for_each` (keyed by identity) instead of `count` where element add/remove shouldn't recreate others
- [ ] IAM policies scope `Action` and `Resource` — no `"*"`/`"*"` admin grants
- [ ] Resources tagged (owner/env/cost-center), ideally via provider `default_tags`
- [ ] Dependencies expressed by direct resource references, not `data` lookups that race on a fresh apply
- [ ] Drift detected in CI (`plan -detailed-exitcode`); IaC is the single writer

## Containers & Kubernetes
- [ ] Dockerfile runs as a **non-root** `USER`; base image pinned to version + **digest** (no `:latest`)
- [ ] No secrets in `ARG`/`COPY`/layers; build secrets via BuildKit `--mount=type=secret`
- [ ] **Multi-stage** build; runtime stage minimal/distroless; `.dockerignore` excludes `.git`/`.env`/secrets
- [ ] `COPY` over `ADD`; a `HEALTHCHECK` (or pod probes); image scanning runs in CI and fails on high CVEs
- [ ] Both `resources.requests` and `resources.limits` set for cpu and memory
- [ ] Liveness + readiness (+ startup) probes; liveness has no downstream dependency
- [ ] `securityContext`: `runAsNonRoot`, `allowPrivilegeEscalation: false`, `readOnlyRootFilesystem`, `drop: ["ALL"]`
- [ ] No `privileged`/`hostPath`/`hostNetwork`/`hostPID` on app workloads (scoped + justified for node agents)
- [ ] Image referenced by **digest**; rolling-update strategy bounds unavailability; a `PodDisruptionBudget` exists
- [ ] Secrets via `secretKeyRef` from a real store — no plaintext in manifests; etcd encryption on
- [ ] ServiceAccount RBAC is a namespaced least-privilege `Role`, not `cluster-admin`; default-deny `NetworkPolicy`

## Deploy & release safety
- [ ] A **tested rollback path** exists — one action, not "roll forward"
- [ ] Risky/user-facing changes roll out **progressively** (canary/blue-green), not big-bang to 100%
- [ ] Migrations are **backward-compatible during rollout** (expand/contract); destructive `CONTRACT` is a later deploy
- [ ] Promotion gated on **golden-signal metrics**; a failed gate triggers **automatic rollback**
- [ ] App handles **SIGTERM**: fails readiness, drains in-flight work, then exits; grace period + `preStop` set
- [ ] Breaking changes additive/versioned or behind a **feature flag** rolled out gradually
- [ ] Risky new behavior has a **kill switch** that defaults safe and flips without a deploy
- [ ] Deploy scripts **idempotent** and `set -euo pipefail` — a partial failure can be re-run
- [ ] No single deploy fuses code + schema + infra into one **irreversible** step
- [ ] A **post-deploy smoke test** exercises critical paths and can trigger rollback

## Config, secrets & observability
- [ ] One immutable image; environment-specific **config injected** at runtime, not baked in
- [ ] Config and credentials are **per-environment**; non-prod never holds prod creds or points at prod data
- [ ] No secrets committed or in plaintext config; all secrets from a **manager**, only references in the repo
- [ ] The new component ships **alerts** on its golden signals, each with a `for:` and a runbook link
- [ ] A **dashboard** covers latency / traffic / errors / saturation for the new component
- [ ] Trace/correlation IDs **propagated** through the component; every log line carries the id
- [ ] An **owner, SLO, and runbook** exist; alerts route to the owning on-call rotation
- [ ] Autoscaling has sensible **min/max** bounds tied to downstream capacity and budget
- [ ] A **budget/cost alert** (and anomaly detection) guards runaway spend; resources tagged
- [ ] Component IAM/service-account role is **least-privilege**, bound via workload identity, not a static key

## Verdict
- [ ] Findings grouped by severity with `file:line` + WHY (outage / data loss / exposure / cost / no rollback) + concrete fix
- [ ] Explicit decision: ✅ approve · 💬 approve with comments · 🔁 request changes · ⛔ block
