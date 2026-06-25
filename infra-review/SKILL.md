---
name: infra-review
description: |
  Reviews infrastructure and DevOps changes — CI/CD pipelines, IaC (Terraform/CDK/Pulumi), containers and Kubernetes, deploy/release safety, and config/secrets/observability — for blast radius and operability, not just style. It catches the changes that take prod down or expose data: long-lived CI secrets and fork pwn-requests, unpinned actions/`:latest` images, destructive Terraform replaces on stateful resources, root containers with no limits, and deploys with no tested rollback path.
  Use when: reviewing CI/CD changes, reviewing a GitHub Actions/GitLab pipeline PR, Terraform/IaC review, reviewing a Dockerfile or Kubernetes/Helm chart, deploy/release safety review, reviewing infra config, reviewing autoscaling or alerting changes, "is this infra change safe to ship?"
allowed-tools:
  - Read
  - Grep
  - Glob
  - Bash
  - WebFetch
---

# infra-review

A senior-level review pass for infrastructure and delivery code. It reads the change for *blast radius* first — what it provisions, what it deploys, what it can take down — then works to the line level using the reference guides, favoring concrete failure modes (the outage, the data loss, the breach, the cost blowup, the missing rollback) over taste.

## When this fires

- A PR touches `.github/workflows/*.yml`, `.gitlab-ci.yml`, Jenkinsfile, CircleCI/Buildkite config, or any pipeline/release automation.
- A `*.tf`, `*.tfvars`, CDK/Pulumi stack, CloudFormation template, or any IaC module changes.
- A `Dockerfile`, `docker-compose.yml`, `.dockerignore`, or a container build changes.
- A Kubernetes manifest, Helm chart, Kustomize overlay, or operator CRD changes.
- A deploy script, release workflow, rollout/canary config, or migration-ordering step changes.
- Config, secret wiring, alerting/dashboards, autoscaling, or IAM policy changes ship.
- The `end-to-end-review` orchestrator routes infra/CI/deploy files here.

Out of scope (route elsewhere): SQL schema/index/migration *content* → **database-review** (this skill reviews migration *ordering vs deploy*); secret-handling depth, authn/z, OWASP → **security-review** (this skill flags exposure and cross-references); server-side handler logic → **backend-review**; React/CSS/bundle → **frontend-review**. This skill cross-references those rather than duplicating them.

## The review model

Four phases, run in order. Don't nitpick a YAML line before you understand the blast radius.

1. **Context** — What environment does this touch, and what's the blast radius? Is it prod or staging? What does it provision (a database? an IAM role? a whole VPC?) or deploy (one service? the cluster?)? Who can trigger it, and with what credentials? Read the PR description, the pipeline triggers, the Terraform plan, the target environment. A change is "safe" only relative to what it can break and how widely.
2. **High-level** — Is this the right *topology and failure story*? Does the pipeline shape fit (least-privilege creds, gated deploys, separate per-env paths)? Is the rollout strategy appropriate to the risk (big-bang vs canary)? **For every change, ask: what's the rollback path, and has it been tested?** A missing rollback story caught here saves an incident.
3. **Detailed** — Line by line, using the reference guides below. Pinned versions, token permissions, state locking, resource limits, probes, securityContext, health gates, secret wiring, alerts. This is where most findings land.
4. **Verdict** — Summarize, group findings by severity, and give one explicit decision.

### Severity labels

Every finding carries exactly one. For infra, a 🔴 is something that can **take prod down or expose data** — an irreversible destructive change, a leaked credential, a deploy with no rollback.

| Label | Meaning |
| --- | --- |
| 🔴 **blocking** | Must fix before merge — outage risk, data loss, secret exposure, no rollback path |
| 🟠 **important** | A real problem; should fix, not strictly a blocker |
| 🟡 **nit** | Minor style/polish, non-blocking |
| 🔵 **suggestion** | Optional improvement or alternative |
| 📚 **learning** | Context/teaching, no action required |
| 🌟 **praise** | Genuinely good work, worth calling out |

### Verdict states

`✅ approve` · `💬 approve with comments` · `🔁 request changes` · `⛔ block`

