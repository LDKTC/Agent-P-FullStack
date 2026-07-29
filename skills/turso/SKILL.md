---
name: turso
description: Turso/libSQL-specific expertise — choosing among the three connection modes (remote HTTP, embedded replica with background sync, local file) and their consistency semantics, why a remote connection has no interactive multi-statement transaction and work must go through `batch`/`transaction`/stored-SQL round trips, avoiding N+1 chatty queries at the edge, the per-tenant "database per customer" pattern with groups and schema databases, running migrations across many databases, auth-token scoping/expiry, and platform limits. Use when `package.json` lists `@libsql/client` or a Drizzle/Prisma libSQL adapter, when a `libsql://` or `turso.io` URL, `TURSO_DATABASE_URL`/`TURSO_AUTH_TOKEN`, or a `turso` CLI config appears in the repo, or when asked directly (Thai or English) to ต่อ/ย้ายฐานข้อมูลไป Turso หรือ libSQL. Not a replacement for `skills/sqlite/SKILL.md` — all SQLite schema, type, index, and query rules still apply underneath; this skill covers only what changes when SQLite becomes a networked, replicated service.
---

# Turso (libSQL)

Turso is hosted libSQL, a fork of SQLite with a network protocol and
replication. That means `skills/sqlite/SKILL.md` is the base layer here —
type affinity and `STRICT` tables, integer booleans, ISO-8601 or epoch
timestamps, `EXPLAIN QUERY PLAN` index work, the restricted `ALTER TABLE`,
foreign keys needing to be enabled — all of it still holds. This skill is
only the delta: what changes once the database file is behind a network and
possibly replicated to the app process.

## Step 1 — Confirm Turso/libSQL is actually the target

Check `package.json` for `@libsql/client` (or `@libsql/client/web`), a
Drizzle `turso`/`libsql` driver, or `@prisma/adapter-libsql`; a
`libsql://…turso.io` URL; `TURSO_DATABASE_URL`/`TURSO_AUTH_TOKEN` in
`.env.example`; or a `turso` CLI config. If the project uses a plain local
`.db` file with `better-sqlite3` and no libSQL client, this is ordinary
SQLite — use that skill alone.

## Step 2 — Pick the connection mode deliberately

The client URL decides the semantics, and the three modes are not
interchangeable:

- **Remote** (`url: 'libsql://…'`): every query is an HTTP round trip to
  the primary or a replica. Simplest and stateless — correct for serverless
  and edge functions, but latency-sensitive.
- **Embedded replica** (`url: 'file:local.db', syncUrl: 'libsql://…'`):
  reads hit a local SQLite file at microsecond latency; writes go to the
  remote primary and are then pulled back. This is the mode that makes Turso
  interesting, and the one with the consistency caveat below.
- **Local file** (`url: 'file:…'`): plain SQLite, usually for tests and
  local development. Keep dev and production on the same *schema* path even
  when the modes differ.

State which mode the code assumes. A handler written against an embedded
replica's read latency behaves very differently when deployed with a remote
URL, and nothing in the type system catches the swap.

## Step 3 — Embedded replica consistency

- Reads from an embedded replica are **stale** until a sync completes. A
  write followed immediately by a local read can miss the row that was just
  written unless the client's write-then-read guarantee is used (the write
  goes to the primary and the replica is refreshed) — verify the behavior in
  the client version pinned here rather than assuming read-your-writes.
- Call `sync()` on a schedule (`syncInterval`) or explicitly after a write
  whose result the next read must see. A UI that shows "saved" and then
  renders stale data is the classic symptom.
- Sync pulls the whole delta since the last sync; a replica that syncs
  rarely does more work per sync and holds more local disk.
- Serverless functions with ephemeral filesystems lose the replica between
  cold starts, so the first request after a start pays a full bootstrap —
  embedded replicas fit long-lived processes (a container, a VM, a desktop
  app) far better than short-lived invocations.

## Step 4 — Transactions and round trips

- Over a remote connection there is no long-lived interactive transaction
  held open across application logic — each statement is a round trip, and
  an idle transaction holds a write lock on the primary the whole time. Use
  the client's `batch()` for a set of statements applied atomically in one
  round trip, and `transaction()` only for a short, tightly-scoped block.
