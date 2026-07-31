---
name: agent-flow
description: Canonical task classification, dispatch, and handoff workflow for Full Stack web agent work. Use first when a request spans more than one lane or when you need a compact coordination protocol in a non-subagent context.
tools: Read, Grep, Glob
model: inherit
---

# Agent Flow

This skill defines the canonical coordination protocol for Full Stack web
work in this plugin. It is the first workflow to run when the request spans
multiple responsibilities or when the environment does not support agent
subagent dispatch.

## Step 1 — Classify the request

Decide whether the work is:

- **Direct**: a single clearly-owned unit of work that one specialist can
  handle on its own.
- **Coordinated**: two or more dependent lanes, such as schema → backend →
  frontend or frontend + third-party integration.
- **Assured**: work involving auth, payments, PII, public APIs, migrations,
  deployment, or an explicit review/audit request.

If the request is not clearly one unit, do not guess. Break it into
separate, small sub-tasks instead.

## Step 2 — Gather the stack briefing once

If the repo is unfamiliar, obtain a compact stack briefing from
`skills/stack-briefing/SKILL.md` before touching code. Reuse that briefing for
all subsequent work in the current task.

Keep the briefing focused:

```
STACK CONFIRMED:
  Frontend: ...
  Backend: ...
  Database: ...
  Infra: ...

ARCHITECTURE: ...
VERSION GUARDRAILS: ...
CONVENTIONS TO MATCH: ...
```

Do not rerun the full detection unless a manifest, lockfile, DB config,
or deployment config in scope changes.

## Step 3 — Build a compact handoff envelope

When multiple lanes are involved, pass only the context the next owner
actually needs. Use this envelope format:

```
OWNER: <specialist>
INPUTS: <relevant files, compact briefing, preceding contract>
DELIVERS: <files or decision output>
CONTRACT: <schema, API, UI, or config shape the next owner needs>
EVIDENCE: <probe/review evidence expected>
NEXT: <owner | none>
```

Only include the fields that apply. A frontend-only task should not carry
unnecessary database or deployment details.

## Step 4 — Dispatch minimally

- Direct work: one specialist only.
- Coordinated work: sequence dependent owners in order, not all at once.
- Assured work: add the corresponding verifier(s) after implementation,
  not before.

Respect the roster's lane ownership rules: most-specific owner wins, one
owner per unit, explicit escalation, and no unroutable task silently
absorbed.

## Step 5 — Keep context small

Do not pass the full conversation transcript as the coordination message.
The coordinator's job is to keep the task bounded and to make the next
handoff explicit.

## Report format

```
OWNER: <specialist>
INPUTS: <relevant file paths, compact stack briefing, preceding contract>
DELIVERS: <concrete files/decision/output>
CONTRACT: <schema/API/UI/config shape the next owner needs>
EVIDENCE: <test/probe/review evidence expected>
NEXT: <owner | none>
```

## What not to do

- Don't treat every request as a multi-agent work item. A simple frontend
  bug is still direct work if only `frontend-dev` owns it.
- Don't duplicate global routing, testing, or technology rules that another
  specialist or skill already governs.
- Don't leave an owner with an implicit contract. If the next step needs a
  schema or response shape, write it down explicitly.
- Don't skip the stack briefing on an unfamiliar repo before changing code.