### Principles

- **Review the code, not the coder.** Comment on the change; prefer "this workflow runs on `pull_request_target` with checkout of the fork's head" over "you made a security hole."
- **Say why.** Every blocking/important finding names the concrete impact: the outage, the data loss, the security exposure, the cost blowup, or the missing rollback. "`force_replace` on the RDS instance → prod database destroyed and recreated empty" beats "be careful with replaces."
- **Least privilege, immutable, repeatable.** Prefer short-lived OIDC over long-lived keys; pinned, reproducible artifacts over moving tags; and a **tested rollback for everything** that changes prod.
- **Stay in scope.** Flag pre-existing problems separately; don't hold the PR hostage to unrelated infra cleanup.
- **Praise is signal.** Calling out a clean OIDC setup, a `prevent_destroy`, or a real canary gate teaches the pattern.

## What I check

The five pillars. Each links to a reference guide for the dense version.

### 1. CI/CD pipelines
- CI authenticates to cloud via short-lived **OIDC**, not long-lived static keys in secrets.
- Actions/images are pinned to a **SHA** (or at least an immutable tag) — no `@master`, no `:latest`.
- Token permissions are explicit and minimal (`permissions:` set, default-read); deploy creds differ per environment.
- No `pull_request_target` (or equivalent) that checks out and runs **fork** code with secrets in scope — the pwn-request.
- No secret echoed to logs, into PR comments, or interpolated where it lands in build output.
- Production deploys sit behind a **required approval/environment gate**; required status checks + branch protection are enforced.
- Builds are reproducible; caches can't be poisoned by untrusted input; reusable/3rd-party workflows are trusted and pinned.
- `concurrency` prevents overlapping deploys of the same environment.

### 2. IaC (Terraform / CDK / Pulumi)
- Remote state backend with **locking and encryption at rest** — no local state, no unlocked concurrent applies.
- No secrets in state files or plaintext `*.tfvars` committed to the repo.
- `plan` runs in CI and is reviewed; **no auto-`apply` on push** without a gate.
- Providers and modules are **version-pinned**; no floating `latest`/unconstrained ranges.
- Destructive changes are understood: a diff that **force-replaces** a stateful resource (RDS, disk, bucket) is called out; stateful resources carry `lifecycle { prevent_destroy = true }`.
- `for_each` over `count` where list order/identity matters (avoids cascade re-creation on insert/remove).
- IAM policies scope `Action` and `Resource` — no `"*"`/`"*"`; everything is tagged for ownership/cost.

### 3. Containers & Kubernetes
- Dockerfile: **non-root** `USER`, pinned base image (digest), multi-stage build, no secrets in layers/`ARG`, `.dockerignore` present, `COPY` over `ADD`, a `HEALTHCHECK`.
- K8s: **resource requests AND limits** set (noisy-neighbor / OOM), liveness + readiness (+ startup) probes, `securityContext` with `runAsNonRoot`, `readOnlyRootFilesystem`, dropped capabilities.
- No `privileged`, `hostPath`, `hostNetwork`, or `:latest` image tag; pinned digest in manifests.
- A rolling-update strategy + `PodDisruptionBudget` so a node drain / rollout doesn't take all replicas at once.
- Secrets mounted from a secret store / sealed, **not plaintext env** in the manifest; image scanning runs.
- Namespace, **least-privilege RBAC** (no cluster-admin to a workload), and a NetworkPolicy where the default is open.

### 4. Deploy & release safety
- A **rollback path exists and is tested** for every change — not "we'll roll forward."
- Risky changes roll out progressively (canary / blue-green / staged), not big-bang to 100%.
- DB migrations are **backward-compatible during rollout** (expand/contract) so old and new code coexist — schema content → **database-review**.
- A **health gate** guards promotion and triggers **automatic rollback** on failure; smoke tests run post-deploy.
- The app handles **SIGTERM** and drains in-flight work; readiness flips before shutdown (zero-downtime).
- Breaking changes ship behind a **feature flag / kill switch**, not coupled into an irreversible deploy.
- Deploy scripts are **idempotent**; schema + code + infra changes that must be reversible aren't fused into one irreversible step.

