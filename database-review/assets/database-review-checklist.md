# Database review checklist

Copy into the PR. Tick what applies; delete sections that don't. Each box names a concrete risk — flag with a severity label (🔴 blocking · 🟠 important · 🟡 nit · 🔵 suggestion · 📚 learning · 🌟 praise) and say WHY.

## Context
- [ ] I understand what the table/change is for and its rough size (hundreds vs millions of rows)
- [ ] I know the access patterns — which columns get filtered, joined, sorted; read vs write heavy
- [ ] I know whether this runs against a live, online table (lock = downtime)

## Schema design
- [ ] Correct types: `numeric`/`decimal` for money (not float), `timestamptz` (not naive timestamp)
- [ ] No dates/numbers/booleans stored as strings
- [ ] Enums constrained: real enum type, `CHECK (col IN (...))`, or lookup table + FK — not free `varchar`
- [ ] `NOT NULL` on columns where null is meaningless; sensible `DEFAULT`s present
- [ ] `NULL` is not abused as a sentinel for 0 / "none"
- [ ] `UNIQUE` on every logically-unique column (closes check-then-insert races)
- [ ] Foreign keys declared for every reference; `CHECK`s for domain rules (`>= 0`, date ordering)
- [ ] Index on every foreign key column (Postgres does NOT auto-create these)
- [ ] Primary key choice sound: surrogate vs natural; not a mutable natural key; not random UUIDv4 on a hot insert path
- [ ] JSON columns justified (sparse/variable) — not hiding fields that should be real, typed, indexed columns
- [ ] `created_at` / `updated_at` present on mutable business tables
- [ ] Soft-delete intent explicit; "live row" queries/uniqueness scoped (e.g. partial index `WHERE deleted_at IS NULL`)
- [ ] Any denormalized/cached value has an owned invalidation plan

## Query performance
- [ ] Index exists for each WHERE / JOIN / ORDER BY column
- [ ] Composite index column order = equality columns first, then one range/sort column
- [ ] Predicates are sargable: no `func(col)`, no leading-wildcard `LIKE '%x'`, no implicit type casts
- [ ] No `SELECT *` on hot paths; only needed columns selected
- [ ] User-facing reads have a `LIMIT`
- [ ] Deep pagination uses keyset/seek, not large `OFFSET`
- [ ] No N+1 (ORM eager-loads relations); no accidental cross join / row multiplication
- [ ] Not over-indexed on a write-heavy table (each index taxes every write)
- [ ] EXPLAIN / EXPLAIN ANALYZE checked where performance matters (seq scan? estimate vs actual rows? nested loop on a big set?)

## Migration safety
- [ ] Change is backward/forward compatible with code deploying alongside it (expand-then-contract)
- [ ] No `ADD COLUMN ... NOT NULL DEFAULT <non-constant>` on a large table (table rewrite + lock)
- [ ] Indexes created `CONCURRENTLY` (Postgres), outside a transaction; MySQL uses online DDL / gh-ost / pt-osc for big rebuilds
- [ ] `SET NOT NULL` staged via `CHECK ... NOT VALID` → `VALIDATE` (avoids full-scan lock)
- [ ] Foreign keys added `NOT VALID` then `VALIDATE` on large tables
- [ ] No type-changing/rewriting `ALTER` on a large table without an expand/contract path
- [ ] Backfills run in batches with commits, not one giant transaction
- [ ] Schema change, data backfill, and deploy are separated into distinct steps
- [ ] Columns/tables dropped only after code stops referencing them (contract last)
- [ ] Reversible where feasible; irreversible/destructive steps flagged with a backup/recovery plan
- [ ] `lock_timeout` / `statement_timeout` set on risky DDL so it fails fast instead of piling up locks
- [ ] Migration ordered correctly vs deploy; ORM-generated SQL read (no surprise rewrite/non-concurrent index/auto-sync)

## Transactions & integrity
- [ ] Isolation level matches the correctness need (READ COMMITTED allows non-repeatable read + lost update; REPEATABLE READ still allows write skew; SERIALIZABLE for no-skew invariants + retry)
- [ ] Read-modify-write races fixed (atomic `UPDATE SET x = x - n`, `FOR UPDATE`, or version column) — no lost updates
- [ ] Write-skew invariants protected (SERIALIZABLE or explicit locking)
- [ ] Check-then-insert replaced by `UNIQUE` constraint + `ON CONFLICT` / `ON DUPLICATE KEY` upsert
- [ ] Job-queue claims use `FOR UPDATE SKIP LOCKED`
- [ ] Transactions scoped right: no external I/O / waits inside; related writes that must be atomic are in one transaction
- [ ] Consistent lock ordering across code paths (no deadlock-prone interleaving)
- [ ] FK cascade actions deliberate; cascade-delete blast radius understood
- [ ] Orphan prevention via FK; invariants enforced in the DB where the DB can express them (not app-only)

## Verdict
- [ ] Findings labeled by severity, each with file:line, WHY (concrete consequence), and a fix
- [ ] Verdict chosen by worst unresolved finding: ✅ approve · 💬 approve with comments · 🔁 request changes · ⛔ block
