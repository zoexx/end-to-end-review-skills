---
name: performance-review
description: |
  Reviews a change for performance across the whole stack — frontend bundle and render cost, backend latency and throughput, caching, and behavior under load. It's a cross-cutting perf lens, not a layer: it spots the round trip, the accidental O(n²), the missing cache, and the thing that holds at 1x but melts at 10x, then prioritizes them by how hot the path is and how big the regression. It insists on a number — measure, don't guess — and names the user-facing cost (+300ms p95, OOM under load, timeout at scale).
  Use when: reviewing for performance, perf review, investigating latency/slowness, reviewing a hot path, checking caching, capacity/scalability review, reviewing for p95/p99 latency, profiling a change, performance budgets.
allowed-tools:
  - Read
  - Grep
  - Glob
  - Bash
  - WebFetch
---

# performance-review

A senior-level performance pass that cuts across frontend, backend, and data instead of living in one layer. It reads a change against its performance budget, asks where it sits on the hot path, and works down to the line level — but its first move is always to ask for a number. **Measure, don't guess.** A finding without a measured or near-certain cost is a hypothesis, not a blocker, and premature optimization of a cold path is itself a defect (wasted effort, added complexity, no user benefit).

This skill prioritizes; the layer skills own the depth. Frontend, backend, and database-review each carry the line-level guidance for their domain. This one is the lens that ranks perf issues *across* them and catches the cross-layer ones a single-layer review misses.

## When this fires

- A PR touches a hot path — a request handler, a render path, a loop over user-scaled data, a query on a growing table, a job that runs per-request.
- Someone asks for a "performance review," "is this fast enough?", "will this hold at scale?", "why is p95 up?", or a "capacity/scalability review."
- A change adds a network round trip, a new dependency on a hot path, a cache, a bundle, a fan-out, or unbounded growth.
- A profile, flame graph, load test, or benchmark needs reading.
- The `end-to-end-review` orchestrator routes a perf-sensitive change here.

Out of scope as *depth* (this skill flags and prioritizes, then defers): Core Web Vitals mechanics → **frontend-review**; index design, query plans, lock behavior → **database-review**; event-loop/concurrency correctness and N+1 mechanics → **backend-review**. This skill cross-references them rather than duplicating them.

## The review model

Four phases, in order. Don't micro-optimize a line before you know whether the path is hot.

1. **Context** — What's the performance budget or SLO (p95 target, bundle ceiling, throughput goal)? Where does this code sit on the hot path — every request, or a once-a-day admin job? What scale does it run at — 10 rows or 10 million, 10 RPS or 10k? Read the PR description, ask for the current baseline numbers and the profile/load-test if there is one. Cost is relative to hotness and scale; the same code is a 🔴 on the checkout path and a 🟡 in a nightly batch.
2. **High-level** — Algorithmic complexity and architecture. Does this add a round trip, an O(n²), a sequential await that could be parallel, unbounded growth, work-per-request that could be precomputed or cached? Is a heavy dependency landing on a hot path? A wrong shape caught here outweighs a hundred micro-optimizations.
3. **Detailed** — Line by line, using the reference guides below. Allocations in hot loops, payload size, cache key correctness, bundle splitting, backpressure. Most findings land here, each tied to a measured or estimated cost.
4. **Verdict** — Summarize, group findings by severity, give one explicit decision.

### Severity labels

Every finding carries exactly one. **Perf severity ≈ how hot the path is × the size of the regression**, and it's anchored to evidence: a 🔴 is a *measured or near-certain* user-facing regression, never a micro-optimization or a guess.

| Label | Meaning |
| --- | --- |
| 🔴 **blocking** | A measured or near-certain user-facing regression on a hot path — +Xms p95, OOM/timeout under load, won't hold expected scale. Must fix before merge. |
| 🟠 **important** | A real perf risk that should be fixed, but smaller, not-yet-measured, or on a warm (not scorching) path. |
| 🟡 **nit** | Minor inefficiency with negligible user impact; polish. |
| 🔵 **suggestion** | Optional optimization or alternative worth considering. |
| 📚 **learning** | Context/teaching about a perf concept, no action required. |
| 🌟 **praise** | A genuinely good perf decision worth reinforcing. |

When evidence is missing, don't inflate: say "this *looks* O(n²) on a hot path — what's the measured p95 with production-scale n?" and label it 🟠 pending a number, not 🔴 on a hunch.

### Verdict states

`✅ approve` · `💬 approve with comments` · `🔁 request changes` · `⛔ block`

### Principles

