# Error handling & resilience

Deep-dive for reviewing how a service behaves when things go wrong — which is the only time resilience matters. Each section pairs a ❌ anti-pattern with a ✅ fix and names the incident it prevents.

The core principle: **an error is data. Either handle it or propagate it — never erase it.**

---

## 1. Swallowed exceptions

The single most expensive bug class. An error caught and dropped becomes a silent failure that surfaces hours later as corrupt data or a confused user.

❌ Empty / swallowing catch:
```ts
try {
  await chargeCustomer(order);
} catch (e) {
  // ❌ nothing — the charge failed and the order proceeds as paid
}
await markPaid(order);
```
✅ Handle or rethrow, never erase:
```ts
try {
  await chargeCustomer(order);
} catch (e) {
  log.error("charge failed", { orderId: order.id, err: e });
  throw new PaymentError("charge failed", { cause: e });  // stop the flow
}
await markPaid(order);   // only reached on success
```
> Incident: orders get marked paid without a successful charge; finance finds the gap in reconciliation a week later.

Variants to flag:
- `catch (e) {}` empty block.
- `except: pass` in Python.
- `if err != nil { }` in Go (assigned, never checked).
- `.catch(() => {})` on a promise.
- A bare `catch` that returns a default/`null` and lets the caller proceed as if nothing failed.

---

## 2. Catch-log-continue on a critical path

Logging is not handling. On a path where the failure *matters*, logging and continuing is just a swallowed error with a paper trail nobody reads.

❌ Log and march on:
```python
for item in cart.items:
    try:
        reserve_inventory(item)
    except Exception:
        log.warning("reserve failed", item=item)   # ❌ then we check out anyway
        continue
checkout(cart)   # some items were never reserved
```
✅ Decide: fail the operation, or explicitly degrade:
```python
failures = []
for item in cart.items:
    try:
        reserve_inventory(item)
    except InventoryError as e:
        failures.append((item, e))
if failures:
    raise PartialReservationError(failures)   # don't check out a half-reserved cart
checkout(cart)
```
> Incident: checkout succeeds with unreserved items; the warehouse can't fulfill and you oversell. Logging made it *look* handled in review.

Rule: catch-log-continue is fine for **best-effort, non-critical** work (a cache warm, an analytics ping). On a path where correctness depends on the call, it's a blocking finding.

---

## 3. Missing timeouts on network/IO

Every call that crosses a process boundary can hang forever. Without a timeout, one slow dependency exhausts your threads/connections and takes the whole service down.

❌ No timeout:
```go
resp, err := http.Get(url)   // ❌ default client has NO timeout — can hang forever
```
✅ Bounded:
```go
client := &http.Client{Timeout: 3 * time.Second}
ctx, cancel := context.WithTimeout(r.Context(), 3*time.Second)
defer cancel()
req, _ := http.NewRequestWithContext(ctx, "GET", url, nil)
resp, err := client.Do(req)
```
```ts
// Node fetch — pair with an AbortController; fetch has no default timeout
const ac = new AbortController();
const t = setTimeout(() => ac.abort(), 3000);
try { return await fetch(url, { signal: ac.signal }); }
finally { clearTimeout(t); }
```
> Incident: the payment provider degrades; with no timeout, every request thread blocks on it; the connection pool drains; your *unrelated* endpoints start failing too — a single slow dependency becomes a full outage.

Cover: HTTP clients, DB queries (statement timeout), gRPC deadlines, cache calls, queue publishes, lock acquisitions. The default for most clients is "wait forever" — assume no timeout unless you see one.

---

## 4. Retries without backoff/jitter, or on non-idempotent ops

Retries fix transient failures — and amplify outages if done wrong.

