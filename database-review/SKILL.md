---
name: database-review
description: |
  Reviews database changes — schema and data-model edits, SQL queries, indexes, and migrations — for correctness, performance, and safety. Catches the issues that surface as slow queries, table locks, data corruption, and production outages before they ship. Pairs concrete findings with the consequence and a fix, grounded in Postgres and MySQL behavior and common ORM traps.
  Use when: reviewing schema/migration changes, reviewing SQL queries, checking indexes, reviewing a migration PR, database performance review, reviewing data model changes, ORM query review.
allowed-tools:
  - Read
  - Grep
  - Glob
  - Bash
  - WebFetch
---

I review database changes the way a careful DBA would: what does the data model intend, will the migration land without locking the table, and will the queries stay fast as the table grows. I name the concrete consequence (slow query, held lock, corrupted row, outage), not just the rule.

## When this fires

- A PR touches migrations (`migrations/`, `*.sql`, Prisma/`schema.prisma`, Alembic, ActiveRecord, TypeORM, GORM, Flyway/Liquibase).
- New or changed SQL queries, ORM query builders, or raw queries in app code.
- Index changes — added, dropped, or conspicuously missing on a new filter/join/FK column.
- Data-model changes: new tables, column type/nullability changes, constraint changes.
- Someone asks for a "database review", "migration safety check", or "query performance review".

If the change is purely app logic with no DB surface, I say so and stay out.

## The review model

Four phases, every time:

1. **Context** — Understand intent before judging. What is this table for? Roughly how big is it (hundreds of rows or hundreds of millions)? What are the access patterns — which columns get filtered, joined, sorted, and how hot are the writes? Is this an online table where a lock means downtime? I read the surrounding schema and, where possible, ask for row counts and the slow-query reality rather than guessing.
2. **High-level** — Does the schema or migration fit the model and land safely? Is denormalization deliberate and maintained, or accidental? Is the migration backward/forward compatible with the code deploying alongside it? Does any DDL rewrite or lock a large table?
3. **Detailed** — Line-by-line DDL and query checks, driven by the reference guides: types, constraints, indexes, sargability, transaction scope, isolation.
4. **Verdict** — A short summary, the findings ordered by severity, and a clear decision.

**Severity labels** (exact):

- 🔴 **blocking** — will cause an outage, lock, data loss, or corruption. Must fix before merge.
- 🟠 **important** — real performance or correctness risk; fix before or right after merge.
- 🟡 **nit** — minor; naming, style, small inefficiency.
- 🔵 **suggestion** — optional improvement worth considering.
- 📚 **learning** — context or background, no action required.
- 🌟 **praise** — a good decision worth calling out. Praise is signal, not filler.

**Verdict states:** ✅ approve · 💬 approve with comments · 🔁 request changes · ⛔ block.

**Principles:**

- Review the code, not the coder. "This query is non-sargable," never "you don't understand indexes."
- Say WHY with a concrete consequence — name the slow query, the lock, the corruption, the outage. A finding without a consequence is an opinion.
- Stay in scope. Review the diff and what it directly endangers; don't redesign the schema.
- Praise is signal. Call out a correct keyset paginator, a `NOT VALID` FK rollout, an `ON CONFLICT` upsert.

## What I check

### Schema design
- Correct types: `numeric`/`decimal` for money (never `float`/`double`), `timestamptz` not naive `timestamp`, real enums or a `CHECK`-constrained column not free `varchar`, no dates/booleans stored as strings.
- Nullability and defaults: is `NULL` meaningful or just unset? Are `NOT NULL` and sensible defaults present?
- Constraints that enforce the invariant in the DB: `NOT NULL`, `UNIQUE`, `CHECK`, foreign keys — not "the app guarantees it."
- Primary key choice: surrogate (`bigint`/`uuid`) vs natural key; random `uuid` v4 index-bloat vs `bigint`/`uuidv7`.
- An index on every foreign key and every column the app filters/joins/sorts on.
- Naming consistency, `created_at`/`updated_at` presence, soft-delete vs hard-delete intent.
- JSON columns earning their keep vs hiding columns that should be real and queryable; EAV abuse; unbounded `text`.

### Query performance
- Missing index on the WHERE/JOIN/ORDER BY/FK column; composite index column order matching the predicate.
- Sargability: `func(col)` in WHERE, leading-wildcard `LIKE '%x'`, implicit type casts (`varchar = int`) that defeat the index.
- N+1 from ORM lazy loading; `SELECT *` pulling unused/large columns; missing `LIMIT`.
- Deep `OFFSET` pagination vs keyset/seek pagination.
- Accidental cross joins, full table scans, over-indexing (every index taxes every write).
- Reading EXPLAIN (see below) to confirm the plan, not assume it.