### 5. Config, secrets & observability
- Config is **separated from code** and per-environment; nothing baked into the image that differs by env.
- Secrets come from a **manager** (Vault/SSM/Secrets Manager/sealed), never committed or plaintext in manifests — cross-ref **security-review**.
- The new component ships **dashboards + alerts** on the golden signals (latency, traffic, errors, saturation); alerts fire on symptoms with a `for:` and a **runbook**.
- Structured logs are shipped/retained; traces/correlation IDs flow; an **SLO** and on-call owner exist.
- Autoscaling has **min/max bounds**; a **budget/cost alert** guards against runaway spend.
- IAM/service-account roles are **least-privilege** for the new component.

## Reference guides

Load the guide that matches the change. Don't load all of them — pull the one the diff touches.

| Load this | When the change involves |
| --- | --- |
| `reference/cicd-pipelines.md` | GitHub Actions / GitLab / Jenkins / CircleCI, pipeline triggers, secrets, OIDC, deploy gates, caching |
| `reference/iac-terraform.md` | Terraform / CDK / Pulumi / CloudFormation, state backends, providers, IAM policies, destructive replaces |
| `reference/containers-kubernetes.md` | Dockerfiles, container builds, K8s manifests, Helm/Kustomize, probes, securityContext, RBAC |
| `reference/deploy-release-safety.md` | deploy scripts, rollout/canary/blue-green config, migration ordering vs deploy, rollback, graceful shutdown |
| `reference/config-secrets-observability.md` | config/env wiring, secret injection, dashboards, alerts, tracing, autoscaling bounds, cost/IAM |
| `assets/infra-review-checklist.md` | a copy-pasteable PR checklist for a human reviewer |

## Output format

Write findings as inline comments. Each has: a **severity label**, a **`file:line`** anchor, a one-line **WHY** (the concrete impact — outage, data loss, exposure, cost, no rollback), and a **concrete fix** — show the corrected config when the fix isn't obvious.

> 🔴 **blocking** — `infra/rds.tf:24`
> Renaming `engine_version` here forces a **replace** of the `aws_db_instance`, which **destroys the production database and recreates it empty** — irreversible data loss on `apply`. Run `terraform plan` and confirm the diff shows `# forces replacement`; if so, do an in-place upgrade (modify, don't replace) and add `lifecycle { prevent_destroy = true }` to this resource so a future replace can't slip through.

> 🔴 **blocking** — `.github/workflows/ci.yml:8`
> This job triggers on `pull_request_target` and checks out `github.event.pull_request.head.ref`, so a forked PR can run **attacker-controlled code with repo secrets in scope** (pwn-request) and exfiltrate them. Build untrusted PRs on `pull_request` with **no secrets**, and gate any privileged step behind a separate `workflow_run`/approved environment.

> 🟠 **important** — `k8s/api-deployment.yaml:41`
> No `resources.limits` on this container. Under load one pod can consume a whole node's memory and OOM-kill its neighbors (noisy neighbor). Set `requests` and `limits` for cpu and memory.

> 🌟 **praise** — `.github/workflows/deploy.yml:30`
> Nice — cloud auth is via OIDC with `permissions: id-token: write` scoped to this job, no long-lived AWS keys in secrets. Exactly the pattern to copy.

Close with a verdict block:

```
## Verdict: ⛔ block

🔴 blocking (2)
- rds.tf:24 — force-replace destroys prod database (irreversible)
- ci.yml:8 — pull_request_target runs fork code with secrets (pwn-request)

🟠 important (1)
- api-deployment.yaml:41 — no resource limits (OOM / noisy neighbor)

🟡 nit (1) · 🔵 suggestion (1) · 🌟 praise (1)

Summary: Two blockers that risk prod data loss and secret exfiltration —
must fix before this can merge. The limits gap is important but secondary.
Re-request review after the RDS replace and the workflow trigger are fixed.
```

Pick the verdict honestly: `✅` clean, `💬` only nits/suggestions, `🔁` important issues to address, `⛔` a blocker that risks an outage, data loss, or secret exposure.