❌ Tight-loop retry, no backoff:
```ts
for (let i = 0; i < 5; i++) {
  try { return await call(); } catch { /* retry immediately */ }   // ❌
}
```
✅ Exponential backoff **with jitter**, capped, retry-eligible only:
```ts
async function withRetry<T>(fn: () => Promise<T>, max = 4): Promise<T> {
  for (let attempt = 0; ; attempt++) {
    try { return await fn(); }
    catch (e) {
      if (attempt >= max || !isRetryable(e)) throw e;   // fatal → don't retry
      const base = Math.min(1000 * 2 ** attempt, 8000);
      const jitter = Math.random() * base;              // decorrelate clients
      await sleep(base / 2 + jitter / 2);
    }
  }
}
```
> Incident (retry storm / thundering herd): a dependency blips; thousands of clients retry in lockstep with no jitter; the synchronized wave keeps it down. Jitter spreads the load so it can recover.

Two hard rules:
- **Only retry idempotent operations.** Retrying a non-idempotent `POST` (a charge, a send) without an idempotency key duplicates the side effect. See `api-design.md` §2.
- **Only retry retryable errors.** `429`/`503`/timeouts/connection-reset → retry. `400`/`401`/`422` → retrying is pointless and just hammers the dependency.

❌ Retrying a fatal error:
```python
@retry(times=5)
def f(): return client.post("/charge", json=body)  # ❌ retries a 400, and double-charges on a 504
```

---

## 5. No circuit breaker / bulkhead for flaky dependencies

When a dependency is *down* (not just slow), retrying every request wastes capacity and slows your own failure. A circuit breaker fails fast after a threshold; a bulkhead caps how many concurrent calls one dependency can consume.

❌ Hammer a dead dependency:
```ts
// every request waits the full 3s timeout, then retries, while the dep is hard-down
await withRetry(() => recommendationService.get(userId));
```
✅ Trip open, fail fast, degrade:
```ts
const breaker = new CircuitBreaker(recommendationService.get, {
  timeout: 3000, errorThresholdPercentage: 50, resetTimeout: 30000,
});
breaker.fallback(() => []);   // graceful degradation: empty recs, page still renders
const recs = await breaker.fire(userId);
```
> Incident: the recommendation service dies; without a breaker every page-load waits 3s × retries on a doomed call, so the *whole site* feels down even though recs are non-essential. The breaker turns "site down" into "no recs."

Bulkhead: cap concurrency per dependency (a semaphore / separate pool) so one slow dependency can't consume every worker — see `concurrency-async.md` §8.

---

## 6. Graceful degradation vs hard failure

