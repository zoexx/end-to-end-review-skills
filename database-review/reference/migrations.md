# Migration & safety review

A migration runs against a live database while the old and new code are both serving traffic mid-deploy. Two failure modes dominate: (1) the DDL takes a lock that blocks reads/writes long enough to be an outage, and (2) the schema change is incompatible with the code running next to it. The discipline that prevents both is **expand-then-contract** plus **online DDL**.

Each item: anti-pattern, fix, consequence.

---

## The expand/contract (parallel change) pattern

Never change a column's shape in one shot on a live system. Split any breaking change into additive steps that are each safe with both old and new code deployed:

1. **Expand** — add the new column/table/index, nullable and unused. Old code ignores it.
2. **Backfill** — populate it in batches (below).
3. **Migrate code** — deploy code that writes both and reads the new (dual-write), then reads only the new.
4. **Contract** — once nothing references the old, drop it.

❌ One migration that renames `name` → `full_name` and a deploy that switches the code at the same instant.
✅ Add `full_name`; backfill; dual-write from code; switch reads; later drop `name`.
**Consequence:** A rename is a drop + add to every running process. During the deploy window, old pods query `name` (gone) or new pods query `full_name` (not yet populated) → errors and lost writes. Expand/contract keeps every intermediate state valid for both code versions.

---

## Locking DDL — what rewrites or blocks

The danger is holding a strong lock (Postgres `ACCESS EXCLUSIVE`) on a big table while a full rewrite or scan runs — every read and write queues behind it.

### Adding a NOT NULL column with a volatile/non-constant default
❌ `ALTER TABLE big ADD COLUMN token text NOT NULL DEFAULT gen_random_uuid();`
✅ Add nullable, backfill in batches, then `SET NOT NULL` (via the `NOT VALID` trick below), set default separately.
**Consequence:** A non-constant default must compute a value for every existing row → full table rewrite under `ACCESS EXCLUSIVE`, blocking all access for the duration. (Postgres 11+ stores a *constant* default as metadata — instant — but a volatile expression like `now()`/`gen_random_uuid()` still rewrites. Pre-11, *any* default rewrites.)

