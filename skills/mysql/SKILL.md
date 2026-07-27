---
name: mysql
description: Deep MySQL/MariaDB-specific database expertise — correct data types and charset/collation (utf8mb4, DECIMAL for money, right-sized VARCHAR/INT, ENUM tradeoffs), InnoDB vs MyISAM, index design and EXPLAIN-driven query tuning (composite index order, covering indexes, sargable predicates, keyset pagination), transaction isolation and locking (REPEATABLE READ, gap locks, deadlocks), and MySQL-specific correctness pitfalls. Use when package.json lists mysql2 or mysql, an ORM's dialect/provider is set to mysql (Prisma `provider = "mysql"`, Sequelize `dialect: 'mysql'`, TypeORM `type: 'mysql'`/`'mariadb'`), a docker-compose service runs a mysql/mariadb image, or a `mysql://` connection string is present, including when asked directly (Thai or English) to เขียนหรือตรวจ schema/query MySQL ให้ถูกต้องและมีประสิทธิภาพ. Not a replacement for database-schema-dev's engine-agnostic migration workflow or backend-dev's access-layer code — this is the MySQL-specific expertise both draw on when the engine actually is MySQL/MariaDB.
---

# MySQL

`database-schema-dev`'s and `backend-dev`'s MySQL-specific expertise layer:
those agents are engine-agnostic and match whatever migration tool/ORM a
project already has — this skill is the actual MySQL knowledge they draw on
when the engine happens to be MySQL, or its close-compatible fork MariaDB.
Where MariaDB behaves differently, that's called out — don't assume full
parity on advanced features.

## Step 1 — Confirm the engine is actually MySQL/MariaDB

Check `package.json` for `mysql2`/`mysql`, an ORM dialect/provider
(`provider = "mysql"` in Prisma, `dialect: 'mysql'` in Sequelize,
`type: 'mysql'`/`'mariadb'` in TypeORM), a `docker-compose.yml` service
running a `mysql`/`mariadb` image, or a `mysql://` connection string. If
none of that holds, stop — use `database-schema-dev`/`backend-dev`'s
engine-agnostic defaults, or the matching engine-specific skill instead.

## Step 2 — Charset, collation, storage engine

- Default charset is `utf8mb4`, not `utf8` — MySQL's `utf8` is a 3-byte-max
  alias that can't hold most emoji/astral characters — with an explicit
  collation (`utf8mb4_0900_ai_ci` on MySQL 8+; MariaDB has no `0900`
  collations, use `utf8mb4_unicode_ci` for anything targeting both).
- `InnoDB` unless there's a specific reason for `MyISAM` (no transactions,
  no row locks, no FKs) — it's the default since MySQL 5.5/MariaDB 10.x, so
  this is usually just confirming nothing overrides it.
- `_ci` collation means `'A@b.com'` matches `'a@b.com'` — use `_bin`/`_cs`
  (or a hashed column) for anything that must compare case-sensitively
  (tokens, secrets).

## Step 3 — Correct data types

- Money/exact decimals: `DECIMAL(p,s)`, never `FLOAT`/`DOUBLE` — binary
  float can't represent most base-10 fractions exactly.
- Right-size integers (`TINYINT`/`SMALLINT`/`INT`/`BIGINT`) by real range,
  not `BIGINT` everywhere by habit — smaller types mean smaller indexes.
  `UNSIGNED` is free headroom on an id column, which is never negative.
- `DATETIME` for a wall-clock value with no timezone conversion intended;
  `TIMESTAMP` (UTC-normalized, auto-converts per session timezone, 2038
  range limit) only for an actual instant in time — don't default to it.
- `VARCHAR(n)` sized to the real constraint, not `VARCHAR(255)` reflexively;
  `TEXT`/`BLOB` for genuinely unbounded content — neither takes a default
  value or a full (non-prefix) index.
- `ENUM` only for a fixed, rarely-changing set — values sort by definition
  order, not alphabetically (a surprise in `ORDER BY`), and appending a
  value is metadata-only on MySQL 8 but can rewrite the table on
  MariaDB/older MySQL. A lookup table scales better once the set grows or
  needs metadata.
- Native `JSON` (MySQL 5.7.8+/MariaDB 10.2+ — MariaDB's `JSON` is actually
  `LONGTEXT` plus a `CHECK`, not a binary type, so none of MySQL's binary
  storage/partial-update wins apply) only for genuinely schemaless data —
  anything regularly filtered or joined on belongs in a real column.

## Step 4 — Index design

- Composite indexes: equality-filtered column(s) first, the range/sort
  column last — `(status, created_at)` serves `WHERE status = ? ORDER BY
  created_at`; the reverse order doesn't.
- Leftmost-prefix rule: an index on `(a, b, c)` serves `(a)` and `(a, b)`
  lookups too — don't add a redundant `(a)` index alongside it.
- A covering index (every selected column included) lets MySQL answer from
  the index alone (`Using index` in `EXPLAIN`) — worth it for a hot, narrow
  query, not by default on every query.
- Prefix index (`col(20)`) for long `VARCHAR`/`TEXT` — check
  `COUNT(DISTINCT LEFT(col, n))` against `COUNT(DISTINCT col)` to size `n`.