Decide per dependency: is it **essential** (fail the request if it's down) or **enhancing** (degrade)?

❌ A non-essential call is on the critical path:
```python
profile = load_profile(uid)             # essential
badges  = badge_service.get(uid)        # ❌ enhancing, but an exception here 500s the whole page
return render(profile, badges)
```
✅ Isolate the enhancing call:
```python
profile = load_profile(uid)             # essential — let it raise
try:
    badges = badge_service.get(uid, timeout=0.5)
except Exception:
    badges = []                         # degrade: page renders without badges
    metrics.increment("badges.degraded")
return render(profile, badges)
```
> Incident: a cosmetic badge service outage takes down the entire profile page because its failure wasn't isolated from the essential render path.

---

## 7. Leaking stack traces / internal errors to clients

❌ Raw error to the client:
```ts
} catch (e) {
  res.status(500).send(e.stack);   // ❌ file paths, library versions, maybe a SQL fragment
}
```
✅ Log full internally, return safe + correlatable externally:
```ts
} catch (e) {
  const traceId = req.traceId;
  log.error("handler failed", { traceId, err: e });          // full detail in logs
  res.status(500).json({ error: { code: "INTERNAL", message: "unexpected error", trace_id: traceId } });
}
```
> Incident: a stack trace in the response hands an attacker your stack, paths, and a query shape; and gives clients a brittle, ever-changing error contract.

---

## 8. Error-as-control-flow

Exceptions are for the exceptional. Using them for expected outcomes is slow, hides intent, and makes real errors hard to spot.

❌ Exception as a normal branch:
```python
try:
    user = db.get(uid)
except NotFound:
    user = create_user(uid)   # ❌ "not found" is an expected, normal case
```
✅ Make the expected case explicit:
```python
user = db.find(uid)           # returns None / Optional, doesn't raise
if user is None:
    user = create_user(uid)
```
> Incident: not a crash, but: every "user doesn't exist yet" throws, fills the error logs, trips error-rate alerts, and buries the *real* exceptions in noise. (In Go, this is the inverse smell — returning sentinel errors for genuinely exceptional cases; match the language's idiom.)

---

## 9. Ignoring partial failures in fan-out

When you call N things in parallel, "they all succeeded" and "the first one failed" are different outcomes. Don't let one failure silently drop or one failure abort the rest unintentionally.

❌ `Promise.all` aborts on first rejection, losing the others' results:
```ts
const results = await Promise.all(ids.map(notify));   // ❌ one failure → whole batch rejects, no record of which sent
```
✅ Collect every outcome, then decide:
```ts
const settled = await Promise.allSettled(ids.map(notify));
const failed = settled.flatMap((r, i) => r.status === "rejected" ? [{ id: ids[i], reason: r.reason }] : []);
if (failed.length) {
  log.warn("partial notify failure", { failed });
  await deadLetter(failed);     // don't drop them — DLQ for retry/inspection
}
```
> Incident: a batch of 10,000 notifications, one fails, `Promise.all` rejects; the caller treats the whole batch as failed and *re-sends all 10,000* — or worse, treats it as done and silently drops the 9,999 that hadn't resolved yet.

Pair with **dead-letter handling**: messages/jobs that fail terminally go to a DLQ with their error and context, not `/dev/null`.

---

## 10. Resource leaks on the error path

The happy path closes the handle; the error path forgets to. Under load, leaks are what actually take you down.

❌ Leak on early return / exception:
```python
conn = pool.acquire()
rows = conn.query(sql)        # ❌ if this raises, conn is never released → pool exhausts
process(rows)
pool.release(conn)
```
✅ Release in `finally` / with-context / `defer`:
```python
with pool.acquire() as conn:          # released even on exception
    rows = conn.query(sql)
    process(rows)
```
```go
conn := pool.Acquire()
defer conn.Release()   // runs on every return path, including panics
```
> Incident: an intermittent query error leaks one connection each time; after a few hours the pool is empty and *every* request hangs waiting for a connection that will never come back.

Same pattern for files, locks, temp dirs, spans, and transactions — acquire and release must be paired across *all* exit paths.

---

## 11. Context / cancellation not propagated

When a client disconnects or a parent times out, the work it requested should stop — otherwise you do expensive work for a response nobody will read.

❌ Ignoring the request context:
```go
func handler(w http.ResponseWriter, r *http.Request) {
    rows := db.Query(context.Background(), bigQuery)  // ❌ won't cancel if the client leaves
}
```
✅ Thread and honor `ctx`:
```go
func handler(w http.ResponseWriter, r *http.Request) {
    rows, err := db.Query(r.Context(), bigQuery)      // cancels when the client disconnects
}
```
> Incident: clients time out and retry; each abandoned request keeps running a heavy query; the DB drowns in work for responses that were thrown away — a self-inflicted overload that snowballs.

In Node, propagate an `AbortSignal`; in Python async, let `CancelledError` propagate (don't swallow it); in Go, pass `ctx` down every call and check `ctx.Err()` in loops.

---

## Quick scan checklist

- [ ] No empty/swallowing catch; errors are handled or rethrown.
- [ ] No catch-log-continue on a path where the failure matters.
- [ ] Every network/IO call has an explicit timeout/deadline.
- [ ] Retries use exponential backoff + jitter, capped, and only on retryable + idempotent ops.
- [ ] Flaky/essential dependencies have a circuit breaker; non-essential ones degrade gracefully.
- [ ] No stack traces / internal errors leak to clients; full detail goes to logs with a trace id.
- [ ] Exceptions aren't used for expected control flow (or sentinels for genuine errors).
- [ ] Fan-out collects partial failures (`allSettled`/DLQ), doesn't drop or over-retry.
- [ ] Resources released on every exit path (`finally`/`defer`/with-context).
- [ ] Request context/cancellation propagated to downstream calls.