### CREATE INDEX without CONCURRENTLY (Postgres)
❌ `CREATE INDEX idx ON big (col);`
✅ `CREATE INDEX CONCURRENTLY idx ON big (col);` — and run it *outside* a transaction
**Consequence:** A plain `CREATE INDEX` takes a lock that blocks writes (and the build can take minutes on a large table) → write outage. `CONCURRENTLY` builds without blocking writes (slower, two table scans, can't run in a txn, and leaves an `INVALID` index to drop+retry if it fails). MySQL 5.6+/InnoDB online DDL builds most indexes without blocking, but check the specific operation.

### SET NOT NULL does a full validating scan
❌ `ALTER TABLE big ALTER COLUMN col SET NOT NULL;` directly
✅ Postgres 12+: add `CHECK (col IS NOT NULL) NOT VALID`, then `VALIDATE CONSTRAINT` (takes only a `SHARE UPDATE EXCLUSIVE` lock, allows writes), then `SET NOT NULL` is cheap because the validated constraint proves it.
**Consequence:** A direct `SET NOT NULL` scans the whole table to verify, holding `ACCESS EXCLUSIVE` the entire time → blocked table. The `NOT VALID` → `VALIDATE` path scans under a weak lock.

### Adding a foreign key validates immediately
❌ `ALTER TABLE child ADD FOREIGN KEY (parent_id) REFERENCES parent(id);`
✅ `... ADD FOREIGN KEY (...) REFERENCES parent(id) NOT VALID;` then later `VALIDATE CONSTRAINT ...;`
**Consequence:** The default form scans the whole child table to verify existing rows, holding strong locks on both tables. `NOT VALID` adds the constraint instantly (enforced for new writes only), and `VALIDATE` checks the backlog under a weaker lock.

### ALTER that rewrites the table
❌ Changing a column type in a way that requires a rewrite (`int` → `text`, narrowing, some `USING` casts).
✅ Add a new column of the target type, backfill, switch over (expand/contract); or confirm the change is metadata-only for your engine/version.
**Consequence:** A rewriting `ALTER TYPE` copies every row under `ACCESS EXCLUSIVE`. Some conversions are free (e.g. `varchar(50)` → `varchar(100)`, or `text` widening), others rewrite — know which before running it on a large table.

### MySQL specifics
- InnoDB online DDL (`ALGORITHM=INPLACE, LOCK=NONE`) handles many changes without blocking, but some still force `COPY` (rebuild) and some take a brief metadata lock — verify per operation in the docs for your version.
- For changes that rebuild a large table, use **gh-ost** or **pt-online-schema-change** (pt-osc): they build a shadow table, backfill, sync via triggers/binlog, and swap — no long lock.
- **Gap locks** under `REPEATABLE READ` can surprise you during DML-heavy migrations (see transactions-integrity.md).

---

## Backfills

### Backfilling in one giant transaction
❌ `UPDATE big SET col = compute(...);` over 50M rows in a single statement/transaction.
✅ Batch by primary key: loop `UPDATE big SET col = ... WHERE id BETWEEN $lo AND $hi`, a few thousand rows per batch, committing each, with a short pause to let autovacuum/replication keep up.
**Consequence:** One huge `UPDATE` holds row locks on everything it touches, bloats the WAL/undo log, and in Postgres creates 50M dead tuples that `VACUUM` must later clean — plus it can run for hours and, if it fails at row 49M, rolls all of it back. Batches keep locks short, bound the bloat, and are resumable.

### Backfill coupled to the schema change in the same migration
❌ One migration that adds the column *and* backfills it inline.
✅ Separate **schema change**, **data backfill**, and **deploy** into distinct steps (often distinct PRs/releases).
**Consequence:** A slow backfill inside a DDL transaction extends the lock window; failure forces you to redo the schema change too. Decoupling lets the fast DDL land instantly and the slow backfill run on its own clock.

---

## Destructive & irreversible changes

### Dropping a column/table before the code stops using it
❌ Drop `legacy_status` in the same release that still has code reading it (or before that code is fully rolled out).
✅ Contract last: deploy code that no longer references the column, confirm it's everywhere, *then* drop in a later migration.
**Consequence:** During the deploy, still-running old pods (or a rollback) `SELECT legacy_status` against a table where it's gone → errors. Drops are forever — order them after the code change, never before.

### No down migration / no recovery plan
❌ An irreversible migration (drop column, destructive transform) with an empty or `raise NotImplementedError` `down()`.
✅ Provide a real reverse where feasible; where truly irreversible (data loss), say so explicitly and ensure a backup/snapshot exists and the change is gated behind review.
**Consequence:** If the release goes wrong you can't roll the schema back, and a rollback of the *code* now hits a schema it doesn't expect. At minimum the reviewer must know it's irreversible and that a backup covers it.

### NOT NULL rollout in one step on a populated table
❌ Add a `NOT NULL` column or flip `SET NOT NULL` immediately on a large, in-use table.
✅ Steps: add nullable → backfill in batches → enforce via `CHECK ... NOT VALID` + `VALIDATE` → `SET NOT NULL` → add default for new rows.
**Consequence:** The one-step version either rewrites the table (default) or scans it under a strong lock (validation). The staged rollout keeps every lock short and every intermediate state valid.

---

## Guards & ordering

### No lock_timeout / statement_timeout on risky DDL
❌ A migration that can block indefinitely if it can't immediately get its lock — and meanwhile queues every query behind it.
✅ `SET lock_timeout = '3s'; SET statement_timeout = '...';` before risky DDL so it fails fast instead of stalling traffic; retry off-peak.
**Consequence:** Without `lock_timeout`, the `ALTER` waits behind a long-running transaction for its lock, and *every new query also waits behind the ALTER* — a lock pile-up that takes the table (or the app) down. A short timeout makes the migration give up quickly instead of cascading.

### Migration ordered wrong relative to the deploy
❌ Schema change deployed after the code that depends on it (or a drop before the code stops using it).
✅ Additive schema first → deploy code → contracting schema last. Make the migration and deploy order explicit.
**Consequence:** Code that references a not-yet-existing column, or a column dropped out from under running code, both throw in production. The expand/contract ordering is what keeps each step compatible with the code on either side of it.

---

## ORM migration tools — what they hide

- **Prisma** (`migrate`): generates SQL you should *read* — it may emit a plain `CREATE INDEX` (not `CONCURRENTLY`) or a column add that rewrites. Edit the generated SQL for online-DDL safety.
- **Alembic / Django / Rails (ActiveRecord)**: `add_column ..., default:` can trigger a rewrite/lock on the engine/version; Rails has `disable_ddl_transaction!` + `algorithm: :concurrently` for Postgres indexes — use them. Django data migrations run in a transaction by default; mark long backfills `atomic = False` and batch them.
- **TypeORM / GORM auto-sync / migration sync**: never run `synchronize: true` / `AutoMigrate` against production — it diffs and applies whatever it wants, including drops, with no safety review. Generate explicit migrations and read them.
- Across all of them: the generated DDL is a draft. Open the SQL, check for table rewrites, non-concurrent indexes, immediate FK/NOT NULL validation, and unbatched backfills before it touches a large table.
