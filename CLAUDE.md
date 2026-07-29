# CLAUDE.md

This file provides guidance to Claude Code when working with code in this
repository.

## What this repo is

A multi-tool agent+skill **plugin** ("Agent-P"): a Full Stack web dev
department — subagent definitions (`agents/*.md`) in Claude Code's format,
six skills (`skills/stack-briefing/SKILL.md`,
`skills/dev-testing/SKILL.md`, `skills/html-css/SKILL.md`,
`skills/react/SKILL.md`, `skills/tailwind/SKILL.md`,
`skills/mysql/SKILL.md`), and adapter files
(`AGENTS.md`, `.github/copilot-instructions.md`) so the same personas are
usable from tools without a subagent mechanism. No application code, no
build step, no test suite — editing this repo means editing the prose/YAML
that defines agent behavior.

## Commands

There is no build/lint/test tooling — content is markdown + YAML frontmatter,
validated by installing and exercising it in a real session:

```
/plugin marketplace add "d:\AI Project\Agent-P-FullStack"
/plugin install agent-p-fullstack@agent-p-fullstack
```

## Architecture

### Agent frontmatter contract

Every file in `agents/` is:

```
---
name: <kebab-case, must match filename>
description: <capability + explicit "Use PROACTIVELY when ..." trigger, incl. Thai phrasing where relevant>
tools: <comma-separated allowlist, least-privilege for that agent's role>
model: inherit
---
<system prompt body>
```

`description` is load-bearing, not documentation — it's what an invoking
agent/router matches against to decide whether this agent fits a sub-quest,
so it must state the domain, what's distinct from sibling agents, and the
literal trigger phrases (English and Thai). `tools` is a real permission
boundary: `fullstack-head` (orchestrator) gets `Agent, Read, Grep, Glob,
TodoWrite` — no `Edit`/`Write`, it delegates rather than implements;
`fullstack-senior-dev` (read-only briefing) gets `Read, Grep, Glob, Bash` —
no `Edit`/`Write` on purpose; `ui-ux-researcher` (read-only/advisory pattern
research) gets `WebSearch, WebFetch, Read, Grep, Glob` — no
`Edit`/`Write`/`Bash`, it hands a recommendation to `frontend-dev` rather
than implementing; four of the five implementers (`backend-dev`,
`database-schema-dev`, `devops-dev`, `api-integration-dev`) get
`Edit`/`Write`/`Bash` scoped to what their domain actually needs;
`frontend-dev` gets `Edit`/`Write` only — no `Bash`, since UI/component/page
work doesn't need a shell. The three review/audit specialists
(`code-reviewer`, `security-auditor`, `performance-auditor`) get
`Read, Grep, Glob, Bash` — the same read-only shape as `fullstack-senior-dev`
— since they report findings for another specialist to fix, never patch
code themselves. `documentation-architect` gets `Read, Edit, Write, Grep,
Glob, Bash` since it produces persisted doc files. `debug-specialist` gets
the same full set and is the one deliberate exception to strict lane
boundaries — it may `Edit`/`Write` whichever frontend/backend file the root
cause of its assigned bug actually lives in, but still hands a schema/
third-party/infra root cause to the specialist who owns that lane.

### Flat department, one head

Unlike a full org chart, this repo is a single department with no parent
CEO/router above it:

```
fullstack-head ─┬─ fullstack-senior-dev  (briefs every implementer first)
                 ├─ ui-ux-researcher
                 ├─ frontend-dev
                 ├─ backend-dev
                 ├─ database-schema-dev
                 ├─ devops-dev
                 ├─ api-integration-dev
                 ├─ fullstack-tester
                 ├─ code-reviewer          (reviews implementer output; no Edit/Write)
                 ├─ security-auditor       (reviews implementer output; no Edit/Write)
                 ├─ performance-auditor    (reviews implementer output; no Edit/Write)
                 ├─ debug-specialist       (reproduces/fixes a specific bug; crosses lanes for its own fix only)
                 └─ documentation-architect (writes persisted docs from other workers' reports)
```

Every agent's cross-references stay inside this list — none of them assume
a CEO, quest-router, or sibling department (Unity, DraconDex/Electron, etc.)
is present, because this repo is meant to be installed standalone into any
full-stack project. If you add an agent that references something outside
this roster, either add that dependency here too or rewrite the reference to
degrade gracefully when it's absent.

### Keep the three surfaces in sync

There are three descriptions of the same roster, at different granularity:

1. `agents/*.md` — full behavior, Claude Code's native format, the source
   of truth.
2. `AGENTS.md` — a routing table + the one hard rule, for tools that read a
   single root instructions file (Codex, Cursor, etc.) instead of loading
   subagents.
3. `.github/copilot-instructions.md` — the same routing table, Copilot's
   own convention.

Adding, renaming, or re-scoping an agent means updating all three — a stale
`AGENTS.md` entry sends non-Claude tools to a persona that no longer matches
what `agents/*.md` actually does. The same applies to skills: each one is
referenced from `AGENTS.md`, `.github/copilot-instructions.md`, `README.md`,
and the skill count in both `.claude-plugin/*.json` descriptions.

### Verification is a shared contract, not one agent's job

`skills/dev-testing/SKILL.md` defines the probe protocol every runtime check
in this repo refers to: clear stale port listeners, snapshot the database,
wait on the service's own ready line, send **one** request or drive **one**
user-level browser task, then judge on response + log + database delta
together. The rules that follow from reading three signals instead of one —
a 2xx with no row change is a FAIL, an uncaught console error is a FAIL —
are stated in the skill and referenced (not restated in full) by
`fullstack-tester`, `backend-dev`, `debug-specialist`, and `fullstack-head`.
Keep it that way: if a verdict rule changes, it changes in the skill, and
the agents keep pointing at it. `frontend-dev` deliberately has no `Bash` and
therefore never probes — it emits a browser task for a shell-capable agent
to run, which is why its report format carries `BROWSER TASK TO PROBE`.

### Repo/marketplace relationship

`.claude-plugin/plugin.json` is the plugin manifest; `.claude-plugin/
marketplace.json` is a self-referencing marketplace entry (`source: "./"`)
so this repo installs directly with no separate marketplace repo needed.
Keep both files' `name` (`agent-p-fullstack`) in sync — Claude Code resolves
the install as `<plugin-name>@<marketplace-name>`, both of which are this
same name here.
