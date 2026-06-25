# Caching strategy

Deep-dive for reviewing a cache. Caching trades **freshness for speed**, and every cache bug is a variation on that trade going wrong: stale data the cache should have refreshed, a stampede when it expires, or a key so wrong it serves the wrong user's data. Each section pairs a ❌ anti-pattern with a ✅ fix and names the user-facing cost.

The reviewer's two questions for any new cache: **what invalidates it, and what happens when it expires under load?** If the PR can't answer both, it isn't done.

> Backend-review's `data-access-patterns.md §7` covers the app-layer TTL/invalidation/stampede basics; this guide goes wider — layers, key design, HTTP caching, hit-ratio targets, and when *not* to cache — and is the place to route cache *strategy* questions.

---

## 1. What to cache at which layer

Cache as close to the user as the freshness requirement allows — the closer the layer, the cheaper the hit, but the harder it is to invalidate.

| Layer | Caches | Invalidation | Best for |
| --- | --- | --- | --- |
| **Browser** (HTTP cache) | static assets, GET responses | `Cache-Control` max-age, ETag/304 | hashed static files, public GETs |
| **CDN / edge** | public responses, assets | purge API, surrogate keys, short TTL | global static + cacheable HTML/API |
| **App / in-memory** (per-process) | computed values, hot lookups | TTL, manual evict | per-instance hot data, tiny TTL |
| **Shared cache** (Redis/Memcached) | query results, sessions, rendered fragments | TTL + explicit del on write | cross-instance, the common case |
| **Database** (materialized view, query cache) | aggregates, denormalized rollups | refresh schedule / triggers | expensive aggregates |

❌ Caching mutable, user-specific data at a shared/public layer:
```
Cache-Control: public, max-age=3600    on  GET /account/balance   ← ❌ caches one user's balance for everyone
```
✅ Match the layer to the data's scope and volatility:
```
Cache-Control: private, no-store       on  /account/balance        // never shared, never cached
Cache-Control: public, max-age=31536000, immutable   on  /static/app.a1b2c3.js  // safe forever (hashed)
```

> Cost of the wrong layer: caching per-user data at a public/CDN layer leaks one user's data to another (a correctness + security bug); caching volatile data with a long TTL serves stale data widely. Pick the layer by *who can share it* and *how fresh it must be*.

---

## 2. Cache key design & cardinality

The key is the contract. Miss a dimension and you serve the wrong data; over-specify and you never get a hit.

❌ Key missing a dimension the response depends on:
```ts
cache.get(`product:${id}`)   // ❌ but the response varies by currency + locale + user tier
                             //    → a EUR user gets a USD-cached price
```
✅ Include every input the output depends on — and *only* those:
```ts
cache.get(`product:${id}:${currency}:${locale}:${tier}`)   // complete, correct
```
❌ Over-specific key → near-zero hit ratio:
```ts
cache.get(`search:${JSON.stringify(allParams)}:${timestamp}`)  // ❌ timestamp makes every key unique
```
✅ Normalize and drop irrelevant dimensions so keys actually collide:
```ts
cache.get(`search:${normalize(query)}:${page}`)   // stable, high hit ratio
```

> Cost (missing dimension): wrong-user / wrong-currency / wrong-locale responses — a data-correctness bug that's intermittent (only on a cache hit) and brutal to debug.
> Cost (too-high cardinality): a 0.5% hit ratio — you pay all the cache machinery and complexity for almost no benefit, and you may be *slower* than no cache (extra round trip to a cold cache, then the real query anyway).

Watch **unbounded key cardinality** (a key per request, per timestamp, per raw URL) — it bloats cache memory and evicts useful entries.

---

## 3. TTL & staleness tradeoffs

TTL is the explicit answer to "how stale is acceptable?" Two failure modes: TTL too long (stale data lingers) and **no TTL at all** (stale forever, plus an unbounded cache).

❌ No TTL:
```ts
cache.set(key, value);   // ❌ no expiry — stale until something explicitly deletes it (and if
                         //    the invalidation has a bug, stale forever; also grows unbounded)
```
✅ Always set a TTL as a backstop, even with explicit invalidation:
```ts
cache.set(key, value, { ttl: 300 });   // bounded staleness even if invalidation is missed
```
- Choose TTL by tolerance for staleness × cost to recompute: a rarely-changing config can be minutes; a price or inventory count, seconds.
- TTL is a *safety net* under explicit invalidation (§4), not a replacement — invalidation makes it fresh on write; TTL bounds the damage when invalidation is missed.
- Add **jitter** to TTLs so a cohort of keys set together doesn't all expire in the same instant (→ stampede, §5).

