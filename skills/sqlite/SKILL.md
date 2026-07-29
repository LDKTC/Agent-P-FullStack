---
name: sqlite
description: SQLite-specific database expertise — dynamic typing and column affinity (and `STRICT` tables), the absence of native boolean/date types and what to store instead, foreign keys being off by default per connection, the single-writer concurrency model with WAL mode and `busy_timeout`, `BEGIN IMMEDIATE` for write transactions to avoid upgrade deadlocks, `EXPLAIN QUERY PLAN`-driven index design, the limited `ALTER TABLE`, and safe backup/deployment of a file-backed database. Use when `package.json` lists `better-sqlite3`, `sqlite3`, `node:sqlite`, `@libsql/client`, or `expo-sqlite`, when an ORM's dialect/provider is `sqlite` (Prisma `provider = "sqlite"`, Drizzle `better-sqlite3`/`libsql`, TypeORM `type: 'sqlite'`), when a `.db`/`.sqlite`/`.sqlite3` file or a `file:` connection string is in the repo, or when asked directly (Thai or English) to เขียนหรือตรวจ schema/query SQLite ให้ถูกต้อง. Not a replacement for database-schema-dev's engine-agnostic migration workflow or backend-dev's access-layer code — this is the SQLite-specific expertise both draw on, and it also underlies `skills/turso/SKILL.md`.
---

# SQLite

`database-schema-dev`'s and `backend-dev`'s SQLite-specific expertise layer,
the same way `skills/mysql/SKILL.md` is for MySQL. SQLite is an embedded
library writing to a single file, not a server — most of what follows comes
from that one difference. If the project is Turso/libSQL, everything here
still applies to the schema and queries; the connection, replication, and
transaction-shape rules on top of it are `skills/turso/SKILL.md`.

## Step 1 — Confirm the engine is actually SQLite

Check `package.json` for `better-sqlite3`, `sqlite3`, `node:sqlite`,
`@libsql/client`, `expo-sqlite`, or `drizzle-orm` with a SQLite driver; an
ORM provider/dialect set to `sqlite`; a committed or gitignored
`.db`/`.sqlite` file; or a `file:`/`libsql:` connection string. If the
engine is MySQL or Postgres, use `skills/mysql/SKILL.md` or
`skills/postgres/SKILL.md` instead — the type systems and concurrency models
are genuinely different, and SQLite habits port badly.

## Step 2 — Dynamic typing, affinity, and STRICT

- SQLite does not enforce declared types by default. A `VARCHAR(10)` column
  happily stores a 500-character string and an integer; the declared type
  only sets a *type affinity* that guides conversion.
- Use `STRICT` tables (SQLite 3.37+) on anything new —
  `CREATE TABLE t (…) STRICT` enforces the declared type per row and turns a
  whole class of silent data corruption into an error. Confirm the shipped
  library version supports it first.
- There is no `BOOLEAN`: store `INTEGER` `0`/`1`. Add a
  `CHECK (flag IN (0,1))` so a stray `2` can't appear.
- There is no date/time type. Pick one representation per project and stay
  with it — ISO-8601 UTC `TEXT` (`'2026-07-29T10:00:00Z'`, sorts and
  compares lexicographically) or an `INTEGER` Unix epoch. Mixed
  representations across tables is a recurring source of wrong comparisons.
- `REAL` for money is wrong here for the same reason as everywhere else —
  store minor units as `INTEGER` (cents), since SQLite has no `DECIMAL`.
- Only `INTEGER PRIMARY KEY` aliases the internal rowid; `INT PRIMARY KEY`
  does not, and creates a second index. `AUTOINCREMENT` is rarely needed —
  it only prevents rowid reuse, at the cost of an extra table and writes.

## Step 3 — Foreign keys are off by default

- `PRAGMA foreign_keys = ON` must be issued **per connection**, before
  anything else — it is not a schema property, and it does not persist.
  Every pooled connection needs it. Most ORMs have a setting or an
  after-connect hook for this; confirm it's actually set rather than
  assuming.
- Without it, `REFERENCES` clauses are inert documentation and orphan rows
  accumulate silently.
- `ON DELETE CASCADE`/`SET NULL` likewise do nothing until the pragma is on.

## Step 4 — Concurrency: one writer at a time

- SQLite allows many concurrent readers but exactly one writer per database
  file. That's the model, not a tuning problem to solve.
- Enable WAL mode (`PRAGMA journal_mode = WAL`, persistent per database) so
  readers don't block the writer and vice versa. It's the single biggest
  practical win for a web workload.
- Set `PRAGMA busy_timeout = 5000` (or the value the app can tolerate) per
  connection — without it, a concurrent write returns `SQLITE_BUSY`
  immediately instead of waiting.
- Use `BEGIN IMMEDIATE` for any transaction that will write. A default
  deferred transaction takes a read lock first and must upgrade on the first
  write, which fails with `SQLITE_BUSY` under contention and cannot be
  retried safely mid-transaction.
