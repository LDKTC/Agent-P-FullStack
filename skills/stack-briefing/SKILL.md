---
name: stack-briefing
description: Detect a project's actual frontend/backend/database/infra stack and versions by reading lockfiles and config directly (not from memory or assumption), and produce a short briefing block other work in this session should treat as a hard constraint. Use before writing any non-trivial full-stack code in an unfamiliar repo, when switching between projects with different stacks, or when asked to confirm/check the stack/version before coding.
---

# Stack briefing

The inline version of `fullstack-senior-dev`'s detection routine — for tools
or contexts that don't have subagent dispatch, run this directly in the main
conversation before writing full-stack code.

## Step 1 — Confirm this is a full-stack web project

Look for a frontend build (`package.json` with React/Vue/Svelte/Next/Nuxt),
a backend service (Node/Express/Fastify/NestJS, Python/Django/FastAPI/Flask,
Ruby/Rails, Go), and a database (Postgres/MySQL/SQLite/MongoDB config, or an
ORM's schema/migration files). If it's not actually this kind of project,
say so and skip the rest.

If the repo is empty or greenfield, there is no stack to detect and the
briefing becomes a recommendation instead: name the scaffold to start from —
an established starter for the intended stack, or the framework's own
`create-*` tool — rather than assembling a skeleton by hand, which loses the
wiring conventions, build config, and dev-server setup a scaffold gets right
for free. State the choice and why, then treat it as the constraint the rest
of the work builds within.

## Step 2 — Pin down real versions

- **Frontend**: `package.json` — framework + version, bundler, UI library,
  state-management library, styling approach.
- **Backend**: the ecosystem manifest (`package.json`,
  `requirements.txt`/`pyproject.toml`, `Gemfile`, `go.mod`) — framework,
  language/runtime version, API style (REST/GraphQL/tRPC).
- **Database**: engine, access layer (raw driver, ORM, query builder),
  migration approach.
- **Infra**: `Dockerfile`/`docker-compose.yml`, CI config, deploy target
  hints (`vercel.json`, `Procfile`, IaC folder).

Never assume a stack matches a project you worked on earlier — re-detect
every time.

## Step 3 — Note existing conventions

Open one representative file per layer (a component, a route/controller, a
schema/model) and note naming, folder layout, and how cross-cutting concerns
(auth, error handling, validation) are already handled.

## Step 4 — Output the briefing

```
STACK CONFIRMED:
  Frontend: <framework> <version> — <state mgmt> — <styling approach>
  Backend: <framework/language> <version> — <API style>
  Database: <engine> — <access layer> — <migration approach>
  Infra: <containerized? CI present? deploy target>

ARCHITECTURE: <folder layout, layering convention, where cross-cutting concerns live>

VERSION GUARDRAILS: <APIs/patterns unavailable or deprecated at these specific versions>

CONVENTIONS TO MATCH: <naming, file placement, error-handling, validation — one real example file per layer>
```

Treat this as a hard constraint for the rest of the session until the
project's stack actually changes.

## What not to do

- Don't skip straight to implementation without this pass on an unfamiliar
  repo — a wrong version assumption compounds into every file touched after.
- Don't invent a convention when the repo already has one, even an
  inconsistent one — match what's there.