> Cost (no TTL): one missed invalidation → permanently stale data and a cache that only grows until it OOMs or evicts your hot keys. A TTL is cheap insurance against both.

---

## 4. Invalidation strategy

"There are only two hard things in computer science: cache invalidation and naming things." Pair every write with the invalidation it implies.

❌ Write the DB, forget the cache:
```ts
await db.products.update(id, { price });   // ❌ cache still serves the old price until TTL
```
✅ Invalidate (or update) on write:
```ts
await db.products.update(id, { price });
await cache.del(`product:${id}`);          // explicit invalidation paired with the write
// or write-through: update cache with the new value in the same step
```
Write patterns:
- **Cache-aside (lazy):** read → miss → load → populate; write → invalidate. Simple, the default. Risk: a window where DB is updated but cache isn't (use write-then-invalidate, and accept the small race or use versioning).
- **Write-through:** write goes to cache *and* store together — cache always fresh, write is slightly slower.
- **Write-back (write-behind):** write to cache, flush to store async — fastest writes, but data loss risk on crash before flush; rarely worth it outside specialized cases.

> Cost of missed invalidation: a price/permission/feature-flag changes in the DB but the cache serves the old value until the TTL lapses — customers check out at the old price, or a revoked permission still works. Intermittent (only on a hit) and high-stakes.

For multi-key/derived caches, track what to invalidate (tag/surrogate keys, or invalidate a version prefix) so one write clears all dependent entries.

---

## 5. Stampede / dogpile / thundering herd

The outage caused by the cache *expiring*, not by a traffic spike. A hot key expires; every concurrent request misses at once and they all run the expensive recompute simultaneously.

❌ Naive get-or-compute with a stampede hole:
```ts
let v = await cache.get(key);
if (!v) {                          // ❌ on expiry, all N concurrent requests see a miss...
  v = await db.expensiveQuery();   // ...and all N run the heavy query in the same instant
  await cache.set(key, v, { ttl: 60 });
}
return v;
```
✅ Pick a stampede defense:
```ts
// single-flight / request coalescing: only ONE caller recomputes; the rest await its result
const v = await cache.getOrSet(key, () => db.expensiveQuery(), { ttl: 60, singleFlight: true });
```
- **Single-flight / lock:** only one recompute runs; others wait for it (`golang.org/x/sync/singleflight`, a short-lived Redis lock).
- **Stale-while-revalidate (SWR):** serve the stale value immediately and refresh in the background — no request ever blocks on the recompute, no herd.
- **Jittered / staggered TTL:** randomize expiries so keys set together don't expire together.
- **Early/probabilistic recompute:** refresh a hot key *before* it expires (XFetch).

> Cost: a hot key expires and 5,000 in-flight requests all miss and all run the same heavy query in the same instant; DB CPU spikes to 100% and the service falls over — an outage triggered by an *expiry*, not a load increase, which makes it baffling in the postmortem.

---

## 6. Negative caching

Cache the *absence* of a result too, or every lookup for a missing/invalid key re-hits the backend.

❌ Only caching hits:
```ts
const u = await cache.get(`user:${id}`) ?? await db.users.find(id);  // miss for a non-existent id
// every request for the same missing id → a DB query, every time (and a vector for cache-busting abuse)
```
✅ Cache the "not found" with a short TTL:
```ts
let u = await cache.get(`user:${id}`);
if (u === undefined) {
  u = await db.users.find(id);
  await cache.set(`user:${id}`, u ?? NULL_SENTINEL, { ttl: u ? 300 : 30 });  // short TTL for misses
}
```

> Cost: a flood of requests for non-existent keys (a buggy client, or a deliberate cache-penetration attack) bypasses the cache entirely and hammers the DB — every one is a guaranteed miss. Negative caching (short TTL, so a later-created record appears soon) absorbs it.

---

## 7. Memoization scope & leaks

