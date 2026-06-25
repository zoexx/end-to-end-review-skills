# Measuring & profiling

The foundation pillar: **measure, don't guess.** Every other guide in this skill assumes you have a number. This one is how you get a trustworthy one, read it, and gate on it in CI. Each section pairs a ❌ way that lies to you with a ✅ way that doesn't, and names the cost of getting it wrong.

The single rule that prevents the most wasted work: **profile before you optimize, and measure after.** An optimization with no before/after number is a guess that adds complexity for an unproven gain — itself a defect.

---

## 1. Percentiles, not averages — why p99 over the mean

The average hides the users who are suffering. Latency distributions are right-skewed: a handful of slow requests drags the tail far past the mean, and those slow requests are disproportionately the ones that matter (a user making 10 requests to render a page hits the p99 of *at least one* with high probability).

❌ Reporting and gating on the mean:
```
avg response time: 45ms   ← looks great
```
✅ Report the distribution, gate on the tail:
```
p50  40ms   p95  120ms   p99  850ms   p99.9  3.2s   max  11s
```
> Cost of averaging: a change that improves the mean from 45ms→40ms while pushing p99 from 300ms→850ms is a *regression* for the users on the tail — and the mean told you it was a win. The slow tail is what fills support tickets and trips timeouts.

Why the tail dominates a page: if rendering a page needs 20 backend calls and each has a 1% chance of being slow (p99), the chance the *page* hits at least one slow call is `1 − 0.99²⁰ ≈ 18%`. **Tail latency amplifies under fan-out** — the more calls per user action, the more your p99 becomes their p50. Always optimize and gate on p95/p99, never the average.

---

## 2. Tail-latency amplification in fan-out

A request that fans out to N backends waits for the *slowest* of the N, so the parent's latency is the **max** of its children, not the average.

❌ Sizing a fan-out by the average child latency ("each call is ~50ms, so ~50ms total"). With 50 parallel calls each at p99=500ms, the parent's p99 is dominated by whichever child hit its tail — effectively the p99 of the slowest, often seconds.

✅ Mitigations to look for in review:
- **Per-call timeouts** with a budget so one slow child can't hold the whole request (see backend-latency-throughput §6).
- **Hedged requests** (fire a second copy after a short delay, take the first to return) for read-only idempotent calls.
- Reducing fan-out width, or returning partial results when a non-critical child times out (graceful degradation, load-scalability §7).

> Cost: a dashboard that fans out to 30 services feels fine in testing (you rarely hit a tail) but is routinely slow in production because *someone's* child is always on its p99.

---

## 3. Flame graphs & sampling profilers — read before you cut

A flame graph shows where wall-clock or CPU time actually goes: width = time spent, stacked by call depth. **Find the widest frame on the hot path; that's where the time is.** Optimizing anything else is guessing.

❌ "This function looks expensive, let me optimize it" — based on reading the code.
✅ Profile, find the widest frame, optimize *that*, re-profile to confirm it moved.

Profiler per stack (sampling profilers are low-overhead and safe in prod-like settings):

| Stack | Profile CPU / flame graph | Notes |
| --- | --- | --- |
| **Node.js** | `node --prof` then `--prof-process`; **`clinic flame`** / `clinic doctor`; `0x`; `--cpu-prof` for Chrome DevTools | clinic also surfaces event-loop blocking |
| **Python** | **`py-spy record -o out.svg --pid <pid>`** (sampling, no code change, attaches to running process); `cProfile` + `snakeviz` for deterministic | py-spy is the go-to for prod; cProfile adds overhead |
| **Go** | **`pprof`** — `import _ "net/http/pprof"`, then `go tool pprof http://.../debug/pprof/profile`; flame graph via `-http=:8080` | also heap, block, mutex profiles |
| **JVM** | **`async-profiler`** (flame graphs, low overhead, CPU+alloc+lock); JFR (Flight Recorder) | avoid pure-sampling JVMTI profilers that suffer safepoint bias |
| **Browser** | **Chrome DevTools Performance panel** (record interaction → flame chart, long tasks); the Performance Insights panel | for INP/long-task hunting — frontend-performance-budgets §6 |
| **Ruby** | `stackprof`, `rbspy` | rbspy is py-spy's sibling |
| **.NET** | `dotnet-trace`, `dotnet-counters`, PerfView | |