- SQLite's single-writer rule still applies to the primary. Keep write
  transactions short; a transaction that waits on application code or an
  external API blocks every other writer.
- N+1 queries that were nearly free against a local file become N network
  round trips here. Join, or use a batch — this is the most common
  performance regression when a project migrates from local SQLite to
  Turso.
- Prefer `RETURNING` over insert-then-select to halve round trips.
- Deploy the primary (or its group's region) near the writers, not just near
  the users — reads can be served from replicas, writes cannot.

## Step 5 — Multi-database and per-tenant patterns

- Turso's model makes "one database per tenant" practical, unlike a single
  large Postgres. It buys hard isolation and per-tenant restore, at the cost
  of running every migration N times and holding N connections.
- If that's the design, migrations need a runner that iterates every
  database and records per-database version state, with a report of which
  ones failed — not a one-shot script assumed to have worked. Schema
  databases (a parent schema propagated to member databases) exist for this;
  confirm the account's plan supports them before designing around it.
- Databases live in **groups** that determine regions and replicas. Adding a
  location replicates every database in the group.
- Never mint a client-side token that can reach all tenants' databases.
  Tokens are scoped (full-access vs read-only, per database or per group)
  and expire — scope to the narrowest thing that works, keep the token
  server-side, and store it as a secret through `devops-dev`'s env wiring,
  never in a client bundle.

## Step 6 — Local development and testing

- Use `turso dev` (a local libSQL server) or a plain `file:` URL for tests,
  so the suite doesn't depend on a network and a shared remote database.
- Keep migrations in the repo as ordered SQL/ORM migration files applied by
  a runner — a schema changed by hand through the CLI or dashboard leaves
  environments silently divergent, and there's no diff to review.
- Point verification at the real target: run
  `skills/dev-testing/SKILL.md`'s probe against the mode the environment
  actually uses, and read the database delta from the same connection the
  app writes through — a query to the primary while the app reads a replica
  can show a row the app can't see yet.

## Step 7 — Platform limits and cost shape

- Billing counts rows read and written plus storage, so a `SELECT *` over a
  wide table or a full-table scan is a cost line, not only a latency one.
  This makes the SQLite index work in `EXPLAIN QUERY PLAN` directly
  financial.
- Per-database size, per-query response, and concurrent-connection limits
  exist and vary by plan — check the current limits for the plan in use
  rather than assuming; hitting one appears as a runtime error, not a
  degradation.
- Long-running analytical queries and very large transactions are a poor fit
  for the HTTP protocol; move that work to an export or a different store.
- Extension and pragma support is not identical to stock SQLite — verify a
  pragma actually takes effect over the remote protocol instead of assuming
  it was applied.

## Step 8 — Self-check before reporting done

Confirm the connection mode is explicit and the code's latency and
consistency assumptions match it. Confirm any write whose result is read
back on an embedded replica either syncs or reads through the primary.
Confirm multi-statement writes use `batch`/`transaction` rather than a
sequence of independent round trips, and that no request path issues a query
per row. Confirm auth tokens are server-side and scoped to the narrowest
database/permission, and that migrations are file-based and applied to every
database in a per-tenant setup with a per-database result. Then re-check the
schema itself against `skills/sqlite/SKILL.md`'s self-check.

## What not to do

- Don't assume an embedded replica read reflects the write that just
  happened.
- Don't hold an interactive transaction open across application logic on a
  remote connection.
- Don't port a local-SQLite N+1 loop unchanged — each iteration is now a
  network round trip and a billed read.
- Don't ship an auth token to the client, and don't use one full-access
  token across all tenant databases.
- Don't change schema through the dashboard or CLI instead of a committed
  migration, and don't assume a per-tenant migration succeeded everywhere
  without a per-database result.
- Don't put an embedded replica on an ephemeral serverless filesystem and
  expect the local-read advantage.
- Don't restate SQLite's type/index/transaction rules here — they live in
  `skills/sqlite/SKILL.md` and still apply.
