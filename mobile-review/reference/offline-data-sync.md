# Offline, data & sync review reference

Deep-dive for the data-layer failures that only happen on a real network: the
write lost when the train enters a tunnel, the duplicate charge from a replayed
mutation, the corrupted local DB after an app update, the "your changes" that
silently lost to someone else's. Each section pairs a ❌ anti-pattern with a ✅
fix and names the data-loss or corruption it prevents. Cross-platform — RN
(WatermelonDB/SQLite/MMKV), iOS (Core Data/GRDB/SQLite), Android (Room/SQLite).

The mental model: **the network will fail mid-operation and the OS will kill your
app mid-operation — assume both, on every write.** Durable-first, network-second.

---

## 1. Offline-first reads and writes

❌ A screen that calls the network directly and dead-ends on failure:
```
onLoad { items = api.list() }      // throws/blank screen with no connection
onSave { api.create(item) }        // lost if offline
```
✅ Read from local store (single source of truth), write locally + enqueue sync:
```
onLoad { items = db.list() }                    // always works; sync updates it
onSave { db.insert(item); queue.enqueue(SyncCreate(item)) }  // never blocks the user
```
> Consequence: a network-coupled UI is unusable on the subway, in an elevator, on a plane. Offline-first means the local DB is the source of truth and the network is a background reconciler — the user never waits on or loses to connectivity.

---

## 2. Local DB choice & migrations

❌ Changing a schema (new column, new table, type change) without a migration:
```
// Room: bump version but ship no Migration → IllegalStateException, or
// fallbackToDestructiveMigration() → the user's data is WIPED on update
```
✅ Ship an explicit migration for every schema change; test upgrading from the *oldest supported* installed version:
```kotlin
Room.databaseBuilder(...)
  .addMigrations(MIGRATION_3_4)   // ALTER TABLE ... ; never destructive in prod
  .build()
```
> Consequence: a missing migration crashes on launch for everyone who updates (the DB version mismatches), and `fallbackToDestructiveMigration` silently deletes their data. Migrations must be forward-only and tested across version jumps, since users skip versions.

- Pick the store for the access pattern: a key-value store (MMKV/DataStore/`UserDefaults`) for small prefs; SQLite/Room/Core Data/WatermelonDB for relational/large/queried data.
- Encrypt the DB or the sensitive columns if it holds personal/financial data (SQLCipher / Core Data encryption / EncryptedSharedPreferences-backed keys).
> Consequence: storing large/queried data in a KV blob means loading and rewriting the whole blob per change (slow, racy); unencrypted sensitive data at rest is readable from a backup or rooted device.

---

## 3. Write queue + retry with backoff

❌ Firing the mutation and retrying in memory only:
```
async function save(x) {
  for (let i = 0; i < 3; i++) { try { await api.save(x); return } catch {} }
} // all 3 tries lost if the app is killed; tight retry loop hammers a down server
```
✅ Persist the intent to a durable queue *before* the network call, drain with exponential backoff + jitter, and resume on next launch / connectivity:
```
db.queue.insert({ op: 'save', payload: x, key: idempotencyKey })  // durable first
drain()   // pops, calls api, deletes on success; backoff+jitter on failure
```
> Consequence: an in-memory retry vanishes when the OS kills the app (and it will, under memory pressure or when swiped away) — the write is silently lost. A persisted queue survives the kill and finishes the write on next launch. Tight no-backoff retries also DDoS a recovering backend; back off with jitter.

---

## 4. Idempotency for replayed mutations

❌ A retried/replayed POST with no dedupe key:
```
await api.charge(amount)   // retry after a timeout that actually succeeded → double charge
```
✅ Attach a stable idempotency key generated once per logical operation and reused across retries:
```
const key = op.idempotencyKey ?? uuid()    // generated when the op is created, persisted
await api.charge(amount, { idempotencyKey: key })  // server collapses duplicates
```
> Consequence: networks make "did it succeed?" ambiguous — a request can succeed while the response is lost, so the client retries and the action happens twice (double charge, duplicate order, double message). The idempotency key lets the server dedupe. (Server-side idempotency enforcement → **backend-review**.)

---

## 5. Conflict resolution — make it explicit

