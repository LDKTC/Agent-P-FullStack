---
name: supabase
description: Supabase-specific expertise — the key model as a trust boundary (the publishable/anon key is public and ships to the browser; the secret/`service_role` key bypasses Row Level Security and must never leave the server), RLS as the actual authorization layer with `USING` vs `WITH CHECK` per operation and indexed policy predicates, validating a session server-side with `getUser()` rather than trusting a decoded token or user-writable `user_metadata`, CLI-based migrations kept in the repo instead of dashboard edits, Edge Functions, Storage bucket policies and signed URLs, and Realtime. Use when `package.json` lists `@supabase/supabase-js`, `@supabase/ssr`, or `@supabase/auth-helpers-*`, when a `supabase/` directory, `config.toml`, or `SUPABASE_URL`/`SUPABASE_ANON_KEY` env vars are present, or when asked directly (Thai or English) to ต่อ/ตั้งค่า Supabase, เขียน RLS policy, หรือทำ auth ด้วย Supabase. Not a replacement for `skills/postgres/SKILL.md` — Supabase is Postgres, so all of its type, index, EXPLAIN, and locking rules apply underneath; this skill is only what Supabase adds on top.
---

# Supabase

Supabase is Postgres plus an auto-generated API, an auth service, storage,
and Deno-based Edge Functions. `skills/postgres/SKILL.md` is the base layer —
types, indexes, `EXPLAIN (ANALYZE, BUFFERS)`, MVCC and locking, pooling and
migration safety all apply unchanged. This skill covers only what Supabase
adds, and most of that is about a trust boundary that does not exist in a
normal server-only Postgres deployment: **the database is reachable directly
from the browser.**

## Step 1 — Confirm Supabase is actually in play

Check `package.json` for `@supabase/supabase-js`, `@supabase/ssr`, or an
`auth-helpers` package; a `supabase/` folder with `config.toml` and
`migrations/`; or `SUPABASE_URL`/`SUPABASE_ANON_KEY` in `.env.example`. If
the project only talks to a Postgres connection string that happens to be
hosted by Supabase, with no client SDK and no RLS in play, treat it as plain
Postgres.

## Step 2 — The two keys are a security boundary

- The **anon / publishable** key is designed to be public. It ships in the
  browser bundle and grants exactly what RLS policies allow — no more. Its
  exposure is not a vulnerability; missing RLS is.
- The **`service_role` / secret** key **bypasses RLS entirely**. It belongs
  only in server-side code (a route handler, an Edge Function, a job) and
  only in a server-only environment variable. Never under `NEXT_PUBLIC_`/
  `VITE_`/`EXPO_PUBLIC_`, never in a client component, never in a mobile
  app, never committed.
- A leaked `service_role` key is a full database compromise. Treat finding
  one in client-reachable code as a stop-everything finding, route it
  through `security-auditor`, and rotate the key — deleting the line from
  the source does not un-publish it if it ever shipped or was committed.
- Instantiate two clearly named clients (browser vs server/admin) rather
  than one that picks a key by condition; the conditional is exactly where
  the mistake happens.

## Step 3 — RLS is the authorization layer

- Enable RLS on **every** table in an exposed schema, including join tables,
  lookup tables, and views' underlying tables. A table without RLS in the
  `public` schema is readable and writable by anyone holding the public key.
  Supabase's advisors flag this; check them rather than assuming.
- RLS default-denies once enabled: no policy means no access. Write one
  policy per operation (`select`, `insert`, `update`, `delete`) rather than
  a single `for all` — the four cases genuinely differ.
- `USING` filters the rows a statement may see or touch; `WITH CHECK`
  constrains the rows it may produce. An `update` policy with only `USING`
  lets a user edit their own row into someone else's ownership — `update`
  needs both, `insert` needs `WITH CHECK`.
- Compare against `(select auth.uid())` and index the column the policy
  filters on. An unindexed `user_id` in a policy predicate turns every
  query into a scan, and wrapping `auth.uid()` in a `select` lets the
  planner evaluate it once instead of per row.
- Policies are the enforcement; a `.eq('user_id', currentUser.id)` filter in
  client code is a convenience the caller can simply remove. Never rely on
  the client's filter for authorization.
- Anything requiring privileged logic goes in a `security definer` function
  with a pinned `search_path` and a narrow signature — not in a broader
  policy. Be deliberate: a `security definer` function is a hole you are
  opening on purpose.
- Test policies from the perspective of a real anon and a real authenticated
  user, not with the service key, which sails past all of them.

## Step 4 — Auth

- On the server, validate the session with `supabase.auth.getUser()`, which
  verifies the JWT against the auth server. `getSession()` returns whatever
  is in storage/cookies without revalidating — fine for rendering a UI
  state, not for an authorization decision.
