---
name: backend-review
description: |
  Reviews server-side changes — API contracts, business logic, error handling, concurrency, observability, and data access — for correctness and resilience, not just style. It catches the failures that page someone at 3am: swallowed errors, non-idempotent retries, unbounded queries, races, and breaking API changes shipped without a version bump.
  Use when: reviewing backend changes, API review, reviewing a service/endpoint PR, checking error handling, reviewing concurrency/async code, server-side code review, reviewing business logic, reviewing a handler/controller/worker/queue consumer.
allowed-tools:
  - Read
  - Grep
  - Glob
  - Bash
  - WebFetch
---

# backend-review

A senior-level review pass for server-side code. It reads the change against the contract it's supposed to honor, then works down to the line level using the reference guides — favoring concrete failure modes (the bug, the outage, the data corruption) over taste.

## When this fires

- A PR touches an HTTP/RPC/GraphQL handler, controller, service, use-case, or domain model.
- A queue consumer, cron job, background worker, or batch process changes.
- Someone asks for an "API review," "error-handling review," or "is this safe under concurrency?"
- A change adds or modifies a network/IO call, a transaction, a retry, or a cache.
- The `end-to-end-review` orchestrator routes server-side files here.

Out of scope (route elsewhere): SQL schema/index/migration depth → **database-review**; authn/z, injection, secrets, OWASP → **security-review**; React/CSS/bundle → **frontend-review**. This skill cross-references those rather than duplicating them.

## The review model

Four phases, run in order. Don't nitpick lines before you understand the contract.

1. **Context** — What is this endpoint/service *supposed* to do, and who calls it? Read the PR description, the linked issue, the API contract (OpenAPI/proto/GraphQL schema), and existing callers. Establish the blast radius: is this a public API with external consumers (breaking changes are expensive), an internal service, or a leaf worker? Correctness is relative to intent.
2. **High-level** — Is this the right *shape*? Check service boundaries, the API contract, and how data flows through. Does the resource model fit REST/RPC semantics? Is the transaction boundary in the right place? Is a new dependency or call introduced on a hot path? A wrong abstraction caught here saves a hundred line comments.
3. **Detailed** — Line by line, using the reference guides below. Status codes, idempotency, timeouts, retries, races, N+1s, logging hygiene. This is where most findings land.
4. **Verdict** — Summarize, group findings by severity, and give one explicit decision.

### Severity labels

Every finding carries exactly one:

| Label | Meaning |
| --- | --- |
| 🔴 **blocking** | Must fix before merge — correctness, data loss, outage risk |
| 🟠 **important** | A real problem; should fix, not strictly a blocker |
| 🟡 **nit** | Minor style/polish, non-blocking |
| 🔵 **suggestion** | Optional improvement or alternative |
| 📚 **learning** | Context/teaching, no action required |
| 🌟 **praise** | Genuinely good work, worth calling out |

### Verdict states

`✅ approve` · `💬 approve with comments` · `🔁 request changes` · `⛔ block`

### Principles

- **Review the code, not the coder.** Comment on the change; prefer "this `catch` swallows the error" over "you forgot."
- **Say why.** Every blocking/important finding names the concrete failure: the incident it causes, the data it corrupts, the client it breaks. "Non-idempotent retry → double charge on timeout" beats "add idempotency."
- **Stay in scope.** Flag pre-existing problems separately; don't hold the PR hostage to unrelated cleanup.
- **Praise is signal.** Calling out a correct timeout or a clean idempotency key teaches the pattern.

## What I check

The six pillars. Each links to a reference guide for the dense version.

### 1. API design & contracts
- Status codes match semantics — no `200 {"error": ...}`, correct 4xx vs 5xx, `201`/`202`/`204` where due.
- `PUT`/`DELETE` are idempotent; mutating `GET` is forbidden; `POST` that retries safely carries an idempotency key.
- Collection endpoints are paginated and bounded — no "return all rows."
- One consistent error envelope across the API; errors don't leak stack traces or internal IDs.
- Breaking changes (removed/renamed field, narrowed type, new required input) ship behind a version or are additive-only.
- Input is validated at the edge; request bodies are size-bounded; no verbs in URLs; money/time aren't floats.

### 2. Business logic correctness
- Boundary conditions, null/empty/zero, off-by-one, and overflow are handled.
- Invariants hold across all paths; the transaction boundary wraps exactly the work that must commit together.
- Partial failures in multi-step flows are handled (compensate or fail atomically), not left half-applied.
- Money is integer minor-units or decimal; time is UTC with explicit zones; DST and leap handling considered.
- Feature flags / config default safe and are read consistently.

