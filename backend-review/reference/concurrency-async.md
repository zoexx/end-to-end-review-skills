# Concurrency & async

Deep-dive for reviewing code that runs more than one thing at a time — threads, goroutines, async tasks, queue consumers, and anything touching shared state. These bugs are the worst kind: intermittent, load-dependent, and invisible in a single-threaded test. Each section pairs a ❌ anti-pattern with a ✅ fix and names the incident it prevents.

The mental model: **between any two lines you write, another execution can run.** Review every shared read/write as if it will be interleaved, because under load it will be.

---

## 1. Check-then-act races

The classic. You check a condition, then act on it — but the world changed in between.

❌ Check then act:
```ts
if (!(await db.users.exists(email))) {     // T1 and T2 both see "no"
  await db.users.create({ email });        // ❌ both create → duplicate users
}
```
✅ Make it atomic in the database (let the constraint arbitrate):
```ts
try {
  await db.users.create({ email });        // UNIQUE(email) enforces it
} catch (e) {
  if (isUniqueViolation(e)) return existingUser(email);
  throw e;
}
```
> Incident: two near-simultaneous signups with the same email both pass the existence check and both insert; you now have duplicate accounts and downstream joins break. The DB unique constraint is the only reliable arbiter — application-level checks always have a window.

---

## 2. Non-atomic read-modify-write / lost updates

Read a value, change it in app memory, write it back — and concurrently another writer's update vanishes.

❌ Read, modify, write in the app:
```python
stock = db.get_stock(sku)        # both read 10
db.set_stock(sku, stock - 1)     # ❌ both write 9 — should be 8 (lost update)
```
✅ Atomic update in the store, or optimistic concurrency:
```sql
-- atomic decrement, guarded so it can't go negative
UPDATE inventory SET stock = stock - 1
WHERE sku = :sku AND stock >= 1;
-- rows-affected == 0 means it was out of stock; handle that
```
```python
# optimistic: version column, retry on conflict
UPDATE doc SET body=:body, version=version+1 WHERE id=:id AND version=:expected
# if 0 rows updated, someone else won — reload and retry
```
> Incident: overselling — two orders each decrement stock from 10, both write 9; you sold two units but only decremented one, and eventually ship inventory you don't have.

---

## 3. Unawaited async work / fire-and-forget without error capture

An async call you don't await may not finish before the response, and its errors vanish into an unhandled rejection.

❌ Dropped promise:
```ts
app.post("/orders", async (req, res) => {
  const order = await create(req.body);
  sendConfirmationEmail(order);   // ❌ not awaited — may not run; if it throws, silent
  res.status(201).json(order);    // response sent before email resolves
});
```
✅ Await it, or hand it to a durable queue (not the request lifecycle):
```ts
const order = await create(req.body);
await emailQueue.enqueue({ type: "confirmation", orderId: order.id }); // durable, retried
res.status(201).json(order);
```
> Incident: in serverless/edge runtimes the process freezes after the response, so the un-awaited email *never sends*; and because the rejection is unhandled, no log, no alert — confirmations silently stop.

If you genuinely want background work, make it explicit and capture errors:
```ts
void doBackground().catch(err => log.error("bg task failed", { err }));  // intentional, observable
```

---

## 4. Blocking the event loop (Node / single-threaded async)

In a single-threaded runtime, a synchronous CPU/IO chunk stalls *every* concurrent request.

❌ Sync work on the hot path:
```ts
app.post("/verify", (req, res) => {
  const hash = crypto.pbkdf2Sync(pw, salt, 600000, 64, "sha512"); // ❌ blocks the loop for ~100ms
  res.json({ ok: verify(hash) });        // every other request is frozen meanwhile
});
const data = fs.readFileSync(path);      // ❌ sync IO in a request handler
const rows = JSON.parse(hugeString);     // ❌ a 50MB parse blocks too
```
✅ Async / offload:
```ts
const hash = await promisify(crypto.pbkdf2)(pw, salt, 600000, 64, "sha512"); // libuv threadpool
const data = await fs.promises.readFile(path);
// for heavy CPU, use a worker_thread or move it out of the request path
```
> Incident: under concurrency, a sync 100ms hash on every login request serializes the whole server — p99 latency for *unrelated* endpoints spikes to seconds because they're all queued behind the blocking call.

Smells to grep for: `*Sync(` (`readFileSync`, `pbkdf2Sync`), large synchronous loops, big `JSON.parse`, regex with catastrophic backtracking on user input.

---

## 5. `Promise.all` vs `allSettled` for partial failure

`Promise.all` rejects on the *first* failure and abandons the rest. Choose deliberately. (Also covered in error-handling §9 — here the concern is the concurrency semantics.)

