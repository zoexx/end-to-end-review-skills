# Query performance review

A query is fast or slow based on how the planner executes it, not how it reads. The job here is to predict the plan: which index can serve this, will the predicate be sargable, how many rows will it touch as the table grows. "Fast on my machine" with 10k rows says nothing about 10M.

Each item: anti-pattern, fix, consequence.

---

## Missing indexes

### No index on the WHERE / JOIN / ORDER BY column
❌ `SELECT * FROM events WHERE user_id = $1 ORDER BY created_at DESC` with no index covering it
✅ `CREATE INDEX ON events (user_id, created_at DESC);`
**Consequence:** A sequential scan reads every row and an explicit sort spills to disk. Linear in table size — invisible at 10k rows, a multi-second query at 10M.

### No index on a foreign key
❌ Joining `orders` to `order_items` on `order_id`, no index on `order_items.order_id`
✅ `CREATE INDEX ON order_items (order_id);`
**Consequence:** Every join (and, in Postgres, every parent delete/update that checks references) scans the child table. Postgres does not auto-index FKs; MySQL/InnoDB does. The single most common missing-index finding.

### ORDER BY can't use an index
❌ `ORDER BY created_at DESC` where the index is `(created_at ASC)` and you also filter — or no index at all
✅ Match index direction to the sort, or rely on Postgres reading a single-column index backward; for `WHERE a = $1 ORDER BY b` use `(a, b)`
**Consequence:** The planner does a full sort of the matched rows in memory or on disk instead of walking the index in order — slow and memory-hungry on large result sets.

---

## Composite index column order

The rule: **equality columns first, then one range/sort column.** An index on `(a, b, c)` can serve `a`, `a,b`, `a,b,c`, and `a` plus a range on `b` — but **not** `b` alone, nor `a` plus a range on `b` then equality on `c`.

❌ Index `(created_at, tenant_id)` for the query `WHERE tenant_id = $1 AND created_at > $2`
✅ Index `(tenant_id, created_at)` — equality (`tenant_id`) first, range (`created_at`) second
**Consequence:** With the range column first, the index can't seek to one tenant; it scans a date range across all tenants and filters. Reordering the columns turns a scan into a seek.

❌ A separate index per column (`(a)`, `(b)`, `(c)`) hoping the planner combines them
✅ One composite index in the order the queries use; combine independent ones only when access patterns truly vary
**Consequence:** Bitmap-AND of several single-column indexes is far slower than one composite that matches, and you pay write cost on all three.

---

## Sargability (don't defeat the index)

A predicate is *sargable* if the engine can use an index to satisfy it. Wrapping the indexed column kills that.

### Function on the indexed column
❌ `WHERE lower(email) = $1`, `WHERE date(created_at) = $1`, `WHERE EXTRACT(year FROM d) = 2024`
✅ Functional index `CREATE INDEX ON users (lower(email))`; or rewrite to a range: `WHERE created_at >= $1 AND created_at < $1 + interval '1 day'`
**Consequence:** The index is on `email`, not `lower(email)`, so the planner can't use it — full scan. A range rewrite or a matching functional/expression index restores the seek.

### Leading-wildcard LIKE
❌ `WHERE name LIKE '%smith'` or `LIKE '%smith%'`
✅ Anchored prefix `LIKE 'smith%'` uses a B-tree; for contains/fuzzy use a trigram index (`pg_trgm` GIN) or full-text search
**Consequence:** A leading `%` means the engine can't know where in the B-tree to start — full scan. Only a prefix match (no leading wildcard) is index-usable on a plain B-tree.

### Implicit type cast
❌ `WHERE phone = 123456` where `phone` is `varchar`; `WHERE id = '42'` mixing types; joining `int` to `bigint` to `text`
✅ Match types exactly — compare `varchar` to a string literal, align column types across a join
**Consequence:** The engine casts the column (not the literal) to compare, which is `func(col)` in disguise — the index is bypassed and you get a scan. Common after an ORM passes a number for a string column.

### OR across different columns
❌ `WHERE a = $1 OR b = $2` where only one of `a`,`b` is indexed
✅ Index both and let the planner bitmap-OR, or rewrite as `UNION` of two indexed lookups
**Consequence:** A single missing side forces the whole predicate to a scan, since the engine must check every row for the unindexed branch.

---

## Reading too much

### SELECT *
❌ `SELECT * FROM users JOIN profiles ...` when you need three columns
✅ Select only the columns you use
**Consequence:** Pulls wide/`text`/`jsonb`/`bytea` columns over the wire and into memory you don't need, defeats covering-index-only scans, and a later schema change can break or balloon the query. In an ORM, `SELECT *` also tends to hydrate full objects, feeding N+1.