- **Review the code, not the coder.** "This `.includes` in a loop is O(n²)," never "you don't know Big-O."
- **Measure first.** Ask for the profile, the benchmark, the p95 before/after. Don't approve an optimization with no number proving it helped, and don't block a path no one has shown is hot.
- **Say why — name the user-facing cost.** "+300ms p95 on checkout," "OOMs the pod at 2x traffic," "bundle +180KB → +0.4s LCP on mobile." A finding without a cost is an opinion.
- **Optimize the hot path, not the cold one.** A 10x speedup of code that runs once a week is noise; a 10% speedup of the per-request path is real. Spend the complexity budget where the time is.
- **Stay in scope.** Flag pre-existing perf debt separately; don't hold the PR hostage to unrelated tuning.
- **Praise is signal.** Calling out a correct `Promise.all`, a single-flight cache, or a keyset paginator teaches the pattern.

## What I check

Six pillars. Each links to a reference guide for the dense version, and each line names the user-facing cost it prevents.

### 1. Measure first / profiling
- A benchmark, profile, flame graph, or before/after number justifies an optimization — or proves the regression. No number → no 🔴.
- Tail latency, not the average: judged on **p95/p99**, because the slow tail is what users feel and what amplifies in fan-out. (Cost prevented: an "improvement" that helps the mean while the p99 — the path users actually hit — gets worse.)
- Micro-benchmarks avoid the traps (JIT warmup, dead-code elimination, unrealistic n) so the number means something.
- A CI perf gate or baseline exists for paths that matter, so the next regression is caught automatically.

### 2. Algorithmic & hot-path cost
- No accidental O(n²) on the hot path — nested loops over the same collection, `.includes`/`.find`/`in` inside a loop (use a `Set`/`Map`), repeated re-sort. (Cost: fine at 100 items in dev, 8s and climbing at 10k in prod.)
- Work done per-request that could be precomputed, memoized, or cached once. (Cost: every request pays for something that changes hourly.)
- No needless allocations, copies, or serialization in hot loops; no doing in app code what the DB should do in one query. (Cost: GC pressure, CPU, memory churn under load.)
- Unbounded growth (a list/map/cache that only grows) is bounded or evicted. (Cost: slow memory leak → OOM after days of uptime.)

