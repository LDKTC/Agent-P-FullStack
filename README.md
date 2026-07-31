# Agent-P-FullStack

**Agent-P** — a Full Stack web dev agent+skill roster that works across
Claude Code, OpenAI Codex, GitHub Copilot, and any other tool that reads the
[AGENTS.md](AGENTS.md) convention.

One department head plus thirteen specialists, each scoped to a slice of the
stack (UI/UX pattern research, frontend, backend, database schema, devops,
third-party integration, e2e testing, code review, security auditing,
performance auditing, debugging, documentation), each written to stay inside
its lane and hand off an explicit contract to the specialist that owns the
next slice — so a schema decision doesn't leak into UI code, and an API
shape doesn't get invented twice.

## What's inside

- **`agents/`** — 14 Claude Code subagents: `fullstack-head` (routes/reviews),
  `fullstack-senior-dev` (stack/version detection briefing),
  `ui-ux-researcher` (UI/UX pattern research), `frontend-dev`,
  `backend-dev`, `database-schema-dev`, `devops-dev`, `api-integration-dev`,
  `fullstack-tester`, `code-reviewer` (diff/PR review), `security-auditor`
  (exploit-focused OWASP audit), `performance-auditor` (Core Web Vitals
  audit, measurement-honest), `debug-specialist` (reproduce/root-cause/fix a
  reported bug), `documentation-architect` (README/architecture/API docs).
- **`skills/stack-briefing/`** — the stack-detection routine as a standalone
  skill, for running inline without subagent dispatch.
- **`skills/html-css/`** — framework-agnostic semantic HTML & modern CSS
  authoring/review (accessibility, responsive layout, CSS conventions), for
  static pages/email templates/landing pages or auditing existing markup —
  the craft layer `frontend-dev` builds framework-specific work on top of.
- **`skills/react/`** — React-specific expertise (Rules of Hooks and
  dependency arrays, effect misuse, list keys, memoization judgment,
  data-fetching waterfalls), active when the project's stack is React —
  layered on `frontend-dev` and `skills/html-css/`.
- **`skills/tailwind/`** — utility-first Tailwind CSS authoring/review
  (token scale over arbitrary values, responsive/state variants, class-
  string maintainability), active when the project uses Tailwind — works
  with any framework, layered on `skills/html-css/`.
- **`skills/dev-testing/`** — development-oriented testing: verify a change
  against the running app by starting the service, sending one request or
  driving one user-level browser task, and reading the HTTP response, the
  server/console log, and the database delta *together* as a single verdict
  — the check that catches a `201` which persisted nothing, or a page that
  renders correctly while logging an uncaught error. Run per vertical slice
  during implementation, not once at the end.
- **`skills/dry-and-cor/`** — two structural rules. **DRY** treated as
  knowledge rather than characters: the test for real duplication, the
  full-stack cases that matter (a business rule restated across layers, an
  API type hand-maintained on both sides of the seam), and the false-DRY
  traps — client *and* server validation are both required, tests are
  allowed to repeat, and a helper that grew a mode flag should have stayed
  two functions. **Chain of Responsibility** for the middleware/guard/
  interceptor/error pipelines every web stack already has: one decision per
  handler, order as a contract, terminate-or-pass-never-both, and no request
  falling off the end into a silent success. The same five rules are then
  applied to *routing work* across the roster — most specific persona wins,
  one owner per unit, escalation is the chain working, and a task no persona
  covers gets said out loud rather than absorbed — which is how
  `fullstack-head` dispatches.
- **`skills/typescript/`** — the typing layer that cuts across every other
  skill: strict-mode settings, the rule that types are erased at runtime so
  every boundary (request body, `process.env`, `JSON.parse`, third-party
  response) is validated by a schema with the type derived from it,
  `unknown` + narrowing instead of `any`, discriminated unions instead of
  optional-flag soup, and generics only where a real input↔output
  relationship exists.
- **`skills/nodejs/`** — the Node runtime layer under `backend-dev`: what
  blocks the single-threaded event loop, async correctness (floating
  promises, sequential `await` in loops, `async` callbacks whose rejections
  are dropped), streams and backpressure, ESM vs CommonJS, env-validated
  config, graceful SIGTERM shutdown, and Node-specific security.
- **`skills/electron/`** — Electron desktop apps: the main/renderer/preload
  process model treated as a trust boundary, the `contextIsolation` /
  `nodeIntegration: false` / `sandbox` baseline, a narrow `contextBridge`
  surface with every `ipcMain.handle` argument validated,
  `setWindowOpenHandler` and `shell.openExternal` allowlisting, and
  packaging/signing/auto-update. The renderer is still a web UI, so
  `html-css`/`react` apply inside it and `nodejs` applies in main.
- **`skills/flutter/`** — Flutter/Dart apps: `build()` kept pure and cheap
  (no Futures or controllers created there), `const` and rebuild scope,
  disposal plus the `mounted` check after every `await`, the
  constraints-down/sizes-up layout model and the overflow/unbounded errors
  it produces, builder-based lists, and per-platform assets and permissions.
  Explicitly *not* layered on `html-css`/`react`/`tailwind` — there is no
  DOM and no CSS.

### Database and platform skills

Each is `database-schema-dev`'s and `backend-dev`'s engine-specific
expertise layer, and they stack rather than repeat each other:

