# Backend review checklist

Copy-pasteable into a PR. Grouped by the six pillars. Skip rows that don't apply; flag what does with a severity label (🔴 blocking · 🟠 important · 🟡 nit · 🔵 suggestion · 📚 learning · 🌟 praise).

## API design & contracts
- [ ] Status codes match outcome — no `200` with an error body; 4xx (caller) vs 5xx (us) used correctly
- [ ] `PUT`/`DELETE` idempotent; `GET` has no side effects; retryable `POST` carries an idempotency key
- [ ] Every collection endpoint is paginated with a server-enforced max page size
- [ ] One consistent error envelope; no stack traces, SQL, file paths, or internal ids leaked to clients
- [ ] Breaking changes (removed/renamed field, narrowed type, new required input) are versioned or made additive
- [ ] External ids are opaque, not sequential auto-increment
- [ ] Input validated at the edge with the standard error envelope; request body size is bounded
- [ ] No verbs-in-URL drift; REST/RPC paradigm applied consistently
- [ ] Money is integer minor-units/decimal; time is UTC with explicit zone (not float, not naive `now()`)
- [ ] Auth + rate-limit hooks present on mutating/expensive routes (depth → security-review)

## Business logic correctness
- [ ] Boundary conditions handled — null/empty/zero, off-by-one, overflow, negative inputs
- [ ] Invariants hold on every path; transaction boundary wraps exactly the work that must commit together
- [ ] Partial failures in multi-step flows are compensated or fail atomically — never left half-applied
- [ ] Time/timezone/DST and money arithmetic are correct (UTC storage, exact decimal/integer money)
- [ ] Feature flags / config default to safe and are read consistently

## Error handling & resilience
- [ ] No swallowed exceptions; no catch-log-continue on a path whose failure matters
- [ ] Every network/IO call has an explicit timeout/deadline
- [ ] Retries use exponential backoff + jitter, are capped, and only on retryable + idempotent ops
- [ ] Flaky/essential dependencies have a circuit breaker; non-essential ones degrade gracefully
- [ ] Internal errors mapped to safe client responses; full detail logged with a trace id
- [ ] Exceptions aren't used for expected control flow
- [ ] Fan-out collects partial failures (`allSettled`/DLQ) instead of dropping or over-retrying
- [ ] Resources (handles, connections, locks, txns) released on every exit path (`finally`/`defer`/with-context)
- [ ] Request context/cancellation propagated to downstream calls

## Concurrency & async
- [ ] No check-then-act races; uniqueness/atomicity enforced by the store, not app logic
- [ ] Read-modify-write is atomic (DB-side) or guarded by optimistic versioning — no lost updates
- [ ] No unawaited promises / fire-and-forget without explicit error capture
- [ ] No sync CPU/IO (`*Sync`, big parse, blocking loop) on a single-threaded event loop
- [ ] `Promise.all` vs `allSettled` chosen deliberately; large fan-outs are concurrency-capped
- [ ] Spawned goroutines/tasks have a cancellation/exit path; none leak
- [ ] Multiple locks acquired in a consistent order; no IO while holding a lock
- [ ] Scarce connections/threads not held across slow external calls; pools right-sized
- [ ] No shared mutable singleton holding per-request state; context threaded per request
- [ ] Queue consumers are idempotent (dedupe on message id) for at-least-once delivery

## Observability
- [ ] Logs are structured key-value, not `print`/string concat
- [ ] No secrets/PII in logs; logger-level redaction in place
- [ ] Correlation/trace id minted at the edge and propagated to downstream calls + error responses
- [ ] Log levels match operational meaning (404 isn't `error`; pool exhaustion isn't `info`)
- [ ] Key paths emit RED metrics; key resources emit USE metrics
- [ ] Metric labels are low-cardinality (no ids/emails/raw URLs as labels)
- [ ] Alerts fire on user-visible symptoms with a `for:` and a runbook, not raw internal gauges
- [ ] Distributed traces span service hops; `traceparent` propagated
- [ ] No swallowed error is invisible — every failure logs + increments a metric
- [ ] Sensitive actions write a durable audit trail, transactionally with the change

## Data access
- [ ] No DB/HTTP call inside a loop over a result set (N+1) — batch via join/`IN`/eager include
- [ ] No unbounded `SELECT`/`SELECT *`; filter, project, and `LIMIT` in the query
- [ ] Every list/batch read is paginated, preferably keyset over `OFFSET`
- [ ] Transaction scope wraps one invariant — no external IO inside, no related writes split across txns
- [ ] No lazy-loaded relation accessed inside a loop; eager-load where iterating
- [ ] Read-after-write reads from the primary to avoid replica-lag staleness
- [ ] Caches have a TTL, explicit invalidation on write, and single-flight on hot keys
- [ ] Connections pooled and sized to the DB's combined limit; not held across slow IO
- [ ] Queries select only the columns the code uses
- [ ] DB-write-plus-publish goes through a transactional outbox, not two independent steps
- [ ] Index/schema/migration/lock concerns routed to database-review

## Verdict
- [ ] Findings grouped by severity with `file:line` + WHY + concrete fix
- [ ] Explicit decision: ✅ approve · 💬 approve with comments · 🔁 request changes · ⛔ block
