# Load & scalability

Deep-dive for the question single-request profiling can't answer: **what happens under concurrency, sustained load, and 10x traffic?** These are the failures that don't show up in dev or a single request — they emerge from many requests interacting, from running for days, or from a traffic spike. Each section pairs a ❌ anti-pattern with a ✅ fix and names the incident it prevents.

The reviewer's framing: a change can be fast for one user and still take down the service at scale. Ask not "is this request fast?" but "**does this hold when there are 10,000 of it?**"

---

## 1. Statelessness & horizontal scaling

Horizontal scaling (add more boxes) only works if a request can be served by *any* instance. Per-instance state breaks the moment you add a replica behind a load balancer.

❌ Per-process state that assumes one instance:
```ts
const sessions = new Map();           // ❌ in-memory session store — request 2 may hit another box
let requestCounter = 0;                // ❌ per-instance counter, not global
const uploads = '/tmp/uploads';        // ❌ local disk — the next request lands elsewhere
```
✅ Externalize shared state so any instance can serve any request:
```ts
// sessions/rate-limits/counters → Redis; uploads → object storage (S3/GCS); cache → shared
const session = await redis.get(`sess:${id}`);
```

> Cost: works perfectly on one box, then you scale to three behind a load balancer and users randomly lose their session (request hits a box without it), counters are wrong, and uploads vanish (saved to a different pod's local disk). The bug only appears *after* you scale — the worst time to discover it.

Things that quietly assume one instance: in-memory sessions, in-memory rate limiters, local-disk file storage, in-process job schedulers (two boxes run the cron twice), in-memory caches presumed shared, sticky-session dependence. Each needs to be externalized or made instance-agnostic before horizontal scaling works.

---

## 2. Backpressure & rate limiting

A system without backpressure accepts work faster than it can finish it — and unbounded acceptance under overload turns a slowdown into a crash.

❌ Accept everything, queue without bound:
```ts
queue.push(job);                       // ❌ no limit — a traffic spike grows the queue until OOM
app.post('/expensive', handler);       // ❌ no rate limit — one client can saturate the service
```
✅ Bound intake; reject or shed when full:
```ts
if (queue.length >= MAX_QUEUE) return res.status(503).set('Retry-After', '5').end();  // shed load
app.use('/expensive', rateLimit({ windowMs: 1000, max: 50 }));                          // cap intake
```
- **Rate limiting** caps how much work each client can submit (token bucket / sliding window; shared store so it holds across instances, §1).
- **Backpressure** propagates "I'm full" upstream — bounded queues/channels, `503 + Retry-After`, or pausing the consumer — so the system slows gracefully instead of accumulating unbounded work.
- **Bounded concurrency** on fan-out (a semaphore / worker pool) so a burst doesn't spawn 10k simultaneous downstream calls.

> Cost: without backpressure, a traffic spike or a slow downstream makes the in-flight/queued work grow without limit → memory climbs → OOM → the whole instance dies → its load shifts to the others → they OOM too. An unbounded queue converts a brief overload into a full cascading outage.

---

## 3. Queue depth & worker scaling

For async/queue-based work, the health metric is **queue depth over time** — and the rule is producers must not outpace consumers indefinitely.

❌ Fixed worker count that can't keep up; no depth visibility:
```
producers: 500 jobs/s   consumers: 1 worker @ 100 jobs/s   → depth grows 400/s, forever
```
✅ Scale consumers to the arrival rate; alert on growing depth:
- Size the worker pool / autoscale consumers to the *peak* arrival rate, not the average.
- Monitor queue depth and **age of the oldest message**; alert when depth trends up (it means consumers are losing).
- Use a **dead-letter queue** so poison messages don't wedge the pipeline and silently back everything up.

> Cost: a queue whose depth grows unbounded means every job waits longer and longer (latency climbs), the backlog never drains, and eventually the broker hits its limit or memory fills. By the time users notice "my job never ran," the backlog is hours deep. Queue depth is the leading indicator — watch it.

---

## 4. Memory growth & leaks under sustained load

A leak is invisible in a single request and a quick test; it only shows after hours/days of real traffic — then it's a periodic OOM and a restart cycle.

❌ Unbounded accumulation tied to request volume:
```js
const cache = new Map();               // ❌ never evicts — grows with unique keys forever
const listeners = [];
emitter.on('data', h); listeners.push(h);   // ❌ handler added per request, never removed
```
✅ Bound and clean up:
```js
const cache = new LRUCache({ max: 10_000, ttl: 60_000 });   // bounded (caching-strategy §7)
// remove listeners / clear timers / close resources on request end
req.on('close', () => emitter.off('data', h));
```

> Cost: a leak that adds 2MB per 1,000 requests looks fine in a 5-minute test and OOMs the pod every 6 hours in production — a restart-loop that pages someone, with no single request to blame. Common sources: unbounded caches/memos, accumulating event listeners/timers, growing closures, never-closed connections/streams. Confirm stability with a **soak/endurance load test** (steady traffic for hours, watch RSS) — a short test won't catch it.

---

## 5. Connection & file-descriptor limits

Every open socket, DB connection, and file is a finite OS/service resource. Leaking them, or opening too many, hits a hard ceiling that manifests as "can't connect" errors with the service otherwise healthy.

❌ Open without close, or unbounded outbound connections:
```python
f = open(path)                         # ❌ never closed in the error path → FD leak
conn = new_db_connection()             # ❌ per request, not pooled → exhausts max_connections
```
✅ Use context managers/pools; bound outbound concurrency:
```python
with open(path) as f: ...              # closed on every exit path
# DB → a sized pool (backend-latency §5); outbound HTTP → a shared keep-alive client
```

> Cost: leaked file descriptors hit the process `ulimit` → `EMFILE: too many open files` and the service can't accept connections or open files, while CPU/memory look fine — a confusing outage. Unpooled DB connections hit `max_connections` → new requests are rejected. Both are *limit exhaustion*, not slowness, so they don't show on a latency graph until they cliff.

---

## 6. Single points of contention — hot keys & global locks

A perfectly horizontal service can still bottleneck on one shared thing every request touches.

❌ A global lock or a single hot row/key on the request path:
```go
globalMu.Lock()                        // ❌ every request serializes through one mutex
counter++                              // ...so adding boxes doesn't help — they all wait here
globalMu.Unlock()
// or: every request increments the SAME DB row → row-lock contention, a serialization point
```
✅ Remove the single contention point — shard, batch, or make it lock-free:
```go
atomic.AddInt64(&counter, 1)           // lock-free for a simple counter
// hot DB row → shard into N rows and sum, or buffer+batch updates; partition the hot key
```

> Cost: a global mutex or a single hot key (a counter row, a "current id" sequence, a rate-limit key) serializes the whole service through one point — you add instances and throughput *doesn't improve* because every request still queues on that one lock/row. The fix is to eliminate the shared point (sharding, batching, atomic ops, partitioning the key).

---

## 7. Graceful degradation & load shedding

Under overload or a failed dependency, the service should do *less* but stay up — not try to do everything and fall over.

❌ All-or-nothing — one failed/​slow dependency fails the whole request:
```ts
const [feed, recommendations, ads] = await Promise.all([...]);  // ❌ if `ads` is slow, the page hangs
```
✅ Degrade non-critical pieces; shed load before collapse:
```ts
const feed = await getFeed();                                   // critical — must succeed
const recs = await getRecs().catch(() => []);                   // optional — empty on failure/timeout
// under heavy load: serve cached/stale, drop optional features, return 503 early (load shedding)
```
- **Degrade:** make non-essential features (recommendations, related items, live counts) optional — a timeout/failure returns a fallback, not an error.
- **Load shed:** when overloaded, reject the cheapest-to-reject work early (`503 + Retry-After`), serve stale cache, or disable expensive features — keep the core path alive.

> Cost: without degradation, a single slow non-critical dependency (the ads service) takes down the whole page; without load shedding, an overloaded service tries to serve *everything* slowly and fails *all* of it, instead of serving most requests well and rejecting the excess. Staying up at reduced function beats falling over completely.

---

## 8. Capacity planning — does it hold 10x?

The headline scalability question, answered with arithmetic and a load test, not a vibe.

**Little's Law** — the back-of-envelope every capacity estimate starts from:
```
L = λ × W
concurrency = arrival_rate × avg_latency
```
e.g. 500 req/s × 0.2s avg = **100 concurrent requests** in flight. That tells you how many workers/connections/threads you need *now*, and at 10x (5,000 req/s) you need ~1,000 — does the pool, the DB `max_connections`, the thread count support that? If latency rises under load (it usually does), W grows too and concurrency grows *faster* than linearly.

✅ For a change on a scaling-sensitive path:
- Estimate the resource at 10x with Little's Law (connections, memory, CPU, worker count).
- Identify the **first ceiling** you'll hit (DB connections? a single hot key §6? memory §4? a downstream's rate limit?). That ceiling, not your service, is the real capacity.
- **Load-test to the target**, not just to "it works" — drive it to 10x and watch where p99 cliffs (measuring-profiling §5, open-model load).

