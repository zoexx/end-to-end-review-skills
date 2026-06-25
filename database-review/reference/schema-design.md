# Schema design review

The schema is the contract for your data. A wrong type or a missing constraint lets bad data in once and it lives forever; cleaning it up later is a migration, a backfill, and a bug hunt. Enforce invariants in the database where you can — the app is one of several writers, and it has bugs.

Each item below: the anti-pattern, the fix, and the one-line consequence.

---

## Types

### Money as float/double
❌ `price double precision` / `amount float`
✅ `price numeric(12,2)` (Postgres `numeric`/`decimal`, MySQL `DECIMAL`)
**Consequence:** Binary floats can't represent `0.10` exactly; sums drift by cents, invoices don't reconcile, and `WHERE amount = 9.99` silently misses rows. Use exact decimal, or store integer minor units (cents).

### Naive timestamp instead of timestamptz
❌ `created_at timestamp` (Postgres `timestamp without time zone`)
✅ `created_at timestamptz` (stored as UTC), set app/session to UTC
**Consequence:** A naive timestamp has no zone; the same value means different instants depending on who reads it. Across a DST change or a multi-region deploy, ordering and "is this expired?" break. (MySQL: `TIMESTAMP` is zone-aware but capped at 2038 — use `DATETIME` only with an explicit UTC convention.)

### Dates / numbers / booleans stored as strings
❌ `signup_date varchar`, `is_active varchar` holding `'true'`/`'yes'`/`'1'`
✅ `signup_date date`, `is_active boolean`
**Consequence:** String dates sort lexically (`'2020-1-9' > '2020-12-1'`), can't do range math, and accept `'2020-13-40'`. String booleans accumulate `'t'`, `'true'`, `'TRUE'`, `'1'`, `'yes'` — every `WHERE` clause becomes a guessing game.

### Enum-as-string with no constraint
❌ `status varchar` with the app "promising" it's one of `pending|paid|refunded`
✅ Postgres `enum` type, or `status text CHECK (status IN ('pending','paid','refunded'))`, or a lookup table + FK
**Consequence:** Without the constraint a typo (`'paied'`) or a new code path inserts a value nothing else handles; reports silently drop rows. Trade-off: Postgres enums are cheap but adding a value needs `ALTER TYPE`; a `CHECK` or lookup table is easier to evolve.

### Unbounded text where a bound belongs
❌ `country_code text`, `phone text` with no length or format constraint
✅ `country_code char(2) CHECK (country_code ~ '^[A-Z]{2}$')`, or validate length
**Consequence:** Garbage like a 4 MB "country code" gets in, breaks downstream fixed-width assumptions, and bloats the row. (In Postgres `varchar(n)` vs `text` has no perf difference — use a `CHECK` for real rules, not `varchar(255)` cargo-culting.)

### Over-wide numeric / wrong integer width
❌ `id int` (32-bit, max ~2.1B) on a high-volume table; `int` for a Unix-millis timestamp
✅ `id bigint` for anything that can grow; right-size deliberately
**Consequence:** A 32-bit `id` exhausts at 2.1 billion rows (or with gaps, sooner) and inserts start failing — a notorious production incident pattern. Migrating PK width on a huge live table is painful; pick `bigint` up front.

---

## Nullability & defaults

### Missing NOT NULL where null is meaningless
❌ `email text` on a users table where every user must have an email
✅ `email text NOT NULL`
**Consequence:** `NULL` leaks in, then `WHERE email = $1` and `WHERE email <> $1` both exclude the null rows (three-valued logic), and `COUNT(email)` undercounts. Every consumer must now handle a state that should never exist.

### NULL used as a sentinel
❌ `discount numeric` where `NULL` secretly means "0% discount"
✅ `discount numeric NOT NULL DEFAULT 0`, reserve `NULL` only for genuinely-unknown
**Consequence:** Aggregations (`SUM`, `AVG`) treat `NULL` as absent, not zero, so totals are wrong; readers split into two camps about what `NULL` means.

### No default on a column the app always sets
❌ `created_at timestamptz NOT NULL` with no default, relying on every insert path
✅ `created_at timestamptz NOT NULL DEFAULT now()`
**Consequence:** One insert path forgets it and the insert fails (or worse, a different path inserts a wrong value). A DB default makes the invariant hold regardless of which writer is involved.

---

## Constraints (enforce in the DB)

### Missing UNIQUE on a logically-unique column
❌ `email text NOT NULL` with uniqueness "checked in the app"
✅ `UNIQUE (email)` (or a unique index, optionally partial: `WHERE deleted_at IS NULL`)
**Consequence:** Two concurrent signups both pass the app-level check-then-insert and you get duplicate accounts — a classic race. Only the DB unique constraint closes the window (see transactions-integrity.md, check-then-insert).

### Missing foreign key
❌ `order_items.order_id bigint` with no FK to `orders`
✅ `FOREIGN KEY (order_id) REFERENCES orders(id)`
**Consequence:** Orphaned rows — items pointing at deleted/nonexistent orders. Joins silently drop them, counts disagree, and "where did this row's parent go" becomes an investigation. (Note: a declared FK is *not* automatically indexed — see "no index on FK" below.)

