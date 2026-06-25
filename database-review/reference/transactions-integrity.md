# Transactions & integrity review

Transactions decide what "happened together" means and what concurrent writers can do to each other. Most data-corruption bugs here aren't crashes — they're two correct-looking code paths racing, and the database quietly letting both win. The fixes are usually a constraint or an isolation/locking choice, not more application code.

Each item: anti-pattern, fix, consequence.

---

## Isolation levels and the anomalies they permit

Picking a level is choosing which concurrency anomalies you're willing to tolerate. Know the default — most apps run **READ COMMITTED** without realizing it.

| Level | Dirty read | Non-repeatable read | Phantom | Lost update | Write skew |
| --- | --- | --- | --- | --- | --- |
| READ UNCOMMITTED | possible* | possible | possible | possible | possible |
| **READ COMMITTED** (PG/MySQL default**) | no | **possible** | **possible** | **possible** | possible |
| REPEATABLE READ | no | no | no*** | possible (PG detects; MySQL allows) | **possible** |
| SERIALIZABLE | no | no | no | no | **no** |

\* Postgres treats READ UNCOMMITTED as READ COMMITTED (no dirty reads ever).
\** MySQL/InnoDB default is REPEATABLE READ; Postgres default is READ COMMITTED.
\*** Postgres REPEATABLE READ (snapshot isolation) prevents phantoms by reading one snapshot; it does **not** prevent write skew. MySQL REPEATABLE READ uses next-key/gap locks that block many phantoms on locking reads.

**Consequence of getting it wrong:** Code that assumes "the row I read is the row I write" is only true under enough isolation. Under READ COMMITTED two reads in the same transaction can return different values (non-repeatable read); under REPEATABLE READ you can still get write skew. If correctness depends on no write skew (the classic "at least one doctor on call" check), you need **SERIALIZABLE** (and retry logic for serialization failures).

---

## Lost update

Two transactions read the same row, each computes a new value, each writes — the second overwrites the first. The classic read-modify-write race.

❌ App-side: `bal = SELECT balance; bal = bal - 100; UPDATE SET balance = bal`. Two concurrent withdrawals both read 500, both write 400; one withdrawal vanishes.
✅ Option A — atomic write: `UPDATE accounts SET balance = balance - 100 WHERE id = $1 AND balance >= 100;` (the DB serializes the row update).
✅ Option B — pessimistic lock: `SELECT balance FROM accounts WHERE id = $1 FOR UPDATE;` then update in the same txn.
✅ Option C — optimistic lock (version column, below).
**Consequence:** Money/inventory/counters silently drift; the symptom is "the numbers don't add up" with no error in the logs. The atomic `UPDATE ... SET x = x - n` is the simplest fix.

---

## Write skew

Two transactions each read a set, each check an invariant that holds, each write a *different* row — and together they break the invariant. Snapshot isolation (PG REPEATABLE READ) does **not** catch this.

❌ "At least one doctor must stay on call." Two on-call doctors each run `SELECT count(*) WHERE on_call` → sees 2 → each sets their own row `on_call = false`. Now zero on call.
✅ SERIALIZABLE isolation (Postgres detects the conflict and aborts one — add retry), or materialize the conflict with explicit locking (`SELECT ... FOR UPDATE` over the rows the invariant depends on), or enforce the invariant with a constraint where possible.
**Consequence:** A real-world invariant is violated even though every transaction individually saw a valid state. This is the anomaly people don't know exists until it bites; it's why "we use transactions" isn't automatically "we're safe."

---

## Locking reads

### SELECT ... FOR UPDATE
✅ Use when you read a row intending to update it and must block other writers until you commit (the pessimistic-lock half of the lost-update fix).
**Consequence of omitting it:** Another transaction modifies the row between your read and write → lost update. Of overusing it: rows stay locked for the whole transaction, so keep the transaction short.

### Work-queue races / SKIP LOCKED
❌ Many workers `SELECT * FROM jobs WHERE status='pending' LIMIT 1` then update — they all grab the same job, or block in a line behind one lock.
✅ `SELECT ... FOR UPDATE SKIP LOCKED LIMIT 1` — each worker claims a *different* unlocked row.
**Consequence:** Without `SKIP LOCKED`, either duplicate processing (no lock) or a thundering herd serialized behind one lock (plain `FOR UPDATE`). `SKIP LOCKED` is the idiomatic, contention-free job-queue claim (PG 9.5+, MySQL 8.0+).

---

## Optimistic locking (version column)