> Cost of skipping it: a feature that's fine at launch traffic and silently becomes the bottleneck at 3x — discovered during the traffic spike you scaled up *for*, when it's hardest to fix. The estimate is cheap; the outage isn't.

---

## 9. Autoscaling signals & cold start

Autoscaling only helps if it scales on the *right* signal and new instances become useful fast enough.

❌ Scale on CPU when the bottleneck is IO-bound (the box is at 30% CPU but maxed on connection waits — CPU-based autoscaling never triggers, and the service stays overloaded).
✅ Scale on the metric that actually saturates: queue depth / request concurrency / latency for IO-bound work; CPU for compute-bound. And mind **cold start** — if a new instance takes 60s to warm up (JIT, cache fill, connection pool), autoscaling reacts too slowly for a sharp spike. Pre-warm, keep a buffer, or use faster-starting runtimes for spiky traffic.

> Cost: autoscaling on the wrong signal means it doesn't kick in when you need it (overload persists) or thrashes (scale up/down churn). Slow cold starts mean the new capacity arrives after the spike already caused errors. Match the signal to the real bottleneck; account for warm-up time.

---

## 10. Load-test evidence before merge

For a change on a hot or scaling-sensitive path, "it works in dev" is not evidence it scales. The standard is a number under realistic load.