❌ One failure aborts the batch:
```ts
const results = await Promise.all(jobs.map(run));   // ❌ first reject discards the rest's outcomes
```
✅ When you need every outcome:
```ts
const settled = await Promise.allSettled(jobs.map(run));
// inspect each: fulfilled vs rejected, route failures to retry/DLQ
```
Use `Promise.all` only when **all-or-nothing** is truly the semantic (and you're fine aborting on first failure). Cap concurrency for large fan-outs so you don't open 10,000 sockets at once:
```ts
import pLimit from "p-limit";
const limit = pLimit(20);
await Promise.allSettled(items.map(i => limit(() => run(i))));  // at most 20 in flight
```
> Incident: an unbounded `Promise.all` over 50k items opens 50k DB connections/sockets simultaneously and exhausts the pool / hits the OS fd limit.

---

## 6. Goroutine / task leaks

A spawned task that never terminates (blocked on a channel/lock, or with no cancellation) leaks memory and eventually the runtime.

❌ Goroutine with no exit:
```go
go func() {
    for v := range ch {   // ❌ if nothing ever closes ch and no ctx, this lives forever
        process(v)
    }
}()
```
✅ Always give a task a way to stop:
```go
go func() {
    for {
        select {
        case <-ctx.Done():        // cancellation path
            return
        case v, ok := <-ch:
            if !ok { return }     // channel closed
            process(v)
        }
    }
}()
```
> Incident: each request spawns a worker goroutine that blocks forever on an unclosed channel; goroutine count climbs steadily, memory grows, and the service OOMs after a few days — a classic slow leak that survives every short test.

Same idea in Python `asyncio` (`task` references must be held and awaited/cancelled — a lost task reference can be GC'd mid-flight or leak) and in any thread you `start()` but never `join()`.

---

## 7. Deadlocks from lock ordering

Two locks acquired in opposite orders by two paths → each waits on the other forever.

❌ Inconsistent lock order:
```java
// path A: lock(accountX); lock(accountY)
// path B: lock(accountY); lock(accountX)   // ❌ A and B can deadlock on a transfer X↔Y
```
✅ Always acquire in a canonical order:
```java
Account first  = a.id < b.id ? a : b;       // deterministic ordering
Account second = a.id < b.id ? b : a;
synchronized (first) { synchronized (second) { transfer(a, b); } }
```
> Incident: two simultaneous transfers between the same pair of accounts in opposite directions deadlock; both threads hang, the connection/thread pool fills with stuck workers, and the service wedges.

Also: minimize lock scope, never do IO/network calls while holding a lock, and prefer DB-level locking/atomic ops over app-level locks when the state lives in the DB.

---

## 8. Connection / thread pool exhaustion

Pools are finite. Hold a connection too long, or open more concurrent work than the pool, and everything queues or fails.

❌ Holding a connection across a slow external call:
```ts
const conn = await pool.connect();
const user = await conn.query("SELECT ...");
const enriched = await externalApi.fetch(user);  // ❌ a DB connection pinned during a 2s HTTP call
const result = await conn.query("UPDATE ...");
conn.release();
```
✅ Don't hold scarce resources across slow IO:
```ts
const user = await db.query("SELECT ...");        // acquire+release per query
const enriched = await externalApi.fetch(user);   // no DB connection held here
await db.query("UPDATE ...", enriched);
```
> Incident: under load every request pins a DB connection for the duration of a slow third-party call; the pool (say 20 connections) drains; new requests block waiting for a connection and time out — the DB is idle but the app is "out of connections."

Right-size pools to the DB's max connections and your concurrency; an over-large pool can overwhelm the DB, an under-large one serializes you. See `data-access-patterns.md` §8.

---

## 9. Shared mutable singletons

A module-level mutable object shared across requests is a race waiting to happen.

❌ Mutable shared state across requests:
```ts
let currentUser;                       // ❌ module-scope, shared by all concurrent requests
app.use((req) => { currentUser = req.user; });   // request B overwrites A's user
function audit() { log(currentUser.id); }        // logs the wrong user under concurrency
```
✅ Keep per-request state per-request (context/locals, not module scope):
```ts
app.use((req, res, next) => { req.context = { user: req.user }; next(); });
function audit(req) { log(req.context.user.id); }
// or AsyncLocalStorage / context.Context / contextvars for implicit propagation
```
> Incident: under concurrent load, request A's audit log records request B's user id — a correctness *and* compliance bug, and nearly impossible to reproduce locally because it only happens when two requests interleave.

Stateless handlers + explicitly threaded context is the safe default. Caches/counters that must be shared need atomic operations or a lock.

---

## 10. Double-processing under at-least-once delivery

Most queues (SQS, Kafka, Pub/Sub) deliver **at least once** — the same message can arrive twice (redelivery after a missed ack, a rebalance, a retry). Consumers must be idempotent.

❌ Non-idempotent consumer:
```python
def handle(msg):
    charge_card(msg.order_id, msg.amount)   # ❌ redelivery → double charge
    db.mark_shipped(msg.order_id)
```
✅ Dedupe on a stable id (idempotent processing):
```python
def handle(msg):
    if db.already_processed(msg.id):        # idempotency table keyed by message id
        return ack(msg)
    with db.transaction():
        charge_card(msg.order_id, msg.amount)
        db.mark_shipped(msg.order_id)
        db.record_processed(msg.id)         # commit dedupe marker with the work
    ack(msg)
```
> Incident: a consumer crashes after charging but before acking; the broker redelivers; the retry charges the card again — a duplicate charge caused purely by normal at-least-once semantics, not a code bug in the charge itself.

Related: if you need **ordering**, confirm the queue/partition actually guarantees it (most don't across partitions); if you need **exactly-once**, you almost never get it from the broker — you build it with idempotent consumers + dedupe, exactly as above.

---

## Quick scan checklist

- [ ] No check-then-act races; uniqueness/atomicity enforced by the store, not app logic.
- [ ] Read-modify-write is atomic (DB-side) or guarded by optimistic versioning.
- [ ] No unawaited promises / fire-and-forget without explicit error capture.
- [ ] No sync CPU/IO (`*Sync`, big parse, blocking loop) on a single-threaded event loop.
- [ ] `Promise.all` vs `allSettled` chosen deliberately; large fan-outs are concurrency-capped.
- [ ] Spawned goroutines/tasks have a cancellation/exit path; none leak.
- [ ] Multiple locks acquired in a consistent order; no IO while holding a lock.
- [ ] Scarce connections/threads aren't held across slow external calls; pools right-sized.
- [ ] No shared mutable singleton holding per-request state; context threaded per request.
- [ ] Queue consumers are idempotent (dedupe on message id) for at-least-once delivery.