### Missing CHECK for domain rules
❌ `age int`, `quantity int`, `end_date date` with no guards
✅ `CHECK (age >= 0)`, `CHECK (quantity > 0)`, `CHECK (end_date >= start_date)`
**Consequence:** Negative quantities and end-before-start rows get in via a buggy path and corrupt every calculation built on them. A `CHECK` makes the bad write fail loudly at the source.

---

## Indexing the schema (the FK trap)

### No index on a foreign key column
❌ `FOREIGN KEY (user_id) REFERENCES users(id)` with no index on `user_id`
✅ `CREATE INDEX ON order_items (user_id);`
**Consequence:** Two-fold. (1) `WHERE user_id = $1` and `JOIN ... ON user_id` do a full scan. (2) In Postgres, deleting/updating a `users` row must scan the child table for referencing rows — without the index that's a seq scan per parent delete, and it can take a lock that blocks writes. MySQL/InnoDB auto-creates an index for FKs; Postgres does **not** — this is one of the most common Postgres review findings.

### Index doesn't match the access pattern
❌ Index on `(created_at)` but every query filters `WHERE tenant_id = $1 AND created_at > $2`
✅ Composite index `(tenant_id, created_at)` — equality column first, range column second
**Consequence:** The single-column index forces a scan over all tenants. See query-performance.md for composite column-order rules.

---

## Keys

### Natural key as primary key when it can change
❌ PK = `email` or `ssn` or `(country, tax_id)`
✅ Surrogate PK (`bigint generated always as identity` or `uuid`), with a `UNIQUE` on the natural key
**Consequence:** Natural keys change (people change email), and a PK change cascades to every FK referencing it — a massive, lock-heavy update. A stable surrogate key decouples identity from mutable attributes.

### Random UUID v4 as a high-insert primary key
❌ `id uuid DEFAULT gen_random_uuid()` as the clustered/most-used key on a hot table
✅ Prefer `bigint identity`, or time-ordered UUID v7 / ULID if you need a UUID
**Consequence:** Random UUIDs scatter inserts across the B-tree, causing page splits, index fragmentation, and poor cache locality — measurably slower inserts and a bigger index than a monotonic key. In MySQL/InnoDB the PK is the clustering key, so a random PK is especially costly.

---

## Modeling shape

### JSON blob hiding columns that should be real
❌ `data jsonb` where the app constantly does `data->>'status'` and filters on it
✅ Promote frequently-queried, schema-stable fields to real typed columns with constraints and indexes
**Consequence:** You lose type checking, `NOT NULL`/`CHECK`, and cheap indexing; every filter becomes a JSON extraction, and a typo'd key fails silently. JSON is right for genuinely sparse/variable data — not as a dumping ground for fields you query and constrain.

### EAV (entity-attribute-value) abuse
❌ `attributes(entity_id, key, value text)` to model what are really fixed columns
✅ Real columns; reserve EAV for truly dynamic, user-defined, sparse attributes
**Consequence:** Every "row" reconstructs via many self-joins or pivots, you can't enforce per-attribute types/constraints, and queries are slow and unreadable. EAV trades all of the database's strengths for flexibility you usually don't need.

### Denormalization with no invalidation plan
❌ Caching `user.order_count` or `post.like_count` in a column with nothing keeping it current
✅ Denormalize deliberately, and own the invalidation: triggers, transactional update in the same write, or a periodic reconciler — and document it
**Consequence:** The cached value drifts from the source of truth; users see wrong counts and you can't tell which number is right. Denormalization is a valid speed trade-off, but only with an explicit, owned consistency mechanism.

---

## Housekeeping columns & lifecycle

### Missing created_at / updated_at
❌ No audit timestamps on a mutable business table
✅ `created_at timestamptz NOT NULL DEFAULT now()`, `updated_at timestamptz NOT NULL DEFAULT now()` (kept current by a trigger or the ORM)
**Consequence:** When something looks wrong you have no "when did this happen / when did it last change" — debugging, ordering, and incremental syncs all need these.

### Soft-delete vs hard-delete left implicit
❌ A `deleted_at` column that half the queries forget to filter on
✅ Decide deliberately; if soft-deleting, make "live" rows the easy path — a partial unique index `WHERE deleted_at IS NULL`, a `live_rows` view, and unique constraints scoped to non-deleted rows
**Consequence:** Forgotten `WHERE deleted_at IS NULL` clauses resurrect deleted data into the UI; unique constraints that ignore `deleted_at` block re-creating a record with the same key.

---

## Scaling shape (flag early, don't force)

### A table that's obviously a partitioning candidate
📚 A time-series/event/log table that will reach hundreds of millions of rows where queries are almost always bounded by time or tenant.
✅ Consider range partitioning by time (or by tenant) so old partitions can be dropped cheaply and queries hit one partition.
**Consequence:** Without partitioning, deleting old data is an expensive `DELETE` that bloats the table, `VACUUM` struggles, and every query scans the whole history. Partitioning is a real cost — only raise it when the size and access pattern clearly warrant it, not for a 100k-row table.
