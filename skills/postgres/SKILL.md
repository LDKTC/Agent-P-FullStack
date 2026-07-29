---
name: postgres
description: PostgreSQL-specific database expertise — correct types (`text` over `varchar(n)`, `numeric` for money, `timestamptz` not `timestamp`, `jsonb` not `json`, identity columns over `serial`), index selection across btree/GIN/GiST plus partial, expression, and covering indexes, `EXPLAIN (ANALYZE, BUFFERS)`-driven tuning and sargability, MVCC consequences (bloat, long transactions blocking vacuum, `FOR UPDATE SKIP LOCKED` queues, deadlock retry), connection pooling and PgBouncer transaction-mode constraints, and lock-safe migrations (`CREATE INDEX CONCURRENTLY`, `lock_timeout`, adding NOT NULL columns). Use when `package.json` lists `pg`, `postgres`, or `@neondatabase/serverless`, an ORM's dialect/provider is postgres (Prisma `provider = "postgresql"`, Sequelize `dialect: 'postgres'`, TypeORM `type: 'postgres'`), a docker-compose service runs a `postgres` image, or a `postgres://`/`postgresql://` connection string is present, including when asked directly (Thai or English) to เขียนหรือตรวจ schema/query PostgreSQL ให้ถูกต้องและมีประสิทธิภาพ. Not a replacement for database-schema-dev's engine-agnostic migration workflow or backend-dev's access-layer code — this is the Postgres-specific expertise both draw on, and it also underlies `skills/supabase/SKILL.md`.
---

# PostgreSQL

`database-schema-dev`'s and `backend-dev`'s Postgres-specific expertise
layer, the sibling of `skills/mysql/SKILL.md`. If the project is Supabase,
everything here applies — Supabase *is* Postgres — and
`skills/supabase/SKILL.md` adds the RLS/auth/key-boundary rules on top.

## Step 1 — Confirm the engine is actually Postgres

Check `package.json` for `pg`, `postgres`, `@neondatabase/serverless`, or
`@supabase/supabase-js`; an ORM provider/dialect of `postgresql`/`postgres`;
a `postgres` (or `pgvector`/`timescale`) image in `docker-compose.yml`; or a
`postgres://` connection string. Note the major version — partitioning,
`MERGE`, and several index features are version-gated. If the engine is
MySQL or SQLite, use the matching skill instead.

## Step 2 — Types

- `text` unless a length limit is a real business constraint;
  `varchar(n)` has no performance advantage in Postgres and a length change
  is a migration.
- `numeric(p,s)` for money and any exact decimal — never `float8`/`real`.
- `timestamptz` for every instant. `timestamp` (without time zone) stores a
  wall-clock reading with no offset and is a durable source of
  off-by-hours bugs; `timestamptz` stores UTC and converts on the session's
  `TimeZone`. Use `date` for a calendar day with no time.
- `jsonb` (binary, indexable, deduplicated keys) over `json` (raw text) —
  but anything filtered, joined, or sorted on regularly belongs in a real
  column, not inside a document.
- `uuid` as a native type, not `text`. `gen_random_uuid()` is built in from
  PG13; UUIDv7 (or a ULID) keeps insert locality that random v4 loses on a
  large table.
- Identity columns (`GENERATED ALWAYS AS IDENTITY`) over `serial` — `serial`
  leaves sequence ownership/permission quirks behind and is no longer the
  recommended form.
- Enum types are cheap to add values to (`ALTER TYPE … ADD VALUE`) but
  values can't be removed or reordered, and before PG12 the add can't run
  inside a transaction — a lookup table stays easier to evolve.
- Arrays and `citext`/`ltree`/`postgis` extensions are real options, but
  confirm the extension is available on the target host before depending on
  it.

## Step 3 — Constraints

- Put the constraint in the database, not only in application code: `NOT
  NULL`, `CHECK`, `UNIQUE`, and foreign keys are the last line that holds
  when two services or a migration script write the same table.
- Partial unique indexes express "unique among active rows"
  (`CREATE UNIQUE INDEX … WHERE deleted_at IS NULL`) — the usual answer for
  soft deletes.
- Exclusion constraints (`EXCLUDE USING gist`) handle non-overlapping ranges
  (bookings, schedules) correctly; application-level checks race.
- Foreign key columns need their own index — Postgres creates one for the
  primary key side only, and an unindexed FK makes deletes on the parent
  scan the child table.

## Step 4 — Indexes

- btree is the default and serves equality, ranges, and `ORDER BY`.
  Composite order follows the same rule as everywhere: equality columns
  first, range/sort column last; the leftmost prefix serves shorter lookups.
- GIN for `jsonb` containment, array membership, and full-text search
  (`tsvector`); GiST for geometric/range types and nearest-neighbour. A
  btree on a `jsonb` column will not serve `@>`.
- Expression indexes (`CREATE INDEX ON users (lower(email))`) are how a
  function-wrapped predicate stays sargable — the index expression must
  match the query's expression exactly.
- Partial indexes (`WHERE status = 'pending'`) are much smaller than a full
  index when queries only ever touch a slice.
- `INCLUDE (…)` adds payload columns for index-only scans without widening
  the key.
- Trigram (`pg_trgm` + GIN) makes `LIKE '%term%'` and fuzzy search
  indexable; plain btree cannot serve a leading wildcard.
- Every index costs write throughput and space. Add for real query patterns,
  and check `pg_stat_user_indexes` for ones never scanned before adding
  more.

