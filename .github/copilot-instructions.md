# Copilot instructions — Agent-P Full Stack

This repo ships a Full Stack web dev persona roster (see [AGENTS.md](../AGENTS.md)
for the full routing table and each persona's linked definition in
[`agents/`](../agents/)). GitHub Copilot has no subagent mechanism, so adopt
the matching persona's constraints inline for the task at hand rather than
switching files:

- Detecting the stack/versions/conventions of an unfamiliar repo before
  writing anything → follow `agents/fullstack-senior-dev.md` (or the
  lighter-weight `skills/stack-briefing/SKILL.md`).
- UI/component/page/client-state work → `agents/frontend-dev.md`.
- API routes/handlers/business logic → `agents/backend-dev.md`.
- Schema/migrations/indices → `agents/database-schema-dev.md`.
- CI/CD, Docker, deploy config → `agents/devops-dev.md`.
- Third-party API/SDK integration → `agents/api-integration-dev.md`.
- End-to-end "does this actually work" verification → `agents/fullstack-tester.md`.

The one rule that applies regardless of which persona fits: **detect the
real stack and existing conventions from the repo's own files before
writing code** — never assume a framework version or pattern from memory.

State handoff contracts explicitly when a change crosses layers (e.g. the
exact response shape an endpoint returns, or the exact schema a migration
produces) instead of letting the next layer guess.
