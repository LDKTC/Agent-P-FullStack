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

This loads all 14 agents and the `stack-briefing`, `html-css`, `react`,
`tailwind`, and `mysql` skills into any project you work on with Claude
Code.

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

## Known caveat

If a target project already has same-named agents or skills from another
source, both will exist side by side. Rename or remove the duplicate rather
than assuming one silently wins.
