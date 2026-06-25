# API design & contracts

Deep-dive for reviewing the shape of an API: status codes, idempotency, pagination, versioning, validation, and the error envelope. Each section pairs a ❌ anti-pattern with a ✅ fix and names the incident it prevents.

For auth, rate-limit *enforcement*, and injection depth, hand off to **security-review** — this guide only flags where those hooks are *missing*.

---

## 1. Status codes that lie

HTTP status is part of the contract. Clients, proxies, retry libraries, and monitoring all branch on it. A `200` with an error body defeats every one of them.

❌ Error hidden inside a `200`:
```ts
app.post("/transfer", async (req, res) => {
  const r = await transfer(req.body);
  if (!r.ok) return res.status(200).json({ error: "insufficient funds" }); // ❌
  res.json(r);
});
```
✅ Status carries the outcome:
```ts
if (!r.ok) return res.status(422).json({ error: { code: "INSUFFICIENT_FUNDS", message: "..." } });
res.status(200).json(r);
```
> Incident: a client retry policy "retry on non-2xx" never fires; dashboards show 100% success while transfers silently fail.

Rules of thumb:
- **4xx = caller's fault** (don't retry, don't alert as a server error). **5xx = our fault** (retry-eligible, page-worthy).
- `400` malformed syntax · `401` unauthenticated · `403` authenticated-but-forbidden · `404` not found · `409` conflict (version/duplicate) · `422` well-formed but semantically invalid · `429` rate-limited.
- `201` for create (with `Location`), `202` for accepted-async, `204` for success-no-body.
- Never map a validation failure to `500`, and never map a downstream `5xx` you can't recover from to `200`.

❌ Catch-all turns a client error into a server error:
```python
try:
    user = parse_user(body)          # raises on bad input
except Exception:
    return Response(status=500)      # ❌ caller's bad input, our 500 alert
```
✅ Distinguish:
```python
try:
    user = parse_user(body)
except ValidationError as e:
    return json_response(422, error_envelope("VALIDATION", e.messages))
except Exception:
    log.exception("parse_user failed")
    return json_response(500, error_envelope("INTERNAL", "unexpected error"))
```
> Incident: validation failures pollute the 5xx error budget and trigger pager noise during a traffic spike of bad clients.

---

## 2. Idempotency & HTTP method semantics

The method *promises* a safety property. Breaking it breaks every caching/retry layer that trusts it.

| Method | Promise |
| --- | --- |
| `GET`/`HEAD` | safe — no side effects |
| `PUT`/`DELETE` | idempotent — N identical calls == 1 call |
| `POST` | neither — may create on each call |

❌ `GET` with side effects:
```go
// GET /jobs/123/retry  ❌ — a prefetch, a crawler, or a double-tap re-runs the job
func RetryJob(w http.ResponseWriter, r *http.Request) { enqueue(jobID) }
```
✅ Mutations are `POST`/`PUT`:
```go
// POST /jobs/123/retries
```
> Incident: a browser link-prefetcher or monitoring probe fires every `GET`, re-running jobs nobody asked for.

❌ `DELETE` that isn't idempotent:
```ts
app.delete("/items/:id", async (req, res) => {
  const item = await db.get(req.params.id);
  if (!item) return res.status(404).end();   // ❌ second delete now 404s
  await db.delete(req.params.id);
  res.status(204).end();
});
```
✅ Re-deleting an absent resource is a no-op success:
```ts
await db.delete(req.params.id);  // delete-if-exists
res.status(204).end();           // 204 whether or not it existed
```

**Idempotency keys for `POST`.** Any `POST` that can be retried (it can — networks time out) must dedupe on a caller-supplied key.

❌ Retried create makes duplicates:
```ts
app.post("/orders", async (req, res) => {
  const order = await db.orders.create(req.body); // ❌ retry → second order
  res.status(201).json(order);
});
```
✅ Dedupe on `Idempotency-Key`:
```ts
app.post("/orders", async (req, res) => {
  const key = req.header("Idempotency-Key");
  if (!key) return res.status(400).json(err("MISSING_IDEMPOTENCY_KEY"));
  const existing = await db.idempotency.get(key);
  if (existing) return res.status(existing.status).json(existing.body); // replay
  const order = await db.orders.create(req.body);
  await db.idempotency.put(key, { status: 201, body: order }, { ttl: "24h" });
  res.status(201).json(order);
});
```
> Incident: a 504 at the gateway makes the client retry; the first request had already created the order → duplicate orders, double fulfillment.