✅ Add `version int NOT NULL DEFAULT 0`. Update with `UPDATE t SET ..., version = version + 1 WHERE id = $1 AND version = $read_version;` and check the affected-row count — `0` means someone else won; reload and retry.
**Consequence of not using it (or pessimistic locking):** Concurrent edits silently clobber each other (last-write-wins). Optimistic locking is ideal for low-contention, user-facing edits where you'd rather detect a conflict than hold a lock; many ORMs support it (Hibernate/JPA `@Version`, SQLAlchemy `version_id_col`, ActiveRecord `lock_version`).

---

## Check-then-insert (the uniqueness race)

❌ `if not exists(SELECT WHERE email=$1): INSERT`. Two requests both run the `SELECT`, both see "not exists," both `INSERT` → duplicate.
✅ Put a `UNIQUE` constraint on the column and let the DB arbitrate, then handle the conflict:
- Postgres: `INSERT ... ON CONFLICT (email) DO NOTHING` (or `DO UPDATE` for upsert).
- MySQL: `INSERT ... ON DUPLICATE KEY UPDATE`, or `INSERT IGNORE`.
**Consequence:** The application check has an unavoidable gap between `SELECT` and `INSERT`; only the unique constraint closes it atomically. Without it you get duplicate users/orders under concurrency — and the only reliable fix is always the constraint, not a bigger lock in the app. (See schema-design.md: missing UNIQUE.)

---

## Transaction scope

### Transaction too long
❌ Opening a transaction, then doing an HTTP call / sending an email / waiting on user input before committing; or wrapping a giant batch in one txn.
✅ Keep transactions to the database work only. Do external I/O before/after, not inside. Set `idle_in_transaction_session_timeout` (Postgres) as a backstop.
**Consequence:** Held locks block other writers; in Postgres a long-open transaction also pins old row versions so `VACUUM` can't reclaim them → table bloat and degraded performance. "Idle in transaction" connections are a top cause of mysterious lock waits.

### Transaction too short (lost atomicity)
❌ Two related writes (debit one account, credit another) committed in separate transactions.
✅ Wrap operations that must succeed-or-fail together in one transaction.
**Consequence:** A crash between the two commits leaves the system in a half-applied state — money debited but not credited. Atomicity is the whole point of the transaction; splitting it discards the guarantee.

---

## Deadlocks

❌ Transaction A locks row 1 then row 2; transaction B locks row 2 then row 1. They wait on each other forever until the DB kills one.
✅ Acquire locks in a consistent global order (e.g. always by ascending primary key) across all code paths; keep transactions short; be ready to catch a deadlock error and retry.
**Consequence:** The database detects the cycle and aborts one transaction with a deadlock error — intermittent, load-dependent failures that are hard to reproduce. Consistent lock ordering prevents the cycle from forming.

---

## Referential integrity & cascades

### Orphan prevention
❌ Application-only "make sure the parent exists" with no FK.
✅ Declare the foreign key (and index it — see schema-design.md).
**Consequence:** Concurrent deletes/inserts leave children pointing at gone parents; the FK makes the DB refuse the orphaning operation.

### Cascade delete surprises
❌ `ON DELETE CASCADE` on a FK where deleting one parent quietly removes huge subtrees — or a chain of cascades you didn't trace.
✅ Choose the action deliberately: `RESTRICT`/`NO ACTION` (refuse if children exist), `SET NULL`, or `CASCADE` only where the children genuinely have no life without the parent. Trace the full cascade chain.
**Consequence:** A routine "delete this user" cascades through orders, items, payments, and audit logs, destroying data you needed and possibly taking a long lock on a big delete. `CASCADE` is convenient and occasionally catastrophic — know the blast radius.

---

## Where to enforce invariants: DB vs app

Prefer the database for anything that must *always* be true, because the app is one of several writers (jobs, admin scripts, the next service, a future migration) and any of them can skip the app-layer check.

- **Uniqueness, foreign keys, non-null, value domains, simple cross-column rules** → enforce in the DB (`UNIQUE`, `FOREIGN KEY`, `NOT NULL`, `CHECK`). The DB enforces them against every writer, atomically, under concurrency.
- **Complex multi-row / multi-table / external invariants** → enforce in the app (or a SERIALIZABLE transaction / explicit locking), since they can't be expressed as a constraint — but design for the race (see write skew).

**Consequence of app-only enforcement of a DB-expressible rule:** The first bulk import, hotfix script, or second service that writes the table bypasses the rule and corrupts the data — and you find out months later. If the database *can* enforce it, it *should*.
