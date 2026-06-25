# Performance review checklist

Copy-pasteable into a PR. Grouped by the six pillars. Skip rows that don't apply; flag what does with a severity label (🔴 blocking · 🟠 important · 🟡 nit · 🔵 suggestion · 📚 learning · 🌟 praise).

> **Measure first.** A 🔴 is a *measured or near-certain* user-facing regression on a hot path — never a micro-optimization or a guess. Before flagging (or approving) anything, ask for the number: the profile, the benchmark, the before/after **p95/p99**. Severity ≈ how hot the path is × the size of the regression. Optimize the hot path, not the cold one.

## Measure first / profiling
- [ ] There's a **number** behind every optimization claim or regression flag — a profile, benchmark, or before/after p95
- [ ] Judged on **p95/p99**, not the average; tail-latency amplification considered for fan-out
- [ ] The hot path was identified from a **flame graph** (widest frame), not from reading the code
- [ ] Micro-benchmarks handle warmup + dead-code elimination + realistic `n`; endpoint claims use macro (load-test) evidence
- [ ] Load tests use an **open model / constant arrival rate** (no coordinated omission)
- [ ] A **CI perf gate** (bundlesize / Lighthouse CI / k6 threshold / benchstat) protects the path; a baseline was recorded
- [ ] Regression confirmed against **field/RUM**, not just one lab run
- [ ] No optimization merged with zero measurement; no cold path optimized at the cost of added complexity

## Algorithmic & hot-path cost
- [ ] No accidental **O(n²)** on the hot path — nested loops, `.includes`/`.find`/`in` inside a loop (use a `Set`/`Map`), repeated re-sort
- [ ] Per-request work that could be **precomputed/memoized/cached** is hoisted out of the request path
- [ ] No needless allocations, copies, or re-serialization in hot loops; no doing in app code what one DB query could do
- [ ] **Unbounded growth** (an ever-growing list/map/cache) is bounded or evicted — no slow leak → OOM

## Backend latency & throughput
- [ ] Independent awaits run in **parallel** (`Promise.all`/`errgroup`/`gather`), not serially
- [ ] No **N+1** / chatty round trips — batched via join/`IN`/DataLoader; one RPC over N (mechanics → backend/database-review)
- [ ] No sync CPU or blocking IO on a single-threaded **event loop**; heavy work offloaded (depth → backend-review)
- [ ] Responses **compressed** (gzip/br), **paginated**, and **projected** to fields used
- [ ] DB connections **pooled** and right-sized; outbound HTTP reuses connections (**keep-alive**)
- [ ] Every downstream call has a **timeout** that fits the parent's deadline **budget**; deadlines propagate
- [ ] Global **middleware** is cheap — no per-request DB round trip or in-band blocking IO

## Caching strategy
- [ ] Cached at the right **layer** for the data's scope (private vs public) and freshness need
- [ ] Cache **key** includes every input the output depends on — and nothing high-cardinality that kills the hit ratio
- [ ] Every entry has a **TTL** (bounded staleness even if invalidation is missed); TTLs are jittered
- [ ] Writes **invalidate** (or update) the cache; the pattern (cache-aside / write-through) is deliberate
- [ ] Hot keys are **stampede**-protected (single-flight / stale-while-revalidate / jittered TTL / early refresh)
- [ ] **Negative** results cached with a short TTL where misses are common or abusable
- [ ] In-process **memoization** is bounded (LRU/size/TTL) and correctly scoped — no per-request data on a singleton
- [ ] **HTTP** caching correct: `immutable` for hashed assets, ETag/304 for dynamic, `private/no-store` for user data
- [ ] **Hit ratio** justifies the cache; nothing strongly-consistent (balances, inventory, auth) is cached

## Frontend budgets
- [ ] Initial **bundle** has a budget and stays within it; routes are **code-split**
- [ ] Heavy/optional features (charts, editors, maps) are **dynamic-imported** on demand
- [ ] No whole-library or **barrel-file** imports defeating tree-shaking; per-function/leaf imports
- [ ] **Third-party scripts** deferred/lazy-loaded and budgeted; their real cost audited
- [ ] **Images** modern-format + responsive + compressed; **fonts** subsetted, woff2, weights minimized
- [ ] No client-side **waterfalls** — independent fetches parallelized; origins `preconnect`ed, critical resources `preload`ed
- [ ] No **over-fetching** (only fields/rows shown) or **over-rendering** (long lists virtualized)
- [ ] A **CI budget** gate (size-limit / Lighthouse CI) fails the build on regression
- [ ] Web **Vitals** tracked at a budget level (LCP/CLS/INP targets); render-internals fixes routed to **frontend-review**

## Load & scalability
- [ ] **Stateless** — no in-memory session/counter/rate-limit/local-disk state that breaks behind a load balancer
- [ ] **Backpressure & rate limiting** — bounded queues/concurrency, load shed (`503`) instead of unbounded accept
- [ ] **Queue depth** bounded and monitored; consumers scale to peak arrival rate; a DLQ exists
- [ ] **Memory stable** under sustained load — caches/memos bounded, listeners/timers/connections cleaned up; soak-tested
- [ ] **FD/connection limits** respected — resources closed on every path; outbound concurrency bounded
- [ ] No **single contention point** (global lock, hot row/key) serializing the whole service
- [ ] **Graceful degradation** — non-critical dependencies optional; load shedding under overload
- [ ] **Capacity** estimated (Little's Law); first ceiling identified; "holds 10x?" answered
- [ ] **Load/soak-test** evidence attached for scaling-sensitive changes, not just "works in dev"

## Verdict
- [ ] Findings grouped by severity with `file:line` + WHY (the **measured/estimated cost**) + concrete fix
- [ ] Unmeasured "blockers" downgraded to a request-for-a-number, not blocked on a hunch
- [ ] Explicit decision: ✅ approve · 💬 approve with comments · 🔁 request changes · ⛔ block