### 3. Backend latency & throughput
- Independent awaits run in parallel (`Promise.all`/`errgroup`), not sequentially; round trips and N+1s are batched. (Cost: 5 × 80ms sequential = 400ms that should be 80ms. N+1 mechanics → **backend-review**/**database-review**.)
- The event loop / request thread isn't blocked by sync CPU or IO. (Cost: one slow handler starves every concurrent request. Depth → **backend-review**.)
- Payloads are compressed (gzip/br), paginated, and projected to the fields used; connection pools are sized. (Cost: oversized JSON and per-request connects add latency to every call.)
- Fan-out has per-call timeouts and a budget; one slow downstream doesn't amplify into a tail-latency spike across the whole request.

### 4. Caching strategy
- The right thing is cached at the right layer (client/CDN/app/DB), with a correct, well-scoped cache **key** (no missing dimension → wrong-user bleed; no over-specific key → 0% hit ratio).
- TTL and invalidation are deliberate: bounded staleness, invalidation paired with writes — not "stale forever" and not "never cached." (Cost: stale prices/permissions, or a hot query hammering the DB.)
- Hot keys are protected from stampede (lock/singleflight/stale-while-revalidate/jittered TTL). (Cost: a key expires and 5,000 concurrent misses melt the DB — an outage caused by the cache *expiring*.)
- HTTP caching (`Cache-Control`/`ETag`/304) and a target hit ratio are in place where they apply; over-caching that causes stale bugs is flagged.

### 5. Frontend budgets
- Bundle stays within budget — routes code-split, heavy/optional code dynamically imported, no barrel-file or whole-lib imports defeating tree-shaking. (Cost: every shipped KB is parse/compile/execute time on a mid-range phone → worse LCP/INP.)
- No request waterfalls or over-fetching/over-rendering on the critical path; needed origins `preconnect`ed. (Cost: serial fetches stack latency before first paint.)
- Web Vitals are watched at a **budget** level (LCP/CLS/INP targets) with the mechanics deferred to **frontend-review**; budgets are enforced in CI (Lighthouse CI, bundlesize).

### 6. Load & scalability
- Stateless enough to scale horizontally; no per-instance state that breaks when you add a replica. (Cost: works on one box, corrupts/loses data behind a load balancer.)
- Backpressure, rate limiting, and bounded queues exist; a traffic spike sheds or degrades gracefully instead of falling over. (Cost: unbounded queue → memory blowup → cascading failure.)
- Memory is stable under sustained load (no leak/growth); FD/connection limits won't be hit. (Cost: fine for an hour, OOM after a day.)
- A capacity estimate answers "does this hold 10x?" — backed by a load test where the risk is real, not a vibe.

## How this relates to the other skills

This skill is the **prioritizing perf lens**, not a replacement for the layer skills. They own the line-level depth in their domain; this one ranks and connects.

| For line-level depth on… | Defer to | This skill still… |
| --- | --- | --- |
| Core Web Vitals mechanics (LCP/CLS/INP fixes, render internals) | **frontend-review** | sets and enforces the *budget* and ranks bundle/waterfall cost against backend cost |
| Index design, query plans, EXPLAIN, lock behavior | **database-review** | flags the N+1/round trip and prioritizes it by hot-path impact |
| Event-loop blocking, concurrency correctness, N+1 mechanics | **backend-review** | spots sequential-vs-parallel and per-request work, ranks it by latency cost |

The value this lens adds on top: it weighs a frontend KB against a backend millisecond against a DB round trip *in one ranking*, and catches the cross-layer regression (e.g. a cache removed in the app that quietly doubles DB load) that no single-layer review sees.

## Reference guides

Load the guide that matches the change. Don't pull all of them — read the one the diff touches.

| Load this guide | When the change involves |
| --- | --- |
| `reference/measuring-profiling.md` | Justifying or disproving a perf claim — percentiles, flame graphs, profilers per stack, micro-benchmark traps, load tools, CI perf gates, RUM vs lab |
| `reference/backend-latency-throughput.md` | Request handlers, sequential vs parallel awaits, round trips, payload/compression, pagination, pools, timeout budgets, batching, serialization cost |
| `reference/caching-strategy.md` | Any cache — what/where to cache, key design, TTL/invalidation, stampede protection, HTTP caching, hit-ratio, when *not* to cache |
| `reference/frontend-performance-budgets.md` | Bundles, code splitting, third-party scripts, request waterfalls, over-fetching, Web Vitals at budget level, enforcing budgets in CI |
| `reference/load-scalability.md` | Behavior under concurrency — statelessness, backpressure, queue depth, memory growth, capacity planning, graceful degradation, load-test evidence |
| `assets/performance-review-checklist.md` | A copy-pasteable PR checklist covering all six pillars, measure-first reminder on top |

## Output format

Write findings as inline comments. Each carries: a **severity label**, a **`file:line`** anchor, a one-line **WHY** that names the *measured or estimated cost*, and a **concrete fix** — show the corrected code when the fix isn't obvious. When the cost is a guess, say so and ask for the number.

> 🔴 **blocking** — `search/rank.ts:64`
> `results.filter(r => excludedIds.includes(r.id))` runs `Array.includes` inside a filter over the full result set — O(n·m). With ~5k results and ~2k exclusions that's ~10M comparisons per request; the profile shows this node at **180ms p95** on the search path. Build a `Set` once and look up in O(1).
> ```ts
> // before: O(n·m), 180ms p95
> const out = results.filter(r => excludedIds.includes(r.id));
> // after: O(n), sub-ms
> const excluded = new Set(excludedIds);
> const out = results.filter(r => !excluded.has(r.id));
> ```

> 🟠 **important** — `api/dashboard.ts:31`
> The three independent fetches (`user`, `billing`, `usage`) are `await`ed sequentially — ~90ms each, ~270ms total on a page-load endpoint. They don't depend on each other; run them with `Promise.all` to collapse to ~90ms. (No production number yet — worth confirming against p95, but the sequential shape is near-certain latency.)

> 🌟 **praise** — `cache/products.ts:22`
> Single-flight + jittered TTL on the product-detail cache. Exactly the stampede protection that keeps a hot key from melting the DB when it expires.

Close with a verdict block:

```
## Verdict: 🔁 request changes

🔴 blocking (1)
- search/rank.ts:64 — O(n·m) exclusion filter, +180ms p95 on search (use a Set)

🟠 important (2)
- api/dashboard.ts:31 — sequential awaits, ~270ms that should be ~90ms (Promise.all)
- jobs/import.ts:88 — unbounded in-memory accumulator; OOM risk on large imports

🟡 nit (1) · 🔵 suggestion (1) · 🌟 praise (1)

Summary: One measured hot-path blocker (the O(n·m) filter) and a sequential
fan-out that's easy latency to reclaim. Fix the Set lookup and parallelize the
dashboard fetches and this is good to merge. No load-test was attached for the
import job — please confirm it holds at expected file sizes.
```

Pick the verdict by the worst unresolved finding: a measured 🔴 → ⛔ block or 🔁; only 🟠 → 🔁 or 💬; nits/suggestions only → ✅ or 💬. If the only "blocker" is unmeasured, ask for the number before blocking.