❌ Last-write-wins by accident — whoever syncs last silently overwrites:
```
api.update(doc)   // clobbers any change made on another device since you read
```
✅ Choose a strategy on purpose and detect conflicts:
- **Server-authoritative** — send a base version/`updatedAt`; server rejects a stale write (`409`), client reconciles.
- **Last-write-wins** — acceptable only for low-stakes, single-field data; document it.
- **Merge / CRDT** — for collaborative or additive data where both edits must survive.
```
api.update(doc, { ifMatchVersion: doc.version })  // 409 → reload, merge, retry
```
> Consequence: accidental last-write-wins means a user's edits vanish with no error when two devices (or two tabs) touch the same record — the worst kind of data loss because it's silent. Pick the model the data deserves and surface conflicts.

---

## 6. Optimistic updates + rollback

❌ Optimistically updating the UI/local store and not undoing it when the sync fails:
```
db.like(postId)            // UI shows liked
api.like(postId)           // fails → UI still shows liked, but server disagrees
```
✅ Apply optimistically, then roll back (or reconcile) on failure:
```
const prev = db.snapshot(postId)
db.like(postId)
try { await api.like(postId) }
catch { db.restore(prev); toast('Couldn\'t save — try again') }  // honest rollback
```
> Consequence: an un-rolled-back optimistic update leaves the UI lying — it shows a state the server never accepted, so the next refresh "loses" the action and the user is confused or acts on false data.

---

## 7. Partial sync & pagination

❌ Syncing "everything" on launch, or re-pulling the whole dataset each time.
✅ Sync deltas with a cursor/`since` token, paginate, and persist progress so an interrupted sync resumes:
```
let cursor = db.syncCursor()
do { val page = api.changes(since = cursor); db.apply(page); cursor = page.next }
while (page.hasMore)   // resumable; survives a mid-sync kill
db.setSyncCursor(cursor)
```
> Consequence: a full re-pull is slow, data-hungry, and likely to be interrupted (and then restart from zero). Delta sync with a persisted cursor is fast and resumable. (Pagination/cursor semantics on the server → **backend-review**.)

---

## 8. Detect connectivity — and don't trust "connected"

❌ Gating sync purely on a boolean from `NetInfo`/`NWPathMonitor`/`ConnectivityManager`.
✅ Treat connectivity events as *hints* to attempt a drain; rely on the request actually succeeding, and handle captive portals / connected-but-no-internet:
```
onConnectivityRestored { drainQueue() }   // attempt — but success is decided by the response
```
> Consequence: "connected" doesn't mean "the internet works" (captive portal, no DNS, server down). Code that assumes the boolean is truth marks writes as sent when they weren't. The queue + idempotency model tolerates this; a fire-on-event-and-forget model loses data.

---

## 9. Don't lose in-flight writes on kill

❌ State (a queue, a draft, an in-progress upload) held only in memory.
✅ Persist the queue and drafts to disk synchronously enough that an OS kill can't lose them; on launch, recover and resume.
> Consequence: iOS/Android kill backgrounded apps freely and the user can swipe-kill at any moment. Anything not written to durable storage before the kill is gone — the unsent message, the unsaved form, the queued mutation. Persist before you'd miss it.

---

## 10. Cache TTL / staleness & encrypting sensitive cache

❌ A cache with no expiry that serves stale data forever, or caching tokens/PII in plaintext.
✅ Stamp cached entries with a TTL/`fetchedAt`, show staleness or revalidate (stale-while-revalidate), and encrypt sensitive cached values (Keychain/Keystore-backed key).
> Consequence: an unbounded cache shows wrong prices/balances/content indefinitely; plaintext-cached secrets are exfiltratable from a backup or rooted device. (Crypto/secret-handling depth → **security-review**.)

---

## Quick scan checklist

- [ ] Reads come from a local store (source of truth); writes go local + queued — UI never dead-ends offline.
- [ ] Every schema change ships a tested, forward-only migration; no destructive fallback in prod.
- [ ] Store choice fits the data (KV for prefs, relational DB for queried/large data); sensitive data encrypted at rest.
- [ ] Mutations persisted to a durable queue *before* the network call; drained with exponential backoff + jitter; resumed on launch.
- [ ] Replayable mutations carry a stable idempotency key reused across retries (no double charge/order).
- [ ] Conflict strategy is explicit (server-authoritative / LWW / merge), conflicts detected via version/`updatedAt`.
- [ ] Optimistic updates roll back (or reconcile) on failure — the UI never shows an unaccepted state.
- [ ] Sync is delta-based with a persisted cursor and paginated — resumable after an interrupted sync.
- [ ] Connectivity events trigger a drain attempt; success is decided by the response, not a boolean.
- [ ] Queue, drafts, and in-progress uploads persist to disk so an OS kill / swipe doesn't lose them.
- [ ] Caches have a TTL/staleness story; sensitive cached values are encrypted.
