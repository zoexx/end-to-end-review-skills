# Config, Secrets & Observability

Deep-dive for reviewing how a component is configured, how it gets its secrets, and whether anyone can tell what it's doing in production. A new service that can't be configured per-environment, leaks its secrets, has no dashboard, and can autoscale without bound is a future incident waiting for a trigger. The reviewer's job: **config injected not baked, secrets from a manager not the repo, golden-signal observability on the new thing, and guardrails on cost and access.** Each section pairs a ❌ anti-pattern with a ✅ fix and names the incident it causes.

Secret-handling depth → **security-review** (this guide flags exposure and wiring); alerting/metrics mechanics also appear in **backend-review** observability (this guide owns the infra/deploy angle: per-env config, the new component's dashboards, autoscaling, cost, IAM).

---

## 1. Config baked into the image instead of injected

When environment-specific config is compiled into the image, you need a different image per environment — so "the thing you tested in staging" is not "the thing you ship to prod," and a config change means a rebuild.

❌ Per-env values hardcoded / built in:
```dockerfile
ENV API_URL=https://api.staging.example.com   # ❌ this image only works in staging
ENV LOG_LEVEL=debug                            # ❌ debug baked into the prod image too
```
✅ One image, config injected at runtime:
```yaml
# same image everywhere; the orchestrator supplies env-specific config
envFrom:
  - configMapRef: { name: api-config }     # per-environment ConfigMap
  - secretRef:    { name: api-secrets }    # per-environment Secret (from a manager, §3)
```
> Incident: the staging API URL is baked into the image, so the artifact validated in staging can't be promoted to prod as-is — a separate prod build is made, which differs in some untested way, and *that* untested image is what breaks. Build one immutable image and inject all environment-specific config at runtime, so the exact artifact you tested is the one that ships (the [12-factor](https://12factor.net) config principle).

---

## 2. No per-environment separation

Sharing one config (or worse, one set of credentials) across environments means a staging change can hit prod data, and a prod incident can't be reproduced in staging.

❌ One config / one credential for all envs:
```text
DATABASE_URL points staging AND prod at the same database   ❌ staging test corrupts prod data
```
✅ Distinct config and credentials per environment:
```text
config/staging.yaml  → staging DB, staging keys, debug logging
config/prod.yaml     → prod DB, prod keys, info logging
# selected by environment, never shared; prod creds never present in staging
```
> Incident: staging and prod share a `DATABASE_URL`, so a load test or a destructive integration test run against "staging" wipes production data — because they were never actually separate. Keep config and credentials strictly per-environment; a non-prod environment must not hold prod credentials or point at prod data.

---

## 3. Secrets committed or in plaintext config

A secret in the repo, in a `ConfigMap`, in a `values.yaml`, or in a plaintext `.env` committed to git is exposed to everyone with read access and lives in history forever.

❌ Secret in version control / plaintext config:
```yaml
# values.yaml (committed)  ❌
database:
  password: "Pa55w0rd!"
apiKey: "sk_live_abc123…"        # ❌ in git history forever
```
✅ Secrets from a manager, referenced not embedded:
```yaml
# values.yaml references a secret name; the value lives in a manager
database:
  passwordSecretRef: prod/db-password   # resolved by External Secrets / Vault / SSM / Sealed Secrets
# CI/runtime fetches it from the manager at deploy time; nothing secret in the repo
```
> Incident: an API key in a committed `values.yaml` is now readable by everyone with repo access and is permanently in git history; even after it's removed, it must be **rotated** because the old commit still exposes it. Source every secret from a dedicated manager (Vault, AWS Secrets Manager/SSM, GCP Secret Manager, Sealed Secrets, External Secrets Operator) and keep only references in the repo. (Detection, rotation, and storage depth → **security-review**.)

---

## 4. No alerting on the new component

A component that ships without alerts is invisible until a customer complains. You can't operate what you can't see fail.

❌ New service, no alerts:
```text
deploy new payment-webhook-processor → no alert rules added   ❌ failures are silent
```
✅ Ship alerts with the component, on its golden signals:
```yaml
- alert: WebhookProcessorErrorRate
  expr: rate(webhook_processed_total{status="error"}[5m]) / rate(webhook_processed_total[5m]) > 0.05
  for: 5m
  annotations: { summary: "Webhook error rate > 5%", runbook: "https://…/runbooks/webhooks" }
- alert: WebhookProcessorLagHigh
  expr: webhook_queue_depth > 1000
  for: 10m
```
> Incident: a new webhook processor starts silently dropping events a week after launch; because no alert shipped with it, nobody notices until a partner asks why their callbacks stopped — days of lost events. Every new component must ship with alerts on its golden signals (latency, traffic, errors, saturation) so its failures page someone, not surprise a customer.

---

## 5. Missing dashboards / golden signals

Alerts tell you *something* is wrong; dashboards tell you *what*. A new component with no dashboard means every incident starts with "let me wire up a query first."

❌ No dashboard, no standard signals:
```text
new service has scattered logs and one ad-hoc metric   ❌ no at-a-glance health view
```
✅ A dashboard covering the four golden signals:
```text
Latency    : p50 / p95 / p99 request duration
Traffic    : requests/sec (by route, status class)
Errors     : error rate (5xx / failed), by cause
Saturation : queue depth, pool utilization, CPU/mem vs limits
# (RED for request services, USE for resources — see backend-review observability)
```
> Incident: during an incident the on-call has no dashboard for the new service, so the first 15 minutes are spent writing Prometheus queries by hand instead of diagnosing — time the outage runs while they tool up. Ship a golden-signals dashboard with the component so its health is visible at a glance from minute one. (Metric/label hygiene — low cardinality, RED/USE — is covered in **backend-review** observability.)

---

## 6. No tracing / correlation across the component

In a distributed deploy, a request crosses services and queues. Without trace/correlation IDs propagated, you can see each hop's logs but can't follow one request through the new component.

❌ No correlation through the new service:
```text
gateway → new-processor → downstream   ❌ no shared id; logs can't be stitched together
```
✅ Propagate trace context; emit spans:
```text
# adopt incoming W3C traceparent (or mint one at the edge) and pass it on every hop;
# every log line carries the trace/correlation id; the processor opens a span per unit of work
```
> Incident: a request that fans out through the new processor fails somewhere in the chain, but with no correlation id the on-call can't tell which log lines across four services belong to the same request — the investigation takes hours instead of one `traceId=…` query. Propagate W3C `traceparent` through the new component and stamp the correlation id on every log line. (Tracing mechanics → **backend-review** observability.)

---

## 7. No runbook / on-call owner / SLO

A component with no documented owner, no runbook, and no service-level objective has no defined "healthy" and no defined "who fixes it." Alerts fire into the void.

❌ Ships with no operational ownership:
```text
new service deployed; no SLO, no runbook, alerts route to nobody   ❌ "whose is this?"
```
✅ Define ownership, an SLO, and a runbook before/with launch:
```text
Owner   : team-payments (on-call rotation X)
SLO     : 99.9% of webhooks processed < 5s; error budget tracked
Runbook : https://…/runbooks/webhook-processor  (symptoms → diagnosis → mitigation → rollback)
# every alert links its runbook (see §4)
```
> Incident: an alert fires for the new service at 3am, pages a rotation that's never seen it, and there's no runbook — so the responder spends the outage reverse-engineering the system instead of mitigating. Define the owner, an SLO (so "is this bad?" has an answer), and a runbook (so the responder has steps), and route every alert to the owning rotation with its runbook attached.

---

## 8. Unbounded autoscaling — no min/max

Autoscaling without bounds cuts both ways: no upper bound means a traffic spike (or a retry storm) scales you into a massive bill or exhausts a downstream; no sensible lower bound means cold starts or zero capacity.

❌ No bounds:
```yaml
kind: HorizontalPodAutoscaler
spec: { minReplicas: 1, maxReplicas: 10000 }   # ❌ a spike scales to thousands → cost + downstream overload
```
✅ Sane min and max, tied to capacity:
```yaml
spec:
  minReplicas: 3          # survive a node loss, no cold-start floor
  maxReplicas: 40         # ceiling sized to DB connection limit / downstream capacity / budget
  metrics: [{ type: Resource, resource: { name: cpu, target: { type: Utilization, averageUtilization: 60 } } }]
```
> Incident: a retry storm or a bot flood drives autoscaling with `maxReplicas: 10000`; the service scales to thousands of pods, opens thousands of DB connections, exhausts the database's connection limit, and takes down *every* service sharing that database — plus a five-figure cloud bill for the hour. Bound `maxReplicas` to what downstream dependencies (DB connections, rate limits, budget) can actually absorb, and set `minReplicas` for resilience. (Same logic for cluster/node autoscaling and serverless concurrency limits.)

---

## 9. No budget / cost alert

Cloud spend is silent until the invoice. A misconfigured resource, a runaway autoscaler, or a forgotten environment can quietly burn money for a full billing cycle.

❌ No cost guardrail:
```text
provision new infra → no budget alert, no cost anomaly detection   ❌ surprise on the invoice
```
✅ A budget with alerts at thresholds:
```hcl
resource "aws_budgets_budget" "service" {
  budget_type = "COST"
  limit_amount = "2000"   # monthly
  limit_unit   = "USD"
  notification {          # alert at 80% and 100% (and an anomaly detector for spikes)
    comparison_operator = "GREATER_THAN", threshold = 80, threshold_type = "PERCENTAGE",
    notification_type = "ACTUAL", subscriber_email_addresses = ["fin-ops@example.com"]
  }
}
```
> Incident: a forgotten staging cluster (or a NAT-gateway-heavy data path) runs all month and the first anyone hears of it is a cloud bill thousands over expected — money already spent, unrecoverable. A budget with threshold alerts (and cost-anomaly detection) catches the spike on day two instead of in next month's invoice. Tag resources (see iac-terraform §8) so the cost is attributable when the alert fires.

---

## 10. IAM / service-account roles not least-privilege

The role attached to a component is its standing permission set. Over-broad roles mean any compromise of the component — or its leaked credentials — is a much larger breach.

❌ Component runs with a broad role:
```hcl
# the service's task/pod role has AdministratorAccess "to make it work"  ❌
managed_policy_arns = ["arn:aws:iam::aws:policy/AdministratorAccess"]
```
✅ A role scoped to exactly what the component does:
```hcl
# only the specific actions on the specific resources this service touches
statement {
  actions   = ["sqs:SendMessage", "sqs:ReceiveMessage"]
  resources = [aws_sqs_queue.work.arn]              # this queue only
}
# pair with workload identity (IRSA / Workload Identity), not a shared static key
```
> Incident: a service is given admin "to ship," then an SSRF (or a leaked task-role credential) lets an attacker assume that role and, because it's admin, read every bucket and delete every database in the account — a full breach from one service bug. Scope each component's role to the specific actions on the specific resources it needs, and bind it via workload identity (IRSA on EKS, Workload Identity on GKE, managed identity on Azure) so there's no long-lived key to leak. (IAM-in-IaC also in iac-terraform §7; secret/credential depth → **security-review**.)

---

## Quick scan checklist

- [ ] One immutable image; environment-specific **config injected** at runtime, not baked in.
- [ ] Config and credentials are **per-environment**; non-prod never holds prod creds or points at prod data.
- [ ] No secrets committed or in plaintext config; all secrets sourced from a **manager**, only references in the repo.
- [ ] The new component ships **alerts** on its golden signals, each with a `for:` and a runbook link.
- [ ] A **dashboard** covers latency / traffic / errors / saturation for the new component.
- [ ] Trace/correlation IDs are **propagated** through the component; every log line carries the id.
- [ ] An **owner, SLO, and runbook** exist; alerts route to the owning on-call rotation.
- [ ] Autoscaling has sensible **min/max** bounds tied to downstream capacity and budget — not unbounded.
- [ ] A **budget/cost alert** (and anomaly detection) guards against runaway spend; resources are tagged.
- [ ] The component's IAM/service-account role is **least-privilege** and bound via workload identity, not a static key.