> Cost of skipping the profile: teams routinely spend days optimizing the function they *suspect* is slow, ship it, and see no movement — because the real cost was a different frame (often JSON serialization, a lock, or GC). The flame graph would have pointed straight at it.

For memory, use the heap profiler/allocation profiler (Node `--heap-prof` / heap snapshots, Go `pprof` heap, py-spy/`tracemalloc`, async-profiler `alloc`) — a CPU flame graph won't show a leak.

---

## 4. Micro-benchmark traps

Micro-benchmarks (timing one function in isolation) are where most bogus numbers come from. The runtime is smarter than the benchmark.

❌ Naive micro-benchmark that lies:
```js
const t0 = performance.now();
for (let i = 0; i < 1e6; i++) compute(i);   // result unused → may be eliminated
console.log(performance.now() - t0);          // also: no warmup → measures cold JIT
```
Three classic traps:
- **JIT/runtime warmup** — the first iterations run interpreted/cold; the optimizing compiler kicks in later. Measuring cold gives numbers 10x off. Warm up, then measure.
- **Dead-code elimination** — if the result is never used, the compiler may delete the work entirely, so you "benchmark" nothing and get a suspiciously fast result. Consume the result (accumulate it, return it, blackhole it).
- **Unrealistic n / data** — testing with n=10 when production n=100k hides the O(n²); testing with all-cache-hits hides the cold path.

✅ Use a real harness that handles warmup, statistical significance, and dead-code elimination:
```js
// Node: vitest bench / tinybench / benchmark.js
import { bench } from 'vitest';
bench('dedupe with Set', () => { blackhole(dedupe(input)); });
```
- **JS:** `tinybench`, `benchmark.js`, `vitest bench` · **Go:** `testing.B` (`go test -bench` — and use `b.N`, keep `runtime.KeepAlive`/assign to a package var to defeat DCE) · **Python:** `pytest-benchmark`, `timeit` (with `number`/`repeat`) · **JVM:** **JMH** (the gold standard — handles warmup, DCE via `Blackhole`, fork isolation) · **Rust:** `criterion`.

> Cost: a micro-benchmark that "proves" the new code is 5x faster, merged on that basis, that turns out to measure cold JIT vs warm, or eliminated dead code — the optimization ships, adds complexity, and helps nothing in production. Worse, it can be *slower* in the real macro context (cache effects, real data sizes).

**Micro vs macro:** a micro-benchmark isolates one function; a macro-benchmark (a load test against the real endpoint, §5) measures the system. Prefer macro evidence for hot-path claims — micro numbers don't account for IO, cache, contention, or real data distribution.

---

## 5. Load-testing tools (macro evidence)

For "does this endpoint hold up?" you need load against the real path, not a micro-benchmark.

- **k6** (JS-scripted, modern, great for CI; outputs p95/p99 and thresholds you can fail the build on).
- **Locust** (Python-scripted, good for complex user-flow scenarios).
- **wrk** / **wrk2** (very high throughput; wrk2 fixes coordinated omission — see below); **vegeta** (constant-rate, Go); **autocannon** (Node); **Gatling**, **JMeter** (heavier, scenario-rich).

❌ Coordinated omission — the classic load-test lie:
```
// a closed-loop tool that waits for each response before sending the next
// under-counts the tail: when the server stalls, the tool just... waits,
// so the slow period generates fewer (not more) recorded samples
```
✅ Use an **open-model / constant-arrival-rate** load (k6 `constant-arrival-rate`, wrk2, vegeta) so a stall during the test still records the requests that *would have* arrived — otherwise your p99 is fictionally good.