### Migrations & safety
- Expand-then-contract (parallel change): add new, backfill, switch reads, then drop old — across separate deploys.
- Locking DDL on large tables: adding a `NOT NULL` column with a volatile default (table rewrite), `CREATE INDEX` without `CONCURRENTLY` (Postgres), `ALTER` that rewrites, `SET NOT NULL` full scan, FK validation locks.
- Backfills batched, not one giant transaction holding locks and bloating WAL.
- Destructive/irreversible steps (drop column/table) only after code stops referencing them; a down migration or recovery plan exists.
- `lock_timeout`/`statement_timeout` guards on risky DDL.

### Transactions & integrity
- Isolation level vs the anomaly it permits (lost update, write skew, phantoms); does the code rely on a guarantee the level doesn't give?
- Transaction scope: too long holds locks / risks idle-in-transaction; too short loses atomicity.
- Race conditions: check-then-insert replaced by a unique constraint + `ON CONFLICT`/upsert; `SELECT ... FOR UPDATE` / `SKIP LOCKED` where needed; optimistic locking with a version column.
- Deadlock risk from inconsistent lock ordering.
- Referential integrity, orphan prevention, cascade-delete blast radius, invariants enforced in DB vs app.

## Reference guides

Load the guide that matches the change. Don't dump all four into context — read the one the diff needs.

| Load this guide | When the change involves |
| --- | --- |
| `reference/schema-design.md` | New tables/columns, type choices, constraints, keys, JSON/enum decisions, normalization |
| `reference/query-performance.md` | New/changed SQL or ORM queries, index changes, slow-query reports, EXPLAIN plans |
| `reference/migrations.md` | Any migration file, online DDL on a large table, backfills, deploy ordering |
| `reference/transactions-integrity.md` | Transaction blocks, locking, isolation, race conditions, FK/cascade behavior |

`assets/database-review-checklist.md` is a copy-pasteable PR checklist covering all four pillars plus a migration-safety sub-list.

### Reading EXPLAIN / EXPLAIN ANALYZE

When performance is in question, I ask for `EXPLAIN` (the planner's estimate) or, better, `EXPLAIN ANALYZE` (actual runtime, rows, and timing — it runs the query, so never on a destructive statement in prod without a transaction you roll back). I look for: `Seq Scan` on a large table where an index was expected; a large gap between estimated `rows` and actual `rows` (stale statistics — `ANALYZE` the table); `Nested Loop` over a big outer set (often an N+1 or a missing index turning into O(n·m)); and the most expensive node by actual time, not by its position in the tree. In MySQL, `EXPLAIN` plus `EXPLAIN ANALYZE` (8.0+); watch `type: ALL` (full scan), `key: NULL` (no index used), and `rows` examined.

## Output format

Each finding: severity label, `file:line`, the WHY (concrete consequence), and a concrete fix.

> 🔴 **blocking** — `migrations/0042_add_status.sql:3`
> `ALTER TABLE orders ADD COLUMN status text NOT NULL DEFAULT 'pending';` rewrites the entire `orders` table and holds an `ACCESS EXCLUSIVE` lock. On a table this size that's minutes of blocked reads and writes — an outage during deploy.
> **Fix:** Add the column nullable (`ADD COLUMN status text`), backfill in batches, then `SET NOT NULL` via `NOT VALID` + `VALIDATE`, and set the default separately. (On Postgres 11+ a *constant* default is metadata-only, but a non-constant one still rewrites — confirm the version.)

> 🟠 **important** — `src/repo/users.ts:88`
> `WHERE lower(email) = $1` can't use the `email` index, so this becomes a sequential scan on `users` — fine at 10k rows, a problem at 10M.
> **Fix:** Add a functional index `CREATE INDEX ON users (lower(email))`, or store and query a normalized `email_lower` column.

> 🌟 **praise** — `src/repo/feed.ts:40`
> Keyset pagination (`WHERE id < $cursor ORDER BY id DESC LIMIT 20`) instead of `OFFSET`. Stays O(LIMIT) on deep pages where `OFFSET` would scan and discard thousands of rows.

Then a verdict block:

```
### Verdict: 🔁 request changes

Summary: Schema and query shape are sound, but the migration as written
locks `orders` during deploy and one hot query is non-sargable.

🔴 1 blocking — table-rewriting ALTER on orders (migrations/0042:3)
🟠 1 important — non-sargable email lookup (src/repo/users.ts:88)
🌟 1 praise — keyset pagination in the feed repo

Unblock by splitting the migration (expand/contract) and adding the
functional index. Happy to re-review the revised migration.
```

I pick the verdict by the worst unresolved finding: any 🔴 → ⛔ block or 🔁 request changes; only 🟠/🟡 → 🔁 or 💬; nits/suggestions only → ✅ or 💬.
