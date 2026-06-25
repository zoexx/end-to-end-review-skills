# Deploy & Release Safety

Deep-dive for reviewing how a change actually reaches production. Most outages aren't caused by bad code — they're caused by *good code shipped badly*: all at once, with no health gate, with a database migration that the old and new versions can't both tolerate, and with no way back. The reviewer's one non-negotiable question for every change is: **what is the rollback path, and has it been tested?** Each section pairs a ❌ anti-pattern with a ✅ fix and names the incident it causes.

The expand/contract / backward-compat material here pairs with **database-review** (which owns migration *content*); this guide owns migration *ordering vs deploy*.

---

## 1. No rollback path — "we'll roll forward"

If the only recovery from a bad deploy is to write, review, and ship a fix, your mean-time-to-recovery is measured in the time it takes to code under pressure — during an outage. Every change needs a way back that works *now*.

❌ One-way deploy:
```text
deploy = build new image → replace prod → done   ❌ no kept previous version, no documented revert
```
✅ Keep the previous version deployable; make revert one action:
```bash
# immutable, versioned artifacts; rollback = re-point to the last-known-good
kubectl rollout undo deployment/api            # or: deploy the previous image tag
# rollback is a tested runbook step, not "we'll figure it out"
```
> Incident: a deploy introduces a subtle bug that only shows at prod scale; the team's plan is to "roll forward with a fix," so the outage lasts as long as it takes to diagnose, code, review, and re-deploy — an hour of downtime that a one-command rollback would have ended in 30 seconds. Keep the last-known-good artifact deployable and make rollback a single, tested action. A change whose rollback path is "roll forward" is a 🔴.

---

## 2. Big-bang rollout instead of canary / blue-green

Shipping to 100% of traffic at once means any bug you didn't catch hits every user simultaneously — the blast radius is your entire user base before you even see the first alert.

❌ All-at-once:
```text
deploy new version → 100% of traffic immediately   ❌ a bug is a full outage, not a blip
```
✅ Progressive delivery — expose to a slice, watch, then promote:
```yaml
# canary: 5% → 25% → 50% → 100%, with metric checks between steps (Argo Rollouts / Flagger)
strategy:
  canary:
    steps:
      - setWeight: 5
      - analysis: { templates: [{ templateName: error-rate-and-latency }] }   # gate (see §4)
      - setWeight: 50
      - analysis: { templates: [{ templateName: error-rate-and-latency }] }
```
> Incident: a release with a memory leak (or a bad query) goes to 100% instantly; within minutes every pod is degraded and all users are affected before the on-call even acknowledges the page. A canary would have shown the regression on 5% of traffic, failed the analysis gate, and auto-rolled-back — affecting a sliver of users for a couple of minutes instead of everyone. Match the rollout strategy to the risk: canary/blue-green for anything user-facing or stateful.

---

## 3. Migrations that aren't backward-compatible during rollout

During any non-atomic rollout (canary, rolling update, blue-green cutover) **old and new code run at the same time** against **one database**. A migration that the currently-running old code can't tolerate breaks production *during* the deploy.

❌ Destructive schema change shipped with the code that needs it:
```sql
-- migration runs, then new pods roll out  ❌
ALTER TABLE users DROP COLUMN full_name;          -- old pods still SELECT full_name → errors
ALTER TABLE users RENAME COLUMN email TO email_address;  -- old code breaks instantly
```
✅ Expand/contract across separate, ordered deploys:
```text
1. EXPAND  : add new nullable column / new table (old + new code both work)         [deploy A]
2. MIGRATE : backfill data; new code writes both old+new, reads new                  [deploy B]
3. CONTRACT: once no running code reads the old column, drop it                       [deploy C, later]
```
> Incident: a deploy runs `DROP COLUMN`/`RENAME` and then rolls out new pods; for the 90 seconds the old pods are still serving, every request that touches that column throws — a self-inflicted partial outage in the middle of a "safe" rolling deploy. The fix is expand/contract: make schema changes additive and backward-compatible so old and new code coexist, and defer the destructive `CONTRACT` step to a later deploy once nothing reads the old shape. (Migration *content* / locking → **database-review**.)

---

## 4. No health gate / no automatic rollback

