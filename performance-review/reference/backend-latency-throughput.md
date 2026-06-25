# Backend latency & throughput

Deep-dive for making a request *fast* and the service *high-throughput* — the server-side patterns that inflate p95 and cap how many requests a box can serve. Each section pairs a ❌ anti-pattern with a ✅ fix and names the user-facing cost.

**Scope boundary:** this guide is the *performance lens* on backend code. The mechanics of N+1, event-loop blocking, and concurrency *correctness* live in **backend-review** (`concurrency-async.md`, `data-access-patterns.md`); query plans and indexes live in **database-review**. This guide flags those for their latency cost and prioritizes them — it cross-references the depth rather than duplicating it.

---

## 1. Sequential awaits that should be parallel

The single most common easy win. Independent IO awaited one-at-a-time pays the sum of the latencies when it could pay the max.

❌ Sequential — latencies add up:
```ts
const user    = await getUser(id);       // 80ms
const billing = await getBilling(id);    // 90ms   ← doesn't need `user`
const usage   = await getUsage(id);      // 85ms   ← doesn't need either
// total ≈ 255ms
```
✅ Parallel — independent calls fire together:
```ts
const [user, billing, usage] = await Promise.all([
  getUser(id), getBilling(id), getUsage(id),
]);                                       // total ≈ max(80,90,85) = 90ms
```
Go: `errgroup.Group`; Python: `asyncio.gather`; Java: `CompletableFuture.allOf` / structured concurrency.

> Cost: a page-load endpoint that takes 255ms instead of 90ms — 165ms of pure latency reclaimed by deleting two `await` keywords. Multiply across every such endpoint and it's a sitewide slowdown.

**How to spot it in review:** consecutive `await`s where a later one doesn't reference an earlier one's result. That's parallelizable until proven otherwise. (Keep them sequential only when there's a true data dependency or you must limit concurrency — then cap it, don't serialize.)

---

## 2. N+1 and chatty round trips

The latency-lens view of N+1 (mechanics → backend-review §data-access, database-review): the killer isn't per-query time, it's the *round-trip count*, and it scales with data so it degrades silently.

❌ One query per row:
```ts
const orders = await db.orders.findMany({ where: { userId } });  // 1 round trip
for (const o of orders) {
  o.customer = await db.customers.find(o.customerId);            // ❌ N round trips
}
```
✅ Batch into one round trip — join, `IN`, or DataLoader:
```ts
const orders = await db.orders.findMany({ where: { userId }, include: { customer: true } });
```

> Cost: 1ms per query × 500 rows = a 501-round-trip endpoint that's snappy on 10 dev fixtures and 600ms+ in prod — and every network hop's latency floor (often 0.5–2ms each, more cross-AZ) multiplies.

**Batching/DataLoader** collapses per-item lookups within a tick into one query: in GraphQL/resolver code, an un-batched field resolver is N+1 by construction — look for a DataLoader (or equivalent) wrapping every relation. The same applies to chatty internal RPC: prefer one `getUsers([ids])` over N `getUser(id)`.

---

## 3. Blocking the event loop / starving the thread

The latency-lens view (correctness depth → backend-review §concurrency): on a single-threaded runtime (Node, a Python async worker), any synchronous CPU or blocking IO freezes *every* concurrent request, not just the slow one.

❌ Sync CPU on the event loop:
```js
app.get('/report', (req, res) => {
  const csv = parseHugeCsvSync(req.body);   // ❌ 400ms of sync work
  const hash = crypto.pbkdf2Sync(pw, salt, 600000, 64, 'sha512'); // ❌ blocks everyone
  res.send(render(csv));
});
```
✅ Offload CPU (worker thread / worker pool / native async), use async crypto:
```js
const csv = await runOnWorker(parseHugeCsv, req.body);
const hash = await promisify(crypto.pbkdf2)(pw, salt, 600000, 64, 'sha512');
```

> Cost: a single 400ms sync handler doesn't slow one request — it adds up to 400ms to *every* request queued behind it on that event loop. Throughput collapses under concurrency even though each request's "own" work is small. (Profile with `clinic doctor` / event-loop lag metrics.)

---

## 4. Payload size & compression