- Deep pagination: `LIMIT 20 OFFSET 100000` still scans and discards
  100,000 rows every page. Keyset pagination (`WHERE (created_at, id) <
  (:last_created, :last_id) ORDER BY created_at DESC, id DESC LIMIT 20`)
  reuses the same composite index and seeks straight to the cursor — use it
  for infinite scroll or deep API pagination instead of `OFFSET`.
- Every index costs write time and disk space — add indexes for real query
  patterns (`database-schema-dev` Step 1), not speculatively per column.

## Step 5 — EXPLAIN-driven query tuning

- Run `EXPLAIN` (or `EXPLAIN ANALYZE`, MySQL 8.0.18+/MariaDB 10.4+) before
  trusting a query is fast — don't judge from query shape alone.
- `type: ALL` is a full table scan; `ref`/`range`/`const` mean an index is
  actually used — a query worth optimizing shouldn't show `ALL` past
  trivial table size.
- `Extra: Using filesort`/`Using temporary` means MySQL sorted/materialized
  outside an index — usually fixed by an index matching the `ORDER BY`/
  `GROUP BY`.
- A wrapping function or implicit cast on an indexed column
  (`WHERE YEAR(created_at) = ...`, comparing a string column to a number)
  defeats the index — rewrite as a bare-column range instead
  (`created_at >= '2024-01-01' AND created_at < '2025-01-01'`).
- `SELECT *` on a wide table defeats covering indexes and pulls
  `TEXT`/`BLOB` columns nothing asked for — select only what's used.

## Step 6 — Transactions and locking

- InnoDB locks rows, not tables, but the default isolation `REPEATABLE
  READ` also takes **gap locks** on range scans in `UPDATE`/`DELETE`/
  locking `SELECT` — a range condition can block more than the matched
  rows. `READ COMMITTED` avoids gap locks if repeatable reads aren't needed.
- That snapshot only covers a plain `SELECT` — a locking read in the same
  transaction (`SELECT ... FOR UPDATE`, `UPDATE`, `DELETE`) reads the
  latest committed data instead, so it can see a row the snapshot never
  showed. Don't assume snapshot isolation protects a locking read too.
- Keep transactions short — an open transaction holds its locks and, under
  `REPEATABLE READ`, also blocks InnoDB's purge of old row versions.
- `SELECT ... FOR UPDATE` to lock rows read-then-written in one transaction
  (balance check before debit); acquire locks in a consistent row order
  across the codebase to avoid deadlocks.
- A deadlock (`ERROR 1213`, SQLSTATE `40001`) is expected under real
  contention — InnoDB auto-rolls-back one side; catch that code and retry
  instead of "fixing" it with a longer transaction. `SHOW ENGINE INNODB
  STATUS` → `LATEST DETECTED DEADLOCK` shows what each side held/waited for.
- Autocommit is on per connection by default — multi-statement atomicity
  needs an explicit transaction (or the ORM's wrapper), not just adjacency
  in code.

## Step 7 — MySQL-specific pitfalls and MariaDB divergence

- `sql_mode` should include `STRICT_TRANS_TABLES` (default since MySQL
  5.7/MariaDB 10.2) — without it, out-of-range/wrong-type values are
  silently truncated instead of erroring; confirm it hasn't been relaxed.
- `AUTO_INCREMENT` values aren't gap-free (a rollback still consumes one) —
  never rely on ids being contiguous.
- Session `time_zone` affects `TIMESTAMP` conversion but not `DATETIME` — a
  pool with mixed per-connection timezones is a real "wrong hour" bug; pin
  it explicitly.
- MariaDB has its own extensions with no MySQL equivalent (`CREATE
  SEQUENCE`, `RETURNING` on writes) — don't use them if MySQL is also a
  target; window functions/CTEs need MySQL 8.0+/MariaDB 10.2+, check the
  pinned version first.
- MariaDB's optimizer and `EXPLAIN` output aren't the same engine as
  MySQL's — re-run `EXPLAIN` on the actual engine in use rather than
  trusting a plan tuned on the other one.

## Step 8 — Self-check before reporting done

Confirm every new/changed column has an intentional type and charset falls
back to `utf8mb4`. Confirm any new index is backed by a real query pattern
with `EXPLAIN` output showing it's used, and any deep/unbounded list
endpoint uses keyset instead of `OFFSET`. Confirm multi-statement writes
are wrapped in an explicit transaction and a locking-read's isolation
behavior is deliberate, not assumed. If the compose file/CI pins MariaDB,
confirm nothing added assumes a MySQL-8-only feature.

## What not to do

- Don't use `utf8` charset or `FLOAT`/`DOUBLE` for money — `utf8mb4` and
  `DECIMAL` are the correct defaults.
- Don't add an index without a query pattern behind it, or call a query
  "fast" without checking `EXPLAIN`.
- Don't use `OFFSET` pagination on a list that can page deep or run
  unbounded — use keyset.
- Don't let a transaction stay open longer than the work it protects,
  especially under `REPEATABLE READ`'s gap locks.
- Don't rely on `AUTO_INCREMENT` contiguity or implicit type coercion for
  correctness.
- Don't assume a MariaDB target has MySQL-8-only features (native `JSON`,
  `utf8mb4_0900_ai_ci`) without checking the pinned version.
- Don't write the engine-agnostic migration/access-layer code yourself —
  that's `database-schema-dev`/`backend-dev`; this skill is the MySQL
  knowledge they apply on top of it.