A deploy that promotes to the next stage on a timer (or on "the pods are Running") rather than on health metrics will happily roll a broken version to 100%.

❌ Promote regardless of health:
```text
deploy → wait 60s → promote to 100%   ❌ doesn't look at error rate or latency
```
✅ Gate promotion on golden-signal metrics; auto-rollback on breach:
```yaml
analysis:
  metrics:
    - name: error-rate
      successCondition: result < 0.01            # < 1% errors to advance
      provider: { prometheus: { query: 'rate(http_requests_total{status=~"5..",app="api"}[2m]) / rate(http_requests_total{app="api"}[2m])' } }
    - name: p99-latency
      successCondition: result < 0.5             # < 500ms p99
# failure → automatic rollback to previous version, no human needed
```
> Incident: a canary at 5% is clearly erroring, but the rollout promotes on a fixed timer, so it advances to 50% and then 100% — turning a contained 5% problem into a full outage because nothing checked the metrics between steps. Gate every promotion on error-rate/latency/saturation and configure **automatic rollback** on a failed gate, so a bad canary reverts itself before a human is even paged.

---

## 5. No graceful shutdown — dropped in-flight requests

When a pod is terminated (deploy, scale-down, node drain), Kubernetes sends `SIGTERM`. A process that exits immediately drops every in-flight request and any unfinished work — so a routine deploy throws errors at users.

❌ Ignores SIGTERM / exits hard:
```js
// no SIGTERM handler → process dies mid-request on every rollout  ❌
```
✅ Drain: flip readiness off, finish in-flight work, then exit:
```js
process.on("SIGTERM", async () => {
  server.closeIdleConnections();            // stop accepting new
  readiness.fail();                         // pull out of the LB / Service
  await server.close();                     // let in-flight requests finish
  await db.drain();                         // flush queues / close cleanly
  process.exit(0);
});
// give it time: terminationGracePeriodSeconds + a preStop sleep to deregister from the LB first
```
> Incident: every deploy produces a spike of 502s because pods are killed mid-request; users see errors during what should be a zero-downtime rollout, and queue consumers lose the message they were processing. Handle `SIGTERM` to fail readiness, stop accepting new work, drain in-flight requests, and only then exit — and set `terminationGracePeriodSeconds` (plus a `preStop` hook) long enough for the load balancer to deregister the pod before it stops.

---

## 6. Breaking change shipped to everyone at once

A change that breaks a consumer (an API contract change, a new required field, an incompatible message format) shipped all-at-once gives you no time to catch it and no way to limit who's hit.

❌ Breaking change behind no toggle:
```text
ship new required request field / removed endpoint → all clients break the moment it deploys   ❌
```
✅ Make it backward-compatible, or gate it behind a flag rolled out gradually:
```ts
if (flags.enabled("new-checkout-flow", { userId })) {   // ramp 1% → 100%, kill instantly if bad
  return newCheckout(req);
}
return legacyCheckout(req);     // old path stays live until the flag is fully on and proven
```
> Incident: a release removes an endpoint (or adds a required field) that mobile clients still call; the moment it deploys, every old client breaks, and because it's coupled into the deploy you can't turn it off without another full release. Decouple risky behavior changes from the deploy with a feature flag you can ramp and instantly disable, and keep API changes additive/versioned (cross-ref **backend-review**) so old consumers keep working.

---

## 7. No feature flag / kill switch for risky behavior

Even non-breaking changes carry risk. Without a kill switch, the only way to disable a misbehaving new code path is a rollback or a hotfix — slow, and impossible if the bad path is entangled with good changes in the same deploy.

❌ New risky path always on, no off-switch:
```ts
return runExperimentalPricing(cart);   // ❌ if it's wrong, you must roll back the whole deploy
```
✅ Guard it; default safe; flip without deploying:
```ts
return flags.enabled("experimental-pricing")
  ? runExperimentalPricing(cart)
  : runStablePricing(cart);            // ✅ kill switch: disable the path in seconds, no deploy
```
> Incident: a new pricing path computes wrong totals under an edge case discovered in prod; because there's no flag, disabling it means rolling back a deploy that also contained five unrelated good changes — so you either revert all of it or ship a hotfix under pressure. A kill switch lets you turn off just the bad path in seconds and keep everything else. Flags should default safe (off) and be readable consistently across instances.