### Missing LIMIT
❌ A list/search endpoint with no `LIMIT`, returning the whole table
✅ Always bound user-facing reads with a `LIMIT`
**Consequence:** Works in dev with 50 rows, returns 5 million in prod — OOM, timeout, or a frozen client. The fix is one clause.

---

## Pagination

### Deep OFFSET pagination
❌ `... ORDER BY id LIMIT 20 OFFSET 100000`
✅ Keyset/seek: `... WHERE id > $last_seen_id ORDER BY id LIMIT 20`
**Consequence:** `OFFSET n` makes the engine read and discard `n` rows before returning yours — page 5000 scans 100k rows to show 20. Keyset pagination seeks straight to the cursor and stays O(LIMIT) regardless of depth (it needs a stable, indexed, unique sort key — often `(created_at, id)`).

---

## ORM-specific traps

### N+1 from lazy loading
❌ `for (const order of orders) { order.items }` — one query for orders, then one per order
✅ Eager-load: Prisma `include`, SQLAlchemy `selectinload`/`joinedload`, ActiveRecord `includes`, TypeORM `relations`/`leftJoinAndSelect`, GORM `Preload`
**Consequence:** 1 + N queries instead of 1 or 2. 200 orders = 201 round trips; latency is dominated by network, not by the database. Show up as a `Nested Loop` and a flood of identical small queries in the log.

### Accidental cross join / cartesian product
❌ A join missing its `ON`/`USING` condition, or an ORM relation misconfigured so two collections multiply
✅ Ensure every join has a key condition; when joining multiple one-to-many relations, aggregate or split queries to avoid row multiplication
**Consequence:** N × M rows instead of N + M — a few thousand rows each side becomes millions, blowing up memory and time, and inflating aggregates (`SUM` counts each row repeatedly).

### Over-indexing
❌ Adding an index for every column "just in case," or one per query on a write-heavy table
✅ Index for real access patterns; drop unused indexes (check `pg_stat_user_indexes.idx_scan = 0`)
**Consequence:** Every `INSERT`/`UPDATE`/`DELETE` must update every index — more indexes = slower writes, more bloat, bigger storage, and the planner has more options to get wrong. Indexes are not free; they're a write tax paid for read speed.

---

## Index features worth knowing

### Covering / index-only scan
A query whose every referenced column is in the index can skip the table entirely.
✅ Postgres: `CREATE INDEX ON orders (user_id) INCLUDE (status, total);` — `INCLUDE` carries extra columns without making them part of the key. MySQL/InnoDB: secondary indexes already include the PK, and you can add columns to the key.
**Consequence:** An index-only scan avoids the heap fetch per row — often several times faster for hot read paths. (Postgres still needs the visibility map current, so keep the table well-vacuumed.)

### Partial index
✅ `CREATE INDEX ON jobs (created_at) WHERE status = 'pending';`
**Consequence:** A much smaller index that only covers the rows you query (e.g. the work queue), faster to scan and cheaper to maintain than a full-column index — ideal for soft-delete (`WHERE deleted_at IS NULL`) and status-queue patterns.

### Function/expression index
✅ `CREATE INDEX ON users (lower(email));` to make `WHERE lower(email) = $1` sargable
**Consequence:** Restores index use for the case-insensitive / computed lookups that would otherwise scan. The index expression must match the query expression exactly.

---

## Reading EXPLAIN ANALYZE

`EXPLAIN` shows the planner's estimate; `EXPLAIN ANALYZE` runs the query and shows actual rows and time. (It executes — wrap a write in a `BEGIN ... ROLLBACK`.) What to look for:

- **`Seq Scan` on a large table** where you expected an index → missing/unusable index, or the planner thinks a scan is cheaper (sometimes correct for small tables or low selectivity).
- **Estimated `rows` vs actual `rows` far apart** → stale statistics; run `ANALYZE <table>`. The planner picks bad plans (e.g. nested loop instead of hash join) when its row estimate is off by orders of magnitude.
- **`Nested Loop` over a large outer set** → O(outer × inner). Great for a handful of rows, a disaster for thousands; usually means a missing index on the inner side or an N+1 pattern.
- **Most expensive node** → read `actual time` and loops, not tree position. `(actual time=... rows=... loops=N)` — multiply by `loops`. Find where the time actually goes before optimizing.
- **`Sort` / `Hash` spilling to disk** (`Sort Method: external merge Disk`) → not enough `work_mem`, or sorting more rows than needed (missing index for `ORDER BY`).

MySQL: `EXPLAIN` (and `EXPLAIN ANALYZE` on 8.0+). Watch `type: ALL` (full scan) and `type: index` (full index scan — better but still everything), `key: NULL` (no index chosen), `rows` (estimated examined), and `Extra: Using filesort` / `Using temporary` (sort or temp table — often a missing index for `ORDER BY`/`GROUP BY`).