✅ Look for (or ask for) before merging a scaling-sensitive change:
- A **load test** at expected (and ideally 10x) traffic, open-model, reporting p95/p99 and error rate (measuring-profiling §5).
- A **soak test** for anything long-running, to rule out memory/FD growth (§4, §5).
- The identified **first ceiling** and headroom to it.

> Cost: merging a "should scale fine" change with no load evidence is how the capacity bug ships — and the first real load test becomes the production traffic spike. A pre-merge load test turns an outage into a code review comment.

---

## Quick scan checklist

- [ ] **Stateless** — no in-memory session/counter/rate-limit/local-disk state that breaks behind a load balancer; shared state externalized.
- [ ] **Backpressure & rate limiting** present — bounded queues/concurrency, load shed (`503`) instead of unbounded accept.
- [ ] **Queue depth** is bounded and monitored; consumers scale to peak arrival rate; a DLQ exists.
- [ ] **Memory stable** under sustained load — caches/memos bounded, listeners/timers/connections cleaned up; soak-tested.
- [ ] **FD/connection limits** respected — resources closed on every path, connections pooled, outbound concurrency bounded.
- [ ] No **single contention point** (global lock, hot row/key) serializing the whole service; sharded/batched/atomic.
- [ ] **Graceful degradation** — non-critical dependencies optional; load shedding under overload.
- [ ] **Capacity estimated** with Little's Law; the first ceiling identified; "holds 10x?" answered.
- [ ] **Autoscaling** keys on the real bottleneck signal; cold-start time accounted for.
- [ ] **Load/soak-test evidence** attached for scaling-sensitive changes, not just "works in dev."