---

## 8. Non-idempotent / fragile deploy scripts

A deploy script that fails halfway and can't simply be re-run leaves the environment in a broken in-between state — and re-running it makes things worse (duplicate resources, double-applied steps).

❌ Steps that break on re-run:
```bash
kubectl create configmap app-config --from-file=config/   # ❌ fails "already exists" on retry
aws s3 mb s3://app-assets                                  # ❌ errors if bucket exists → script aborts dirty
```
✅ Idempotent, declarative, safely re-runnable:
```bash
kubectl apply -f config/                                  # apply = create-or-update, re-runnable
aws s3api head-bucket --bucket app-assets || aws s3 mb s3://app-assets   # guard side-effects
set -euo pipefail                                          # fail fast, don't continue past an error
```
> Incident: a deploy script dies at step 7 of 12 (a transient registry timeout); re-running it aborts at step 2 with "configmap already exists," so the environment is stuck half-deployed and someone has to hand-unwind it under pressure. Make deploy steps idempotent (`apply`, `||`-guards, upserts) and `set -euo pipefail` so a partial failure can simply be re-run to completion rather than requiring manual surgery.

---

## 9. Fusing schema + code + infra into one irreversible step

Coupling a destructive schema change, a code change, and an infra change into a single deploy creates a step that can't be partially rolled back — if the code is bad, you can't revert it without also reverting (or being blocked by) the irreversible schema change.

❌ One deploy does it all, irreversibly:
```text
deploy: DROP old table + new app version + delete old infra   ❌ code rollback now impossible
```
✅ Sequence reversible steps; defer the irreversible one:
```text
1. additive schema (expand) — reversible
2. new code that works with both old+new schema — reversible (rollback to old code still works)
3. ...soak, verify in prod...
4. destructive cleanup (contract / delete old infra) — only after rollback is no longer needed
```
> Incident: a deploy drops the old table *and* ships new code in one shot; the new code has a bug, but you can't roll back to the old version because it needs the table you just dropped — you're trapped, forced to roll forward under maximum pressure. Keep each deploy individually reversible and push the one irreversible action (drop, delete, destroy) to a separate, later step that only runs once the previous version is provably retired. (Pairs with §1 and §3.)

---

## 10. No post-deploy smoke test

A deploy that's declared "done" the moment pods are running has verified nothing about whether the application actually works end-to-end in production.

❌ Deploy and walk away:
```text
rollout complete → mark deploy successful   ❌ never exercised a real request path
```
✅ Run a smoke test against prod; fail (and roll back) if it fails:
```bash
# after rollout, before declaring success — hit the critical paths
curl -fsS https://api.prod/healthz
curl -fsS -X POST https://api.prod/v1/echo -d '{"ping":1}' | grep -q '"pong"'
# non-zero exit → trigger rollback (ties into the health gate, §4)
```
> Incident: a deploy succeeds at the orchestration level — all pods `Running` — but a misconfigured env var means the app can't reach its database, and nobody notices until customer reports roll in, because nothing actually exercised a real request after the deploy. A handful of post-deploy smoke checks against the critical paths catch "green deploy, broken app" immediately and can trigger the same automatic rollback as the health gate.

---

## Quick scan checklist

- [ ] A **tested rollback path** exists for this change — one action, not "roll forward."
- [ ] Risky/user-facing changes roll out **progressively** (canary/blue-green), not big-bang to 100%.
- [ ] Migrations are **backward-compatible during rollout** (expand/contract); destructive `CONTRACT` is a later deploy.
- [ ] Promotion is gated on **golden-signal metrics**; a failed gate triggers **automatic rollback**.
- [ ] The app handles **SIGTERM**: fails readiness, drains in-flight work, then exits; grace period + `preStop` set.
- [ ] Breaking changes are additive/versioned or behind a **feature flag** rolled out gradually.
- [ ] Risky new behavior has a **kill switch** that defaults safe and flips without a deploy.
- [ ] Deploy scripts are **idempotent** and `set -euo pipefail` — a partial failure can be re-run.
- [ ] No single deploy fuses code + schema + infra into one **irreversible** step; the irreversible action is deferred.
- [ ] A **post-deploy smoke test** exercises critical paths and can trigger rollback on failure.