In-process memoization is a cache too — and an unbounded one is a memory leak.

❌ Unbounded module-level memo keyed by user input:
```js
const memo = new Map();
function render(userId, data) {
  if (!memo.has(userId)) memo.set(userId, expensive(data));  // ❌ grows forever, never evicts
  return memo.get(userId);
}
```
✅ Bound it (LRU + max size, or a TTL), and scope it correctly:
```js
import { LRUCache } from 'lru-cache';
const memo = new LRUCache({ max: 10_000, ttl: 60_000 });   // bounded — evicts, won't leak
```

> Cost: an unbounded memo keyed by something high-cardinality (user id, request body) grows until the process OOMs after hours/days of uptime — a leak that looks like a cache. Also watch **scope**: a memo on a long-lived singleton holding per-request data leaks request state across users (correctness + memory).

---

## 8. HTTP caching — Cache-Control, ETag, 304

For anything served over HTTP, the browser/CDN cache is free performance if you set the headers right.

❌ No caching headers (or caching the wrong things):
```
GET /static/app.js   → 200, no Cache-Control   ← re-downloaded every visit
GET /api/me          → public, max-age=600      ← ❌ caches a user-specific response publicly
```
✅ Long-cache immutable assets; revalidate dynamic ones; never share private data:
```
# hashed static asset — safe to cache forever
Cache-Control: public, max-age=31536000, immutable

# dynamic but cacheable — revalidate cheaply with ETag → 304 (no body re-sent)
Cache-Control: no-cache            # "revalidate every time" (not "don't cache")
ETag: "a1b2c3"                     # server returns 304 if unchanged

# user-specific — never cache
Cache-Control: private, no-store
```
- **`immutable` + content hash** on static assets eliminates revalidation entirely.
- **ETag / `If-None-Match` → 304** lets the server skip re-sending an unchanged body (saves bandwidth, not the round trip).
- `no-cache` means "revalidate," not "don't store"; `no-store` means "don't store at all." They're often confused.

> Cost of getting headers wrong: re-downloading hashed assets that never change (wasted bandwidth + slower loads), or caching a user-specific API response publicly (one user sees another's data — a correctness/security bug at the CDN).

---

## 9. Hit-ratio targets & when NOT to cache

A cache earns its complexity only if it hits often enough.

✅ Measure hit ratio; a cache below ~80–90% on a hot read often isn't paying for itself (and a low-hit cache adds a round trip + the real query — net *slower*). If the ratio is low, fix the key (§2) or reconsider caching at all.

**Don't cache when:**
- The data is cheap to compute or already fast (caching adds a round trip and an invalidation bug surface for no win).
- The data must be strongly consistent (balances, inventory you sell against, auth decisions) — a stale read is a correctness bug; cache invalidation can't guarantee freshness.
- The access pattern is low-reuse / high-cardinality — the hit ratio will be near zero (§2).
- Caching would mask a real problem (an unindexed query — fix the index, don't cache the slow query; database-review).

> Cost of over-caching: every cache is a correctness liability (stale-data bugs) and an operational one (invalidation, memory, stampede). Caching something that didn't need it adds all of that risk for no measurable speedup — and stale-data bugs are among the hardest to reproduce.

---

## Quick scan checklist

- [ ] Cached at the right **layer** for the data's scope (private vs public) and freshness need.
- [ ] Cache **key** includes every input the output depends on — and nothing high-cardinality that kills the hit ratio.
- [ ] Every cache entry has a **TTL** (bounded staleness even if invalidation is missed); TTLs are jittered.
- [ ] Writes **invalidate** (or update) the cache; the pattern (cache-aside / write-through) is deliberate.
- [ ] Hot keys are **stampede**-protected (single-flight / SWR / jittered TTL / early refresh).
- [ ] **Negative** results cached with a short TTL where misses are common or abusable.
- [ ] In-process memoization is **bounded** (LRU/size/TTL) and correctly scoped — no per-request data on a singleton.
- [ ] **HTTP** caching headers correct: `immutable` for hashed assets, ETag/304 for dynamic, `private/no-store` for user data.
- [ ] **Hit ratio** is (or will be) high enough to justify the cache; nothing strongly-consistent is cached.
- [ ] The PR answers *what invalidates this* and *what happens when it expires under load*.