Bytes on the wire are latency, especially on mobile and cross-region. Two levers: send fewer bytes, and compress the ones you send.

❌ Large, uncompressed, over-fetched response:
```ts
res.json(await db.users.findMany());   // ❌ every column incl. big JSON blobs, no compression,
                                       //    no pagination — could be megabytes
```
✅ Project, paginate, compress:
```ts
app.use(compression());                                    // gzip/br on responses
const users = await db.users.findMany({
  select: { id: true, name: true, email: true },           // only fields the client uses
  take: pageSize, cursor,                                   // paginated (keyset)
});
res.json(users);
```
- **Compression:** enable gzip or **Brotli** (`br`, better ratio for text) at the app or edge for responses over ~1KB. JSON compresses 5–10x. (Don't compress already-compressed bytes like images.)
- **Projection:** select only fields the client reads — over-fetching columns inflates serialization + network (data-access depth → backend-review §9).
- **Pagination:** never return an unbounded collection; prefer keyset over deep `OFFSET`.

> Cost: a 2MB uncompressed JSON list endpoint vs a 60KB compressed, paginated, projected one — seconds of transfer on a 3G connection, and serialization CPU on the server for every request.

---

## 5. Connection pooling & keep-alive

Establishing a connection (TCP + TLS handshake) is expensive; doing it per request adds fixed latency and exhausts limits.

❌ New connection per request / no keep-alive on the outbound client:
```python
def handler():
    conn = psycopg2.connect(DSN)   # ❌ handshake every request
    ...
# or an HTTP client with no connection reuse → new TCP+TLS per call
```
✅ Pool DB connections; reuse HTTP connections with keep-alive:
```python
pool = ConnectionPool(DSN, min_size=5, max_size=20)   # reuse
# outbound HTTP: a persistent session/agent with keep-alive
session = requests.Session()        # reuses connections across calls
```
- **Pool sizing** (depth → backend-review §8, database-review for the DB's own limit): too small → requests queue for a connection and time out while the DB is idle; too large → the DB thrashes and total throughput *drops*. Size to the DB's `max_connections` across *all* instances combined.
- **Keep-alive** on outbound HTTP (a shared client/agent, `httpx`/`requests.Session`, Go's default `http.Client` with a reused transport) avoids a TCP+TLS handshake on every downstream call — often 1–3 RTTs saved per request.

> Cost: handshake latency (tens of ms with TLS) added to every request, plus hitting `max_connections` and rejecting traffic under load — an outage from connection churn alone.

---

## 6. Timeout budgets & tail amplification in fan-out

Every downstream call needs a timeout, and the timeouts must fit inside the parent request's budget — otherwise one slow dependency hangs the whole request and amplifies into the tail (measuring-profiling §2).

❌ No timeout, or a child timeout longer than the parent:
```go
// parent has a 1s SLA, but this call can hang for 30s (client default)
resp, _ := http.Get(downstreamURL)    // ❌ no timeout → request hangs on a slow dep
```
✅ Per-call timeout within a propagated deadline; cap the fan-out:
```go
ctx, cancel := context.WithTimeout(ctx, 300*time.Millisecond)  // fits the budget
defer cancel()
resp, err := httpClient.Do(req.WithContext(ctx))               // bounded
```
- **Deadline propagation:** pass the request's remaining budget down (`context.Context`, `Deadline` header) so a child knows how long it's allowed — don't let a leaf call spend 30s inside a 1s request.
- **Budget math:** if the request SLA is 500ms and it makes 3 serial calls, each can't have a 500ms timeout. Apportion (e.g. 150ms each) so a single slow call fails fast instead of blowing the whole budget.
- **Fan-out:** for parallel fan-out, the parent waits for the slowest child — short per-child timeouts + partial-result degradation keep the tail bounded.

> Cost: a missing timeout turns one slow/hung downstream into a pile of stuck requests holding threads/connections; the slowness cascades upstream and the whole service's p99 spikes — a classic cascading-failure trigger.

---

## 7. Per-request work that should be precomputed or cached

The cheapest request is the one that doesn't redo work that didn't change.

❌ Recompute an unchanging-ish value on every request:
```ts
app.get('/pricing', (req, res) => {
  const rates = loadAndParseRateTableFromDisk();   // ❌ same result for an hour, parsed per request
  const config = compileTemplate(rawTemplate);     // ❌ recompiled every call
  res.json(price(req.query, rates, config));
});
```
✅ Hoist invariant work out of the request path (module load / memoize / cache):
```ts
const rates  = loadAndParseRateTableFromDisk();    // once at startup (refresh on TTL/signal)
const tmpl   = compileTemplate(rawTemplate);       // compiled once
app.get('/pricing', (req, res) => res.json(price(req.query, rates, tmpl)));
```

> Cost: every request pays for parsing/compiling/aggregating that only changes hourly — pure waste that scales linearly with traffic. Move it to startup, a memoized value, or a cache with a TTL (caching-strategy). For per-user-but-stable derived data, cache it keyed by user.

Also flag **serialization cost** done repeatedly: re-`JSON.stringify`ing the same large object per request, re-marshaling protobufs — cache the serialized bytes if the input is stable.

---

## 8. Slow middleware on the hot path

Middleware runs on *every* request, so a slow one is a tax on the whole service — and it's easy to miss because it's not in the handler.

❌ Expensive work in a global middleware:
```ts
app.use(async (req, res, next) => {
  req.user = await db.users.find(decode(req.token));  // ❌ a DB round trip on every request,
  await auditLog.writeSync(req);                       //    incl. ones that don't need the user,
  next();                                              //    plus a sync audit write in-band
});
```
✅ Make per-request middleware cheap; defer/async the rest:
```ts
app.use(verifyJwtSignatureOnly);                       // local crypto, no DB
// load the user lazily only on routes that need it; fire audit async / to a queue
```
- Avoid a DB/network round trip in global middleware unless every route truly needs it; do it lazily per route.
- Don't do blocking/sync IO (audit writes, logging to disk synchronously) in-band — push to a queue/async sink.
- Order middleware so cheap rejects (rate limit, auth signature) run before expensive work.

> Cost: a 20ms DB lookup in global auth middleware adds 20ms to *every* endpoint, including health checks and cached routes — a flat latency tax and extra DB load proportional to total traffic.

---

## 9. Serialization & copying in the hot path

Marshaling, deep-cloning, and re-allocating large structures per request burns CPU and GC.

❌ Deep-clone / re-serialize large objects per request:
```js
const safe = JSON.parse(JSON.stringify(bigConfig));  // ❌ full serialize+parse to "clone"
return template.render(safe);
```
✅ Avoid the round-trip clone; reuse immutable shared data; stream large outputs:
```js
return template.render(bigConfig);   // if it's read-only, don't clone it at all
// for big responses, stream (res.write / chunked) instead of building one giant string in memory
```
- `JSON.parse(JSON.stringify(x))` as a clone is a CPU + allocation hotspot on large objects — use structured cloning or, better, don't clone read-only data.
- **Streaming** large responses (NDJSON, chunked) avoids buffering the whole payload in memory and lets bytes start flowing sooner (lower TTFB).
- Watch allocations in hot loops (string concat in a loop, building arrays you immediately discard) — they show up as GC time in the flame graph.

> Cost: serialization and cloning frequently turn out to be the *widest frame* in a backend flame graph — the work nobody suspected, eating CPU that caps throughput.

---

## Quick scan checklist

- [ ] Independent awaits run in parallel (`Promise.all`/`errgroup`/`gather`), not serially.
- [ ] No N+1 / chatty round trips — batched via join/`IN`/DataLoader; one RPC over N. (mechanics → backend/database-review)
- [ ] No sync CPU or blocking IO on a single-threaded event loop; heavy work offloaded.
- [ ] Responses compressed (gzip/br), paginated, and projected to fields used.
- [ ] DB connections pooled and right-sized; outbound HTTP reuses connections (keep-alive).
- [ ] Every downstream call has a timeout that fits the parent's deadline budget; deadlines propagate.
- [ ] Invariant/per-user-stable work is precomputed at startup or cached, not redone per request.
- [ ] Global middleware is cheap — no per-request DB round trip or in-band blocking IO.
- [ ] No needless deep-clone/re-serialize of large objects on the hot path; large outputs streamed.
- [ ] Latency claims backed by a p95/p99 number (measuring-profiling), not a guess.