```js
// k6: constant arrival rate + a pass/fail threshold
export const options = {
  scenarios: { load: { executor: 'constant-arrival-rate', rate: 500, timeUnit: '1s',
    duration: '2m', preAllocatedVUs: 100 } },
  thresholds: { http_req_duration: ['p(95)<200', 'p(99)<500'] },  // build fails if breached
};
```

> Cost of closed-loop / coordinated omission: a load test that reports p99=120ms when real p99 is 4s, because the tool stopped sending traffic exactly when the server got slow. You ship "validated" capacity that doesn't exist.

---

## 6. CI perf regression gates & baselines

A perf number is only durable if something checks it on every change. Without a gate, the path that's fast today drifts slower one PR at a time.

✅ Look for (or recommend) a gate on the paths that matter:
- **Bundle size:** `bundlesize`, `size-limit`, or `@next/bundle-analyzer` failing the build when a budget is exceeded (frontend-performance-budgets §8).
- **Web Vitals:** **Lighthouse CI** (`lhci autorun`) with assertions on LCP/CLS/TBT against a budget.
- **Backend latency/throughput:** a k6 run in CI with `thresholds` that fail the build; or a benchmark-tracking action (e.g. `benchmark-action`, `bencher`) that compares against the stored baseline and comments the delta.
- **Allocations/CPU:** Go `benchstat` comparing `-bench` results across commits; JMH gates.

❌ "We'll notice if it gets slow." (You won't — perf rot is gradual and invisible until it's a 🔴.)

> Cost: without a gate, a year of "tiny" 5ms regressions compounds into a 200ms-slower endpoint that nobody can bisect, because no single PR looked guilty.

**Establish a baseline first:** you can't gate on a delta you never measured. The first step on any perf-sensitive path is to record the current p50/p95/p99 (and bundle size / LCP) so future changes have something to compare against.

---

## 7. RUM vs lab — field data is the source of truth

**Lab** (synthetic: Lighthouse, a k6 run, your laptop) is reproducible and good for *catching regressions in CI*. **RUM** (Real User Monitoring: actual users' devices and networks) is *what's actually happening*. They disagree, and RUM wins for "is this really a problem?"

- Lab can't reproduce your real traffic mix, device distribution (that mid-range Android), network conditions, cache hit ratios, or data sizes.
- A change can look neutral in the lab and regress badly in the field (or vice versa) because your test data/device isn't your users'.

✅ When a regression is suspected, confirm it against field/RUM where available:
- **Frontend:** the `web-vitals` library, **Chrome UX Report (CrUX)**, or your analytics — field p75 of LCP/CLS/INP (frontend-review owns the mechanics).
- **Backend:** your APM / metrics (Datadog, Prometheus histograms, OpenTelemetry) — the *production* p95/p99 by endpoint, not a synthetic run.

> Cost: optimizing for the lab score while field p75 doesn't move — or worse, dismissing a real field regression because "Lighthouse looks fine on my machine." The lab is a proxy; the field is the truth.

---

## Quick scan checklist

- [ ] There's a **number** — a profile, benchmark, or before/after p95 — behind any optimization claim or regression flag.
- [ ] Judged on **p95/p99**, not the average; tail-latency amplification considered for fan-out.
- [ ] Hot path identified from a **flame graph** (widest frame), not from reading the code.
- [ ] Micro-benchmarks handle warmup + dead-code elimination + realistic n; macro (load-test) evidence used for endpoint claims.
- [ ] Load test uses an **open model / constant arrival rate** (no coordinated omission).
- [ ] A **CI gate** (bundlesize / Lighthouse CI / k6 threshold / benchstat) protects the path; a baseline was recorded.
- [ ] Regression confirmed against **field/RUM**, not just one lab run.
- [ ] No optimization merged with zero measurement; no cold path optimized at the cost of complexity.
