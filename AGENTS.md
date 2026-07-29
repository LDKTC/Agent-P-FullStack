# Agent-P — Full Stack

Instructions for any AI coding agent (OpenAI Codex, Cursor, Windsurf, Aider,
GitHub Copilot's AGENTS.md support, or a human skimming for context) working
on a full-stack web project with this roster available. Claude Code reads
the richer per-agent files in [`agents/`](agents/) directly as subagents;
tools that only read a single root instructions file should follow this one.

## What this is

A Full Stack web dev "department": one department head and thirteen
specialists, each scoped to a slice of the stack, each written so it stays
inside its lane and hands off to the specialist that owns the next slice.
Adopt the persona that matches the task at hand rather than doing everything
as one undifferentiated "full stack" pass — the lane boundaries below exist
specifically to stop schema decisions leaking into UI code, or API contracts
getting invented twice by two different layers.

## Roster and routing

| Persona | Owns | Full definition |
|---|---|---|
| **fullstack-head** | Claude-Code-only: dispatches the rest of this roster as subagents, reviews output, reports one consolidated result. Tools without subagent dispatch can't run this persona directly — see "More than one row applies" below. | [agents/fullstack-head.md](agents/fullstack-head.md) |
| **fullstack-senior-dev** | Detects the real stack/versions and existing conventions; produces a briefing every other persona below treats as a hard constraint. Run this **first** on an unfamiliar repo. Read-only. | [agents/fullstack-senior-dev.md](agents/fullstack-senior-dev.md) |
| **ui-ux-researcher** | Researches UI/UX interaction/layout/information-architecture patterns corroborated across independent high-traffic sites plus an authoritative UX source; hands frontend-dev an implementation-ready recommendation. Read-only. | [agents/ui-ux-researcher.md](agents/ui-ux-researcher.md) |
| **frontend-dev** | UI components, pages, client-side state, styling. | [agents/frontend-dev.md](agents/frontend-dev.md) |
| **backend-dev** | API routes/handlers, business logic, input validation. | [agents/backend-dev.md](agents/backend-dev.md) |
| **database-schema-dev** | Tables/columns/indices/relations and migration files. | [agents/database-schema-dev.md](agents/database-schema-dev.md) |
| **devops-dev** | CI/CD, Dockerfiles, deploy config, env/secrets wiring (never values). | [agents/devops-dev.md](agents/devops-dev.md) |
| **api-integration-dev** | Third-party API/SDK clients (payments, auth providers, email/SMS). | [agents/api-integration-dev.md](agents/api-integration-dev.md) |
| **fullstack-tester** | End-to-end verification that a feature works across all layers, with real evidence. | [agents/fullstack-tester.md](agents/fullstack-tester.md) |
| **code-reviewer** | Reviews a diff/PR for correctness, readability, architecture, security-awareness, performance-awareness before merge. Read-only. | [agents/code-reviewer.md](agents/code-reviewer.md) |
| **security-auditor** | Exploit-focused security audit (OWASP Top 10, STRIDE from trust boundaries) with severity + proof-of-concept + fix. Read-only. | [agents/security-auditor.md](agents/security-auditor.md) |
| **performance-auditor** | Core Web Vitals/loading/rendering/network audit; measurement-honest (Quick vs. Deep mode). Read-only. | [agents/performance-auditor.md](agents/performance-auditor.md) |
| **debug-specialist** | Reproduces, root-causes, and fixes a specific reported bug/error/crash — the one worker allowed to cross lanes for its own fix. | [agents/debug-specialist.md](agents/debug-specialist.md) |
| **documentation-architect** | Writes/updates persisted docs (README, architecture notes, API reference) grounded in the actual code and other workers' reports. | [agents/documentation-architect.md](agents/documentation-architect.md) |

Lighter-weight alternative to dispatching `fullstack-senior-dev` as a
subagent: [skills/stack-briefing/SKILL.md](skills/stack-briefing/SKILL.md)
walks the same detection routine inline, for tools with no subagent
mechanism.

For raw HTML/CSS authoring or review with no framework in front of it (a
static page, an email template, a landing page) — or auditing the semantic
structure/accessibility of existing markup — use
[skills/html-css/SKILL.md](skills/html-css/SKILL.md) directly instead of
dispatching `frontend-dev`, which assumes a framework and existing
conventions are already in place and builds on top of this craft.

For React-specific work — Rules of Hooks, dependency arrays, effect misuse,
list keys, memoization judgment, data-fetching waterfalls — use
[skills/react/SKILL.md](skills/react/SKILL.md) whenever `package.json`
lists `react`/`react-dom`/`next`, or the work is `.jsx`/`.tsx`. This is
`frontend-dev`'s React-specific expertise layer, not a replacement for it;
[skills/html-css/SKILL.md](skills/html-css/SKILL.md) still applies
underneath the JSX.

For Tailwind CSS work — composing utilities instead of custom CSS, the
project's token scale over arbitrary values, mobile-first responsive
prefixes, state variants, and keeping class strings maintainable — use
[skills/tailwind/SKILL.md](skills/tailwind/SKILL.md) whenever
`package.json` lists `tailwindcss` (or a `tailwind.config`/`@theme` block
exists). Applies in any framework, not just React, layered on the same
[skills/html-css/SKILL.md](skills/html-css/SKILL.md) markup foundation.

For verifying that a change actually works — starting the service, sending
one request or driving one browser task, and reading the HTTP response, the
server/console log, and the database delta together as a single verdict —
use [skills/dev-testing/SKILL.md](skills/dev-testing/SKILL.md). It applies
during implementation, not only at the end: a 2xx that wrote no row, or a
page that rendered with an uncaught console error, is a failure that only
this three-signal check catches. `fullstack-tester` and `debug-specialist`
run it directly; `backend-dev` probes its own endpoint before handing over;
`frontend-dev` has no shell and instead states the browser task for someone
else to run.

For duplication and handler-chain judgment —
[skills/dry-and-cor/SKILL.md](skills/dry-and-cor/SKILL.md) — use it when a
rule is about to exist in two layers, when deciding whether to extract a
shared abstraction, or when adding to/reordering a middleware, guard,
interceptor, or error-handler chain. DRY here is about knowledge rather than
characters (two blocks that change for different reasons aren't duplication,
and server-side validation is never redundant with the client's); CoR covers
one decision per handler, order as a contract, explicit termination, and no
silent fall-through. `backend-dev` and `frontend-dev` apply it while
implementing; `code-reviewer` reviews against it.

For TypeScript-specific work — strict-mode settings, the rule that types are
erased so every boundary (request body, `process.env`, `JSON.parse`,
third-party response) needs schema validation with the type derived from it,
disciplined `any`/`as`/`!` use, and discriminated unions over optional-flag
soup — use [skills/typescript/SKILL.md](skills/typescript/SKILL.md) whenever
a `tsconfig.json` exists. It cuts across every layer rather than sitting
under one persona.

For Node.js runtime work — what blocks the single-threaded event loop, async
correctness and floating promises, streams and backpressure, ESM/CJS
interop, env-validated config, graceful SIGTERM shutdown, and Node-specific
security (`child_process`, path traversal) — use
[skills/nodejs/SKILL.md](skills/nodejs/SKILL.md) whenever the server runtime
is Node. This is `backend-dev`'s runtime layer, not a replacement for its
route/validation/business-logic work.

## Database and platform skills

Each of these is `database-schema-dev`'s and `backend-dev`'s
engine-specific expertise layer — neither agent stops being
engine-agnostic; these are what they draw on once the engine is known.
They also stack, so read the base skill with the one on top of it:

| Skill | Use when | Layered on |
|---|---|---|
| [skills/mysql/SKILL.md](skills/mysql/SKILL.md) — data types and charset/collation, index design, EXPLAIN-driven tuning, transaction isolation and gap locks | `mysql2`/`mysql` in package.json, an ORM dialect/provider of `mysql`, a mysql/mariadb compose image, or a `mysql://` URL | — |
| [skills/postgres/SKILL.md](skills/postgres/SKILL.md) — `timestamptz`/`numeric`/`jsonb` typing, btree/GIN/partial/expression indexes, `EXPLAIN (ANALYZE, BUFFERS)`, MVCC and vacuum, PgBouncer transaction-mode limits, lock-safe migrations | `pg`/`postgres`/`@neondatabase/serverless`, an ORM provider of `postgresql`, a postgres compose image, or a `postgres://` URL | — |
| [skills/sqlite/SKILL.md](skills/sqlite/SKILL.md) — dynamic typing and `STRICT` tables, no native boolean/date, foreign keys off per connection, single-writer + WAL + `busy_timeout`, `BEGIN IMMEDIATE`, `EXPLAIN QUERY PLAN`, the 12-step table recreate | `better-sqlite3`/`sqlite3`/`node:sqlite`/`expo-sqlite`, an ORM provider of `sqlite`, or a `.db`/`file:` database | — |
| [skills/turso/SKILL.md](skills/turso/SKILL.md) — remote vs embedded-replica vs local connection modes, replica staleness, no interactive transactions over HTTP, per-tenant database patterns, token scoping | `@libsql/client`, a `libsql://` URL, or `TURSO_DATABASE_URL` | `sqlite` |
| [skills/supabase/SKILL.md](skills/supabase/SKILL.md) — the anon vs `service_role` key boundary, RLS as the authorization layer (`USING` vs `WITH CHECK`), `getUser()` over `getSession()`, `app_metadata` not `user_metadata` for roles, CLI migrations over dashboard edits, Edge Functions/Storage/Realtime | `@supabase/supabase-js`/`@supabase/ssr`, a `supabase/` directory, or `SUPABASE_URL` | `postgres` |
| [skills/firebase/SKILL.md](skills/firebase/SKILL.md) — client config is public but the Admin SDK bypasses every rule, Security Rules are not filters, Firestore modeling for the read shape and hot-document limits, per-read cost, idempotent Cloud Functions, `verifyIdToken` and custom claims, the Emulator Suite | `firebase`/`firebase-admin`, `firebase.json`, `firestore.rules`, or `google-services.json` | — |

## Desktop and mobile skills

For Electron desktop work — the main/renderer/preload process model as a
trust boundary, the `contextIsolation`/`nodeIntegration`/`sandbox` baseline,
a narrow `contextBridge` surface with validated `ipcMain.handle` arguments,
`shell.openExternal` allowlisting, and packaging/signing/auto-update — use
[skills/electron/SKILL.md](skills/electron/SKILL.md) whenever `package.json`
lists `electron`/`electron-builder`/`electron-forge`. The renderer is still
a web UI, so `frontend-dev`, `skills/html-css/SKILL.md`, and
`skills/react/SKILL.md` still apply inside it, and the main process is Node,
so `skills/nodejs/SKILL.md` applies there.

For Flutter/Dart app work — a pure and cheap `build()`, `const` and rebuild
scope, disposal and the `mounted` check after every `await`, the
constraints-down/sizes-up layout model, builder-based lists, and
per-platform assets/permissions — use
[skills/flutter/SKILL.md](skills/flutter/SKILL.md) whenever `pubspec.yaml`
exists. Flutter has no DOM and no CSS, so `skills/html-css/SKILL.md`,
`skills/react/SKILL.md`, and `skills/tailwind/SKILL.md` deliberately do
**not** carry over — `ui-ux-researcher`'s recommendations and
`backend-dev`'s API contracts still do.

## Auth

For "Sign in with Google" and OAuth 2.0/OIDC generally — picking
authorization code + PKCE over the deprecated implicit flow, never shipping
a client secret in a public client, the ID token vs access token
distinction, full server-side ID token verification (signature, `aud`,
`iss`, `exp`, nonce, `email_verified`, `sub` as the stable key),
`state`/redirect-URI safety, minimal and incremental scopes, refresh token
handling, and the account-linking hazard on unverified email — use
[skills/google-auth/SKILL.md](skills/google-auth/SKILL.md). It's the
protocol layer under `api-integration-dev`'s SDK wiring; findings on the
trust boundary still route to `security-auditor`. This applies to Google
sign-in configured through Supabase Auth or Firebase Auth too.

## More than one row applies

`fullstack-head` enforces this in Claude Code by dispatching subagents; a
tool with no subagent mechanism has no equivalent dispatch step. Work
through each relevant specialist section above yourself, in the same
session, in dependency order (stack briefing first, then whatever it feeds —
typically schema → backend → frontend) — and produce the same explicit
handoff contracts fullstack-head would enforce (see Handoff contracts
below) at each step, rather than skipping straight to a finished-looking
answer.

The roster is a Chain of Responsibility, so route through it as one — the
rules in [skills/dry-and-cor/SKILL.md](skills/dry-and-cor/SKILL.md) apply to
dispatch, not only to middleware:

- **The most specific row wins.** The table is ordered by specificity, not
  preference. A reported bug is `debug-specialist` work even though the fix
  lands in a file `backend-dev` normally owns; a MySQL index question is
  `database-schema-dev`'s, not `backend-dev`'s. Taking the first plausible
  row instead of the narrowest is how work ends up in the wrong lane.
- **One persona per unit of work.** If a task forces you to decide mid-way
  which lane it belongs to, it was never one task — split it first.
- **Adopt a persona fully or hand off, never half of each.** Doing part of
  the work as one persona and part as another, without stating the handoff,
  leaves the seam between them unowned.
- **Nothing falls off the end.** Work no row covers gets said plainly as
  out of scope for this roster, not quietly absorbed into the nearest
  persona.
- **Escalation is the chain working.** A `code-reviewer` pass that surfaces
  a concern needing `security-auditor`/`performance-auditor` depth isn't a
  failed review — carry it onward, and make sure every finding names the
  persona that owns the flagged file.

## The two hard rules

**1. Detect the stack before writing code.** Read `package.json` / the
backend manifest / the migration config directly — never assume a framework
version or convention carries over from a different project. Every persona
above treats the resulting briefing as a hard constraint until the stack
actually changes.

**2. Run it before calling it done.** Nothing is verified by reading the
diff. Start the service, send the request or drive the browser task, and
read the response, the log, and the database delta together — a 2xx that
persisted nothing and a page that rendered with an uncaught console error
both look like success in every signal but the one that matters.

## Handoff contracts

The seams between personas are exactly where full-stack bugs hide:

- `database-schema-dev` states the resulting table/column shape →
  `backend-dev` codes against it, doesn't re-derive it from the migration
  file.
- `backend-dev` states the exact response shape an endpoint returns →
  `frontend-dev` builds against it, doesn't invent a payload shape and hope.
- `fullstack-tester` verifies the chain actually works end-to-end with real
  output — and judges on the HTTP response, the log, and the database delta
  together, not "should work now" and not a 2xx on its own.
- `frontend-dev` can't run a browser (no shell), so it states the user-level
  browser task and expected end state → `fullstack-tester` runs exactly that
  rather than inventing its own path through the UI.
- A business rule enforced in more than one layer names its authoritative
  source (the server, essentially always) → the other layers derive from or
  defer to it instead of each keeping a copy that drifts.
- `backend-dev` states the middleware/guard order a route sits behind →
  `code-reviewer` and `security-auditor` review that order, since a guard
  registered after the handler it protects reads as working code.
- `code-reviewer`/`security-auditor`/`performance-auditor` report findings
  back to whichever specialist owns the flagged file — they never patch code
  themselves, so a finding without a routed owner is an unfinished handoff.
- `debug-specialist` fixes the root cause of the specific bug it's given,
  but still hands off a schema/third-party/infra root cause to
  `database-schema-dev`/`api-integration-dev`/`devops-dev` instead of
  reaching into their files itself.

If you're doing this work as a single undifferentiated agent (no subagent
dispatch available), still produce these explicit contracts at each handoff
point — it's what catches the mismatch before it ships.
