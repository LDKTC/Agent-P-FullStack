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
  falling off the end into a silent success.
- **`skills/mysql/`** — MySQL/MariaDB-specific database expertise (data
  types and charset/collation, index design and EXPLAIN-driven query
  tuning, transaction isolation and locking), active when the project's
  database is MySQL or MariaDB — layered under `database-schema-dev` and
  `backend-dev`.
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

This loads all 14 agents and the `stack-briefing`, `dev-testing`,
`dry-and-cor`, `html-css`, `react`, `tailwind`, and `mysql` skills into any
project you work on with Claude Code.

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