## Step 5 — EXPLAIN-driven tuning

- `EXPLAIN (ANALYZE, BUFFERS)` on the real query with realistic data — plan
  shape alone proves nothing, and a plan from an empty dev table is
  meaningless because a seq scan is genuinely optimal there.
- Compare estimated vs actual rows: a large mismatch usually means stale
  statistics (`ANALYZE`), a correlated predicate, or an untracked
  expression — fix the estimate before adding indexes to compensate.
- Watch for `Seq Scan` on a large table, `Rows Removed by Filter` in the
  thousands, an external-merge `Sort` (spilling to disk, i.e. `work_mem`
  too low for that query), and nested loops over big row counts.
- Sargability: `WHERE date_trunc('day', created_at) = $1` and
  `WHERE col::text = $1` can't use a plain index — rewrite as a bare-column
  range, or add the matching expression index.
- Deep `OFFSET` pagination scans and discards every skipped row; use keyset
  pagination (`WHERE (created_at, id) < ($1, $2) ORDER BY created_at DESC,
  id DESC LIMIT 20`) for anything that can page far or run unbounded.
- `SELECT *` defeats index-only scans and drags TOASTed columns along.

## Step 6 — MVCC, locking, and vacuum

- Updates write a new row version and leave a dead one; `autovacuum`
  reclaims it. A long-open transaction (including an idle one holding a
  snapshot) blocks that cleanup database-wide — bloat and a slow table
  usually trace back to `idle in transaction` sessions, visible in
  `pg_stat_activity`.
- Keep transactions short and never hold one open across an HTTP call to a
  third party.
- Default isolation is `READ COMMITTED`; each statement sees a fresh
  snapshot, so a read-then-write pair without `SELECT … FOR UPDATE` is a
  lost-update race. `REPEATABLE READ`/`SERIALIZABLE` push the problem into
  serialization failures (SQLSTATE `40001`) the app must catch and retry.
- Deadlocks (`40P01`) happen under real contention — acquire locks in a
  consistent order and retry the loser rather than lengthening the
  transaction.
- `SELECT … FOR UPDATE SKIP LOCKED` is the correct primitive for a job/queue
  table; polling with a plain `SELECT` then `UPDATE` hands the same row to
  two workers.
- Advisory locks (`pg_advisory_xact_lock`) coordinate application-level
  critical sections (a migration runner, a singleton cron) across replicas.

## Step 7 — Connections, pooling, and migration safety

- Postgres connections are processes and are expensive; use a pool sized to
  the database's `max_connections`, not per-request connections. Serverless
  functions multiply this — a pooler (PgBouncer, Supabase's pooler, Neon's
  proxy) is usually mandatory there.
- PgBouncer **transaction mode** breaks session-scoped features: named
  prepared statements, `SET` that must persist, `LISTEN`/`NOTIFY`, session
  advisory locks, and cursors outside a transaction. Drivers usually need
  prepared statements disabled against it. Use the session-mode/direct port
  for migrations.
- Migration DDL takes locks. `ALTER TABLE … ADD COLUMN` with a non-volatile
  default is fast on PG11+, but adding a `NOT NULL` to an existing column
  scans the table, and most `ALTER` forms take `ACCESS EXCLUSIVE`, which
  queues behind — and then blocks — every reader.
- Set a short `lock_timeout` (and `statement_timeout`) in migrations so a
  blocked DDL fails fast instead of stalling the application.
- Build indexes on live tables with `CREATE INDEX CONCURRENTLY` (which
  cannot run inside a transaction block — many migration tools wrap
  everything in one by default, so this needs an explicit escape hatch).
- Roll out breaking column changes in expand/contract steps: add the new
  column, backfill in batches, dual-write, switch reads, then drop.

## Step 8 — Self-check before reporting done

Confirm every timestamp column is `timestamptz`, money is `numeric`, and new
JSON is `jsonb` with anything queryable promoted to a real column. Confirm
foreign keys are indexed, each new index is justified by a query pattern
with `EXPLAIN (ANALYZE, BUFFERS)` showing it used, and no unbounded list
endpoint uses `OFFSET`. Confirm read-then-write paths take a row lock or
handle a serialization failure, and that no transaction spans an external
call. Confirm migrations set `lock_timeout`, use `CONCURRENTLY` for indexes
on populated tables, and don't take a long `ACCESS EXCLUSIVE` lock in a
deploy window. If the app runs behind a transaction-mode pooler, confirm
nothing added depends on session state.

## What not to do

- Don't use `timestamp` for an instant, `float` for money, or `json` where
  `jsonb` is meant.
- Don't leave a foreign key unindexed, or add an index without a query
  behind it.
- Don't trust a plan from an empty dev database, and don't optimize without
  `EXPLAIN (ANALYZE, BUFFERS)`.
- Don't wrap an indexed column in a function or cast in `WHERE` without a
  matching expression index.
- Don't paginate deep lists with `OFFSET`.
- Don't hold a transaction open across a network call, and don't leave
  sessions `idle in transaction` — that's what stops vacuum.
- Don't read-then-write without `FOR UPDATE`, and don't build a queue
  without `SKIP LOCKED`.
- Don't run `CREATE INDEX` (non-concurrent) or an unbounded `ALTER TABLE` on
  a large live table, and don't run migrations through a transaction-mode
  pooler.
- Don't write the engine-agnostic migration or access-layer code here —
  that's `database-schema-dev`/`backend-dev`.