- `user_metadata` is writable by the user. Roles, plan tiers, and
  entitlements go in `app_metadata` or a table governed by RLS, never in
  `user_metadata` — a role check against it is a privilege-escalation bug.
- Configure the redirect URL allowlist explicitly and keep email
  confirmation on; an open redirect in the auth callback is a real account
  takeover path.
- Mirror auth users into a `profiles` table (trigger on `auth.users` insert)
  rather than joining against the `auth` schema from application queries.
- In SSR frameworks, use `@supabase/ssr` with cookie storage so the server
  and client see the same session, and refresh in middleware — a
  `localStorage`-based session is invisible to the server render and shows
  up as a flash of logged-out UI or a redirect loop.
- Social sign-in providers (including Google) are configured here, but the
  OAuth/OIDC correctness rules — ID token verification, `state`/nonce,
  scopes, refresh handling — are `skills/google-auth/SKILL.md`.

## Step 5 — Migrations and local development

- Schema lives in `supabase/migrations/` in the repo, generated with
  `supabase migration new` / `supabase db diff` and applied with
  `supabase db push` or CI. Changes clicked into the dashboard leave no
  diff to review, don't reach other environments, and get silently
  overwritten by the next push.
- RLS policies, triggers, functions, and grants are schema — they belong in
  migrations too, not applied by hand.
- Run `supabase start` locally so tests and probes hit a local stack rather
  than a shared cloud project. `supabase db reset` + seed data gives a
  reproducible state, which is what `skills/dev-testing/SKILL.md`'s
  database-delta check needs.
- Never point automated tests or a destructive script at a production
  project — check which project ref is linked before running anything that
  writes.

## Step 6 — Edge Functions, Storage, Realtime

- **Edge Functions** run on Deno, not Node: `node:` builtins and npm
  packages need the supported import form, and the runtime is not the one
  `skills/nodejs/SKILL.md` assumes. Read secrets from the function's own env
  (set via the CLI), keep JWT verification on unless the endpoint is
  genuinely public, and remember an inbound webhook can't present a user
  JWT — verify the provider's signature instead.
- **Storage** buckets are private by default and access is governed by
  policies on `storage.objects`, in the same RLS model as tables. Serve
  private files with time-limited signed URLs, not by flipping the bucket
  public; validate content type and size on upload rather than trusting the
  client's declared type.
- **Realtime** subscriptions respect RLS for Postgres changes, but only once
  the publication and RLS are configured — enabling replication on a table
  with no RLS broadcasts every row change to every subscriber. Subscribe to
  the narrowest filter, and unsubscribe on unmount or the channel leaks.
- The auto-generated PostgREST API exposes every table in `public`; keep
  internal tables in a non-exposed schema instead of relying on nobody
  guessing the name.

## Step 7 — Performance and cost shape

- The client library is PostgREST — `select('*')` fetches every column
  including large text/JSON; select the columns actually used.
- Use nested selects for related rows in one request instead of a query per
  row; an N+1 here is N HTTP round trips.
- Paginate with `range()`/keyset rather than fetching a whole table into the
  browser and slicing it, and set an explicit `count` strategy only when the
  count is actually displayed — an exact count on a large table is a scan.
- Serverless deployments should connect through the pooler, with the
  transaction-mode caveats in `skills/postgres/SKILL.md` (no session state,
  no named prepared statements); use the direct connection for migrations.
- Run Supabase's advisors (security and performance) after schema changes —
  unindexed foreign keys, missing RLS, and mutable `search_path` functions
  surface there.

## Step 8 — Self-check before reporting done

Confirm no `service_role`/secret key is reachable from client code or a
public env var. Confirm every new table has RLS enabled with per-operation
policies, that `update` policies carry both `USING` and `WITH CHECK`, and
that policy predicate columns are indexed. Confirm server-side authorization
uses `getUser()` and reads roles from `app_metadata` or an RLS-governed
table, never `user_metadata`. Confirm every schema and policy change exists
as a committed migration and was applied to a local stack first. Confirm the
advisors are clean for what changed, then verify the feature end to end with
`skills/dev-testing/SKILL.md` as a non-privileged user — a query that works
with the service key proves nothing about what a real user can do.

## What not to do

- Don't put the `service_role`/secret key anywhere the browser can reach it,
  and don't treat a leak as fixed without rotating.
- Don't leave RLS off on any table in an exposed schema, and don't rely on a
  client-side `.eq()` filter for authorization.
- Don't write an `update` policy with `USING` alone.
- Don't store or read roles/entitlements in `user_metadata`.
- Don't make an authorization decision from `getSession()` on the server.
- Don't change schema, policies, or functions through the dashboard instead
  of a migration file.
- Don't test policies with the service key, and don't run destructive
  scripts against a linked production project.
- Don't make a Storage bucket public to solve an access problem — sign the
  URL.
- Don't restate Postgres's own type/index/locking rules here; they live in
  `skills/postgres/SKILL.md`.