- **`skills/mysql/`** — MySQL/MariaDB-specific database expertise (data
  types and charset/collation, index design and EXPLAIN-driven query
  tuning, transaction isolation and locking), active when the project's
  database is MySQL or MariaDB.
- **`skills/postgres/`** — PostgreSQL: `timestamptz`/`numeric`/`jsonb`
  typing, index selection across btree/GIN/partial/expression/covering,
  `EXPLAIN (ANALYZE, BUFFERS)` and sargability, MVCC consequences (bloat,
  long transactions blocking vacuum, `FOR UPDATE SKIP LOCKED` queues),
  pooling and PgBouncer transaction-mode limits, and lock-safe migrations.
- **`skills/sqlite/`** — SQLite: dynamic typing and `STRICT` tables, the
  missing boolean/date types, foreign keys being off per connection, the
  single-writer model with WAL and `busy_timeout`, `BEGIN IMMEDIATE`,
  `EXPLAIN QUERY PLAN`, the restricted `ALTER TABLE`, and safe backups of a
  file-backed database.
- **`skills/turso/`** — Turso/libSQL, layered on `sqlite`: choosing among
  remote, embedded-replica, and local modes; replica staleness after a
  write; no interactive transactions over HTTP so writes go through
  `batch`; N+1 loops becoming network round trips *and* billed reads; the
  per-tenant database pattern and migrating across many databases.
- **`skills/supabase/`** — Supabase, layered on `postgres`: the anon key
  being public by design while the `service_role` key bypasses RLS entirely,
  RLS as the actual authorization layer (`USING` vs `WITH CHECK`, indexed
  policy predicates, a client-side `.eq()` filter is not authorization),
  `getUser()` over `getSession()` for server decisions, roles in
  `app_metadata` never `user_metadata`, CLI migrations instead of dashboard
  edits, plus Edge Functions, Storage, and Realtime.
- **`skills/firebase/`** — Firebase: the public client config vs the
  rule-bypassing Admin SDK, Security Rules as authorization with the rule
  that *rules are not filters*, Firestore modeling for the read shape (1 MiB
  documents, hot-document write limits, composite indexes), the per-document
  cost model that makes an unbounded query a bill, idempotent Cloud
  Functions, `verifyIdToken` with custom claims, and the Emulator Suite.
- **`skills/google-auth/`** — "Sign in with Google" at the protocol level:
  authorization code + PKCE (never implicit, never a client secret in a
  public client), the ID token vs access token distinction, full server-side
  verification of signature/`aud`/`iss`/`exp`/nonce/`email_verified` with
  `sub` as the stable user key, `state` and exact redirect URIs, minting
  your own session cookie instead of storing Google tokens in the browser,
  minimal and incremental scopes, refresh-token handling, and the account
  takeover path in linking on an unverified email.

- **`AGENTS.md`** — persona roster + routing table, for Codex/Cursor/Aider/
  Windsurf/any tool that reads a single root instructions file.
- **`.github/copilot-instructions.md`** — the same routing, in GitHub
  Copilot's own convention.

## Install

### Claude Code

```
/plugin marketplace add LDKTC/Agent-P-FullStack
/plugin install agent-p-fullstack@agent-p-fullstack
```

Or from a local checkout:

```
/plugin marketplace add "d:\AI Project\Agent-P-FullStack"
/plugin install agent-p-fullstack@agent-p-fullstack
```

This loads all 14 agents and the 18 skills — `agent-flow`, `stack-briefing`,
`dev-testing`, `dry-and-cor`, `html-css`, `react`, `tailwind`,
`typescript`, `nodejs`, `electron`, `flutter`, `mysql`, `postgres`,
`sqlite`, `supabase`, `turso`, `firebase`, and `google-auth` — into any
project you work on with Claude Code. Each technology skill activates from
the project's own files (a dependency, a config file, a connection string),
so an unrelated stack simply never triggers it.

### OpenAI Codex (or Cursor, Aider, Windsurf, and other AGENTS.md-aware tools)

These tools don't install subagents — copy [AGENTS.md](AGENTS.md) together
with the [`agents/`](agents/) and [`skills/`](skills/) folders into the
target project, preserving the same relative layout, so AGENTS.md's links
(including its stack-briefing fallback) resolve. If you only want the
routing table itself, strip the links before appending the text to an
existing root `AGENTS.md`.

### GitHub Copilot

Copy [`.github/copilot-instructions.md`](.github/copilot-instructions.md)
together with `AGENTS.md`, `agents/`, and `skills/` into the target project
(preserving the relative paths it links through — `../AGENTS.md`,
`../agents/`, `../skills/`), or merge just the instructions text into an
existing `copilot-instructions.md` and drop the links that would otherwise
dangle.

## Prior art

The `dev-testing` skill and the up-front contract plan in `fullstack-head`
adapt ideas from [FullStack-Agent](https://github.com/mnluzimu/FullStack-Agent)
(Lu et al.) — specifically its development-oriented testing approach, where
a backend probe correlates the HTTP response with the service log and the
resulting database rows, and a frontend probe drives a real browser through
a user-level task and treats an uncaught console error as failure. Nothing
is vendored from it; that project is a Python research framework with its
own model and benchmark, while this repo is prose guidance for coding
agents.

## Known caveat

If a target project already has same-named agents or skills from another
source, both will exist side by side. Rename or remove the duplicate rather
than assuming one silently wins.