---

## 3. Unbounded collections

Any endpoint returning a list must be bounded. "It's small today" is not a contract.

❌ Return everything:
```python
@app.get("/events")
def list_events():
    return jsonify([e.dict() for e in db.events.all()])  # ❌ grows forever
```
✅ Cursor pagination with a default and a hard max:
```python
@app.get("/events")
def list_events():
    limit = min(int(request.args.get("limit", 50)), 200)   # default 50, cap 200
    cursor = request.args.get("cursor")
    rows = db.events.page(after=cursor, limit=limit + 1)    # fetch one extra to detect "more"
    has_more = len(rows) > limit
    rows = rows[:limit]
    return jsonify({"data": [r.dict() for r in rows],
                    "next_cursor": rows[-1].id if has_more else None})
```
> Incident: a table that was 1k rows in dev is 40M in prod; the endpoint OOMs the service and the DB ships the whole table over the wire.

- Prefer **cursor/keyset** pagination over `OFFSET` for large or live data (offset gets slow and skips/dupes rows as data shifts).
- Cap `limit` server-side regardless of what the client asks for.
- Filtering/sorting params should be an allow-list, not arbitrary column passthrough (also a security-review concern: injection).

---

## 4. One error envelope, everywhere

A client should parse errors the same way for every endpoint. Inconsistent shapes force per-endpoint special-casing and leak internals.