### 3. Error handling & resilience
- No swallowed exceptions; no catch-log-continue on a path whose failure matters.
- Errors classified retryable vs fatal; retries use exponential backoff **+ jitter** and only on idempotent ops.
- Every network/IO call has a timeout; circuit breakers/bulkheads guard flaky dependencies.
- Resources (handles, connections, locks) are released on the error path; context/cancellation propagates.
- Internal errors are mapped to safe client responses — no stack traces over the wire.

### 4. Concurrency & async
- No check-then-act / read-modify-write races; updates are atomic or guarded (locks, CAS, DB constraints).
- No unawaited async work or fire-and-forget without error capture; the event loop isn't blocked by sync CPU/IO.
- `Promise.all` vs `allSettled` chosen deliberately for partial-failure fan-out; goroutines/tasks don't leak.
- Connection/thread pools sized and not exhausted; ordering and exactly-once vs at-least-once assumptions are explicit.

### 5. Observability
- Structured logs, no secrets/PII; correlation/trace IDs flow across service hops.
- Key paths emit RED/USE metrics; label cardinality is bounded; log levels are used correctly.
- Alerts fire on causes, not just symptoms; sensitive actions leave an audit trail.

### 6. Data access
- No N+1 across the service boundary; no unbounded `SELECT`; result sets are paginated.
- Transaction scope is right-sized; no lazy-load in a loop; replica lag after write is accounted for.
- Caches have TTLs, correct invalidation, and stampede protection; connection pooling is configured.
- (Index/schema/migration depth → **database-review**.)

## Reference guides

Load the guide that matches the change. Don't load all of them — pull the one the diff touches.

| Load this | When the change involves |
| --- | --- |
| `reference/api-design.md` | HTTP/RPC/GraphQL endpoints, status codes, pagination, versioning, request/response contracts |
| `reference/error-handling-resilience.md` | try/catch, retries, timeouts, circuit breakers, calls to other services |
| `reference/concurrency-async.md` | async/await, goroutines, threads, locks, shared state, queue consumers, fan-out |
| `reference/observability.md` | logging, metrics, tracing, alerts, audit trails |
| `reference/data-access-patterns.md` | ORM/query calls, transactions, caching, connection pools |
| `assets/backend-review-checklist.md` | a copy-pasteable PR checklist for a human reviewer |

## Output format

Write findings as inline comments. Each has: a **severity label**, a **`file:line`** anchor, a one-line **WHY** (the concrete failure), and a **concrete fix** — show the corrected code when the fix isn't obvious.

> 🔴 **blocking** — `orders/checkout.ts:142`
> This `POST /charge` retries on timeout but carries no idempotency key, so a slow-but-successful first attempt plus a retry **charges the customer twice**. Thread the client-supplied `Idempotency-Key` to the payment provider and dedupe on it before creating the charge.
> ```ts
> // before: retry with no dedupe
> await retry(() => stripe.charges.create(params), { retries: 3 });
> // after: key the request so the provider collapses duplicates
> await retry(() => stripe.charges.create(params, { idempotencyKey: req.idempotencyKey }), { retries: 3 });
> ```

> 🟠 **important** — `users/list.ts:38`
> `GET /users` returns every row with no `limit`. On a table that grows unbounded this will eventually time out the request and spike DB memory. Add cursor pagination with a default and max page size.

> 🌟 **praise** — `payments/refund.ts:77`
> Nice — the refund path wraps the ledger write and the provider call so a provider failure rolls back the ledger entry. Exactly the right transactional boundary.

Close with a verdict block:

```
## Verdict: 🔁 request changes

🔴 blocking (1)
- checkout.ts:142 — double-charge on retry (no idempotency key)

🟠 important (2)
- users/list.ts:38 — unbounded GET /users (add pagination)
- inventory.ts:91 — read-modify-write race on stock count

🟡 nit (1) · 🔵 suggestion (1) · 🌟 praise (1)

Summary: Core flow is sound, but the retry path can double-charge and the
list endpoint is unbounded. Fix the two blockers/important items and this
is good to merge.
```

Pick the verdict honestly: `✅` clean, `💬` only nits/suggestions, `🔁` important issues to address, `⛔` a blocker that risks data loss/outage/security.