- `PRAGMA synchronous = NORMAL` is the usual pairing with WAL (durable
  against process crashes, a small risk window on power loss);
  `synchronous = OFF` trades real durability and shouldn't be chosen
  silently.
- Never put a SQLite file on NFS or another network filesystem — its locking
  is not reliable there. That constraint is also why a multi-instance
  deployment usually needs a client/server database or Turso instead.

## Step 5 — Query planning and indexes

- `EXPLAIN QUERY PLAN <query>` is the check. `SCAN t` means a full scan;
  `SEARCH t USING INDEX …` means an index was used.
- `USING TEMP B-TREE FOR ORDER BY` means the sort isn't served by an index —
  usually fixed by an index matching the `ORDER BY`.
- Composite index rules match other engines: equality columns first, the
  range/sort column last, and the leftmost prefix serves shorter lookups, so
  don't add a redundant single-column index alongside.
- `SEARCH … USING AUTOMATIC COVERING INDEX` in a plan means SQLite built a
  throwaway index at runtime because a real one was missing — that's a
  standing signal to add the index.
- Index foreign key columns explicitly; SQLite does not create them for you,
  and an unindexed FK makes cascading deletes and joins scan.
- Run `ANALYZE` after a substantial data load so the planner has statistics;
  without it the plan can be chosen on guesses.
- Use FTS5 for text search rather than `LIKE '%term%'`, which cannot use an
  index at all.

## Step 6 — Schema changes and SQL dialect gaps

- `ALTER TABLE` supports only `RENAME TABLE`, `RENAME COLUMN`,
  `ADD COLUMN`, and `DROP COLUMN` (3.35+). There is no `ALTER COLUMN` and
  no `ADD CONSTRAINT` — changing a type or a constraint means the documented
  12-step recreate: new table, copy, drop, rename, all inside a transaction
  with foreign keys temporarily off.
- `ADD COLUMN … NOT NULL` requires a non-null default; adding one with a
  non-constant default is rejected.
- No `RIGHT`/`FULL OUTER JOIN` before 3.39, no stored procedures, and
  `TRUNCATE` doesn't exist (`DELETE FROM t` is the equivalent).
- `LIKE` is case-insensitive for ASCII only, and there is no built-in
  `UPPER`/`LOWER` for non-ASCII — a `_ci`-style comparison over non-English
  text needs `COLLATE NOCASE` (still ASCII-only) or normalization in the
  application.
- Integer division truncates (`3/2` is `1`) — cast when a fraction is meant.
- `UPSERT` (`ON CONFLICT … DO UPDATE`) and `RETURNING` (3.35+) exist and are
  usually better than a read-then-write round trip; check the pinned
  version.

## Step 7 — Backups, files, and deployment

- Never copy a live database file with `cp`/`rsync` — a concurrent write
  produces a corrupt copy, and in WAL mode the `-wal` file holds committed
  data the main file doesn't. Use `VACUUM INTO 'backup.db'` or the driver's
  online backup API.
- The `-wal` and `-shm` sidecar files are part of the database. Move or
  delete them together with it, and don't commit any of them to git.
- A database file on a container's ephemeral filesystem is deleted on every
  redeploy — it needs a mounted volume, and it still doesn't survive
  horizontal scaling. Say this out loud rather than letting it be
  discovered in production; the fix (a persistent volume, or a hosted libSQL
  target) is `devops-dev`'s lane.
- `VACUUM` rebuilds and compacts the file but takes a full lock and needs
  free space equal to the database size — schedule it, don't run it inline
  in a request.

## Step 8 — Self-check before reporting done

Confirm `foreign_keys` is enabled on every connection the app opens, WAL and
a `busy_timeout` are set, and every writing transaction uses
`BEGIN IMMEDIATE`. Confirm new tables are `STRICT` where the version allows,
booleans are `INTEGER` with a `CHECK`, timestamps use the project's single
representation, and money is stored in minor units. Confirm each new index
is backed by a query pattern with `EXPLAIN QUERY PLAN` showing it used, and
that no plan reports an automatic covering index. Confirm any schema change
beyond `ADD COLUMN` is written as a full table recreate inside a
transaction, and that the deployment target actually persists the file.

## What not to do

- Don't assume declared types are enforced — use `STRICT`, and add `CHECK`
  constraints where it isn't available.
- Don't store dates or booleans as their "natural" type; SQLite has neither.
- Don't rely on `REFERENCES` without `PRAGMA foreign_keys = ON` per
  connection.
- Don't leave write transactions as `BEGIN DEFERRED`, and don't run without
  WAL and a `busy_timeout` in a concurrent app.
- Don't call a query fast without `EXPLAIN QUERY PLAN`, and don't leave an
  automatic covering index in a plan unfixed.
- Don't attempt `ALTER COLUMN`/`ADD CONSTRAINT` — recreate the table.
- Don't `cp` a live database or ignore the `-wal`/`-shm` files.
- Don't run SQLite off a network filesystem, or across multiple app
  instances writing the same file.
- Don't write the engine-agnostic migration or access-layer code here —
  that's `database-schema-dev`/`backend-dev`.
