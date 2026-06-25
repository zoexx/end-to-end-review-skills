# Data access patterns

Deep-dive for reviewing how service code *talks to the data store* — the application-side patterns that cause slow endpoints, exhausted pools, and stale reads. Each section pairs a ❌ anti-pattern with a ✅ fix and names the incident it prevents.

**Scope boundary:** this guide is about *how the application accesses data*. Index design, query plans, schema constraints, lock modes, and migration safety belong to **database-review** — this guide cross-references it rather than duplicating it. If a finding is "this query needs an index" or "this migration locks the table," route it there.

---

## 1. N+1 queries across the boundary

One query to fetch a list, then one more query per row. Fine with 5 rows, catastrophic with 500 — and invisible in tests with tiny fixtures.

❌ Query per row:
```ts
const orders = await db.orders.findMany({ where: { userId } });   // 1 query
for (const o of orders) {
  o.customer = await db.customers.findUnique({ where: { id: o.customerId } }); // ❌ N queries
}
```
✅ Batch — join, `IN`, or the ORM's eager include:
```ts
const orders = await db.orders.findMany({
  where: { userId },
  include: { customer: true },          // single query with a join
});
// or hand-rolled: fetch all customerIds, one `WHERE id IN (...)`, then map in memory
```
> Incident: an endpoint that's snappy in dev (10 fixture rows) takes 8 seconds in prod (500 rows = 501 round trips). Each query is fast; the *round-trip count* is the killer, and it scales with data so it degrades silently over time.

How to spot it in review: any DB/HTTP call **inside a loop** (or inside `.map`/list comprehension) over a result set. That shape is N+1 until proven otherwise.

---

## 2. Unbounded SELECTs

Loading a whole table (or an unbounded slice) into memory.

❌ Fetch everything, filter in app:
```python
users = db.query("SELECT * FROM users").fetchall()   # ❌ loads the entire table
active = [u for u in users if u.active]               # ❌ filtering belongs in SQL
```
✅ Filter, project, and limit in the query:
```python
rows = db.query(
    "SELECT id, email FROM users WHERE active = true ORDER BY id LIMIT :n",
    {"n": page_size},
).fetchall()                                          # only what you need, bounded
```
> Incident: `SELECT *` on a table that grew to 40M rows pulls gigabytes into the app, spikes memory, OOMs the pod, and saturates the DB's network — all to then filter in Python. The DB exists to filter; let it.

Also flag `SELECT *` even when bounded — it ships columns you don't use (including large blobs/JSON), defeats covering indexes, and silently breaks when columns are added/reordered. Select the columns you actually read.

---

## 3. Missing pagination

The API-layer view of unbounded reads (see api-design §3): any list endpoint or batch read must page.

❌ No pagination on a growing collection:
```go
rows, _ := db.Query("SELECT * FROM events WHERE account_id = $1", acct) // ❌ could be millions
```
✅ Keyset/cursor pagination (stable under inserts, fast at depth):
```go
// WHERE account_id = $1 AND id > $2 ORDER BY id LIMIT $3
rows, _ := db.Query(
  "SELECT id, type, created_at FROM events WHERE account_id=$1 AND id>$2 ORDER BY id LIMIT $3",
  acct, afterID, limit)
```
> Incident: a busy account accumulates millions of events; the un-paginated query gets slower every day until it times out — and because it's keyed by account, only *some* customers hit it, making it hard to reproduce.

Prefer **keyset** over `OFFSET` for large/live data: deep `OFFSET` scans and discards all skipped rows (slow), and concurrent inserts shift the window (skipped/duplicated rows).

---

## 4. Transaction scope: too wide or too narrow

A transaction must wrap exactly the writes that have to commit-or-rollback together — no more, no less.

❌ Too wide — slow external call inside the transaction:
```python
with db.transaction():                 # transaction opens
    order = create_order(cart)
    charge = payment_api.charge(order)  # ❌ holds DB locks during a 2s external HTTP call
    mark_paid(order, charge.id)
```
✅ Keep external IO out of the transaction; commit the local invariant atomically:
```python
order = create_order(cart)             # committed in its own short tx
charge = payment_api.charge(order, idempotency_key=order.id)   # no DB lock held here
with db.transaction():
    mark_paid(order, charge.id)        # short tx: just the state transition
```
> Incident (too wide): holding row locks across a slow payment call means concurrent orders queue behind those locks; under load the DB fills with lock-waiters and throughput collapses. Long transactions also bloat MVCC/undo and block vacuum.