❌ Three different error shapes in one API:
```json
{ "error": "not found" }
{ "message": "validation failed", "fields": {...} }
{ "err": { "msg": "...", "stack": "at db.query (/app/...)" } }   // ❌ leaks stack + paths
```
✅ One stable envelope, no internals:
```json
{ "error": {
    "code": "VALIDATION_FAILED",          // stable, machine-readable
    "message": "email is required",       // human, safe to display
    "details": [{ "field": "email", "issue": "required" }],
    "trace_id": "a1b2c3"                  // for support to correlate logs
}}
```
> Incident: a leaked stack trace reveals the ORM, file paths, and a SQL fragment — reconnaissance for an attacker, and a brittle contract for clients (database-review/security-review care about the leak; here it's a contract smell).

- `code` is the contract; `message` is for humans and may change.
- Never put exception text, stack frames, SQL, file paths, or internal hostnames in a client response.

---

## 5. Breaking changes & versioning

A change is breaking if an existing client stops working: removed/renamed field, narrowed type, new *required* request field, changed enum meaning, tightened validation, changed status code.

❌ Silent breaking change:
```diff
- "amount": 19.99          // was a number
+ "amount": "19.99"        // ❌ now a string — every client's JSON parse breaks
+ "currency": "USD"        // (additive, fine)
```
✅ Additive-only, or version it:
```
GET /v2/payments        # new shape lives under v2; v1 keeps the old contract
# or: add a new field, keep the old, deprecate with a sunset header/date
```
> Incident: a mobile app pinned to v1 starts crashing on deploy because `amount` changed type; you can't fast-fix because old app versions are in users' hands.

Safe (additive): new optional field, new endpoint, new optional query param, a new enum value *if clients tolerate unknowns*.
Breaking: remove/rename a field, make an optional field required, narrow a type, change defaults/semantics, remove an enum value.

---

## 6. Leaking internal identifiers

❌ Exposing auto-increment DB ids:
```json
{ "id": 10432 }   // ❌ sequential — reveals row counts, enables enumeration/IDOR
```
✅ Opaque external ids:
```json
{ "id": "ord_8fK2nQ" }   // UUID or prefixed nanoid; map to internal id server-side
```
> Incident: sequential ids let anyone estimate your order volume and walk `?id=10431, 10432, ...` to probe for missing authz (the authz hole itself is security-review's domain).

---

## 7. Over- and under-fetching

❌ One bloated response for every caller:
```ts
// returns the user + all orders + all addresses + payment methods, always
res.json(await loadEverything(userId));  // ❌ huge payload, N joins, slow
```
✅ Let the caller ask for what it needs:
```ts
// GET /users/:id?include=orders   — or sparse fieldsets ?fields=id,email
// or a dedicated /users/:id/orders endpoint
```
> Incident: a mobile list view downloads megabytes per row because the only endpoint returns the full object graph; battery and latency tank. (Under-fetching is the inverse — N round-trips because the response omits a needed nested field; balance the two.)

---

## 8. Validation at the edge

Validate *before* the request reaches business logic — once, declaratively, with the same envelope on failure.

❌ Trusting the body:
```ts
const qty = req.body.quantity;          // could be undefined, "5", -3, 1e9
await reserve(req.body.sku, qty);       // ❌ garbage flows downstream
```
✅ Schema validation rejects early:
```ts
const Body = z.object({
  sku: z.string().regex(/^[A-Z0-9-]{4,32}$/),
  quantity: z.number().int().positive().max(1000),
});
const parsed = Body.safeParse(req.body);
if (!parsed.success) return res.status(422).json(envelope("VALIDATION", parsed.error.issues));
await reserve(parsed.data.sku, parsed.data.quantity);
```
> Incident: `quantity: -3` underflows inventory; `quantity: 1e9` reserves the whole warehouse; both because the edge trusted the client.

---

## 9. Verbs in URLs (RPC creeping into REST)

❌ Action verbs as resources:
```
POST /getUser
POST /createOrderAndSendEmail
POST /updateUserSetActive
```
✅ Nouns + methods (or a clean RPC convention if you've chosen RPC):
```
GET    /users/:id
POST   /orders                 # email is a side effect, not part of the name
PATCH  /users/:id  { active: true }
```
> Note: this is a consistency smell, not a bug. If the codebase is deliberately RPC (gRPC, tRPC), apply RPC conventions consistently instead — the failure mode is *mixing* paradigms so callers can't predict the shape.

---

## 10. Money and time as floats

❌ Float money / naive time:
```py
total = price * 1.0825          # ❌ 0.1 + 0.2 == 0.30000000000000004
created = datetime.now()        # ❌ no timezone — ambiguous across hosts/DST
```
✅ Integer minor units / decimal, UTC with explicit zone:
```py
total_cents = round(price_cents * Decimal("1.0825"))   # exact arithmetic
created = datetime.now(timezone.utc)                    # store UTC, render in user's tz
```
> Incident: float rounding makes invoices off by a cent (fails reconciliation/audit); `datetime.now()` on a server in a different tz makes "today's" reports wrong across midnight.

---

## 11. Unbounded request bodies & missing rate-limit/auth hooks

❌ Accept any size, no limits:
```ts
app.use(express.json());                 // ❌ default/large limit → memory DoS
app.post("/import", handler);            // ❌ no auth, no rate limit on a heavy op
```
✅ Bound the body and require the hooks:
```ts
app.use(express.json({ limit: "256kb" }));
app.post("/import", requireAuth, rateLimit({ max: 10, window: "1m" }), handler);
```
> Incident: a single client streams a 2GB JSON body and the parser OOMs the pod. Depth on auth/rate-limit policy lives in **security-review**; here the review note is simply *the hook is absent on a mutating/expensive endpoint.*

---

## Quick scan checklist

- [ ] No `200` with an error body; 4xx vs 5xx used correctly.
- [ ] `PUT`/`DELETE` idempotent; `GET` side-effect-free; retryable `POST` has an idempotency key.
- [ ] Every collection endpoint paginated with a server-enforced max.
- [ ] One error envelope; no stack traces / internal ids / SQL leaked.
- [ ] Breaking changes versioned or made additive.
- [ ] External ids are opaque, not sequential.
- [ ] Input validated at the edge with the standard error envelope.
- [ ] No verbs-in-URL drift; paradigm (REST/RPC) applied consistently.
- [ ] Money is integer/decimal; time is UTC with explicit zone.
- [ ] Request body size bounded; auth + rate-limit hooks present on mutating/expensive routes.