❌ Too narrow — related writes in separate transactions:
```python
with db.transaction(): debit(from_acct, amt)    # commits
with db.transaction(): credit(to_acct, amt)     # ❌ if this fails, money vanished
```
✅ One transaction for one invariant:
```python
with db.transaction():
    debit(from_acct, amt)
    credit(to_acct, amt)        # both commit or neither does
```
> Incident (too narrow): a crash between the two transactions debits one account without crediting the other — money disappears, and the books don't balance.

(Lock modes, isolation levels, and deadlock specifics → **database-review**.)

---

## 5. Lazy-load in a loop (ORM traps)

ORMs make a DB round trip look like a property access. A lazy relation touched in a loop is N+1 in disguise (§1), but it hides better because there's no visible query call.

❌ Lazy relation accessed per iteration:
```python
for order in orders:            # orders loaded
    print(order.customer.name)  # ❌ each .customer triggers a separate SELECT (lazy load)
```
✅ Eager-load up front:
```python
orders = (session.query(Order)
          .options(joinedload(Order.customer))   # one query, joined
          .all())
for order in orders:
    print(order.customer.name)  # already loaded, no extra query
```
> Incident: the same N+1 outage as §1, but harder to catch in review because `order.customer` looks like a field read, not a query. Treat every lazy relation access inside a loop as a query inside a loop.

Related traps: detached-entity access after the session closes (raises or silently re-queries), and serializing an entity to JSON that triggers lazy loads for every relation.

---

## 6. Write-then-read replica lag

In a primary/replica setup, a write goes to the primary but a subsequent read may hit a replica that hasn't caught up — so you read stale (or missing) data you just wrote.

❌ Write to primary, immediately read from replica:
```ts
await dbPrimary.users.update(id, { name });        // write
const user = await dbReplica.users.find(id);        // ❌ replica may lag → reads the OLD name
return user;
```
✅ Read-your-writes: read from primary right after a write (or wait for replication):
```ts
await dbPrimary.users.update(id, { name });
const user = await dbPrimary.users.find(id);         // read from primary for read-after-write
return user;                                         // replicas serve later/independent reads
```
> Incident: a user updates their profile, the success page reads from a lagging replica and shows the *old* name — they think the save failed and submit again, creating duplicate work and a support ticket. Classic, intermittent, and only happens under replication lag.

---

## 7. Caching: stampede, missing TTL, bad invalidation

Caching trades freshness for speed. The three classic failures: no TTL (stale forever), bad invalidation (stale despite a write), and stampede (all clients miss at once and hammer the DB).

❌ Naive cache with a stampede hole:
```ts
let v = await cache.get(key);
if (!v) {                                  // ❌ on expiry, every concurrent request misses...
  v = await db.expensiveQuery();           // ...and they ALL run the expensive query at once
  await cache.set(key, v);                 // ❌ no TTL — also stale forever if no invalidation
}
return v;
```
✅ TTL + single-flight (coalesce concurrent misses):
```ts
const v = await cache.getOrSet(key, async () => db.expensiveQuery(), {
  ttl: 60,                 // bounded staleness — never stale forever
  singleFlight: true,      // only ONE caller recomputes; others await its result
});
return v;
```
And invalidate on write so reads aren't stale before the TTL:
```ts
await db.products.update(id, patch);
await cache.del(`product:${id}`);          // explicit invalidation paired with the write
```
> Incident (stampede): a hot key expires; 5,000 in-flight requests all miss simultaneously and all run the same heavy query in the same instant; the DB CPU spikes to 100% and falls over — an outage caused by the cache *expiring*, not by a traffic increase.
> Incident (bad invalidation): a price changes in the DB but the cache isn't invalidated; customers see and check out at the old price until the TTL lapses — a revenue/consistency bug.

Cache invalidation is genuinely hard; favor short TTLs + explicit invalidation on write, and single-flight every hot key.

---

## 8. Connection pooling: missing, too small, too large

Opening a DB connection per request is expensive; not pooling, or mis-sizing the pool, is a top cause of "the DB is fine but the app is dying."

❌ A fresh connection per request (no pool):
```python
def handler():
    conn = psycopg2.connect(DSN)   # ❌ TCP + TLS + auth handshake on every request
    ...
    conn.close()
```
✅ A shared, sized pool:
```python
pool = ConnectionPool(DSN, min_size=5, max_size=20)   # reuse connections
def handler():
    with pool.connection() as conn:                   # borrow + auto-return
        ...
```
> Incident (no pool): under load, per-request connects saturate the DB's connection-handling and add tens of ms of handshake latency to every request; the DB hits `max_connections` and rejects new connections — an outage from connection churn alone.

Sizing:
- **Too large:** more app connections than the DB can serve well → the DB thrashes on context-switching/memory; total throughput *drops* as the pool grows. Pool size should respect the DB's `max_connections` across *all* app instances combined.
- **Too small:** requests queue waiting for a connection and time out while the DB sits idle (see concurrency-async §8).
- Don't hold a pooled connection across slow external IO (concurrency-async §8) — it pins a scarce resource.

(Where the DB's own `max_connections` / resource limits should sit → **database-review**.)

---

## 9. Reading more than you need (over-fetching rows & columns)

The row-level version of API over-fetching (api-design §7): pulling columns or rows the code never uses.

❌ Select-all then use one field:
```ts
const u = await db.query("SELECT * FROM users WHERE id = $1", [id]);  // ❌ pulls every column
return u.email;                                                      // used: one column
```
✅ Project exactly what you read:
```ts
const u = await db.query("SELECT email FROM users WHERE id = $1", [id]);
return u.email;
```
> Incident: `SELECT *` drags a large `profile_json`/`avatar_blob` column on a hot path, inflating every response's memory and network cost and defeating a covering index that would otherwise satisfy the query from the index alone. Selecting only `email` keeps it lean and index-friendly.

---

## 10. Double-writes without an outbox

Writing to the DB and then publishing an event (or calling another system) as two separate steps: if the second fails after the first commits, the two systems diverge with no record.

❌ DB write then publish — not atomic:
```ts
await db.orders.create(order);     // committed
await kafka.publish("order.created", order);  // ❌ if this throws, the event is lost forever
```
✅ Transactional outbox — write the event in the *same* transaction, relay it asynchronously:
```ts
await db.transaction(async (tx) => {
  await tx.orders.create(order);
  await tx.outbox.insert({ topic: "order.created", payload: order });  // same commit
});
// a separate relay polls the outbox and publishes, marking rows sent (at-least-once → idempotent consumers)
```
> Incident: the order commits but the `order.created` publish fails (broker blip); fulfillment never hears about the order, so it ships nothing — the DB says "order exists," the message bus says "never happened." The outbox makes the event durable with the write so it can't be lost; the relay guarantees it eventually publishes.

Pairs with idempotent consumers (concurrency-async §10), since outbox relay is at-least-once.

---

## Quick scan checklist

- [ ] No DB/HTTP call inside a loop over a result set (N+1) — batch via join/`IN`/eager include.
- [ ] No unbounded `SELECT`/`SELECT *`; filter, project, and `LIMIT` in the query.
- [ ] Every list/batch read is paginated, preferably keyset over `OFFSET`.
- [ ] Transaction scope wraps exactly one invariant — no external IO inside, no related writes split across txns.
- [ ] No lazy-loaded relation accessed inside a loop; eager-load where iterating.
- [ ] Read-after-write reads from the primary (or waits) to avoid replica-lag staleness.
- [ ] Caches have a TTL, explicit invalidation on write, and single-flight on hot keys.
- [ ] Connections are pooled and sized to the DB's combined limit; not held across slow IO.
- [ ] Queries select only the columns the code uses.
- [ ] DB-write-plus-publish goes through a transactional outbox, not two independent steps.
- [ ] Index/schema/migration/lock concerns routed to **database-review**.
