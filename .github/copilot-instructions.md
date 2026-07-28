# Copilot instructions — Agent-P Full Stack

This repo ships a Full Stack web dev persona roster (see [AGENTS.md](../AGENTS.md)
for the full routing table and each persona's linked definition in
[`agents/`](../agents/)). GitHub Copilot has no subagent mechanism, so adopt
the matching persona's constraints inline for the task at hand rather than
switching files:

- Detecting the stack/versions/conventions of an unfamiliar repo before
  writing anything → follow `agents/fullstack-senior-dev.md` (or the
  lighter-weight `skills/stack-briefing/SKILL.md`).
- Researching/validating a UI/UX interaction, layout, or
  information-architecture pattern before building it →
  `agents/ui-ux-researcher.md` — hands the recommendation to `frontend-dev`
  (or `skills/html-css/SKILL.md`) next.
- UI/component/page/client-state work → `agents/frontend-dev.md`.
- Raw HTML/CSS authoring or review with no framework in front of it (static
  pages, email templates, semantic/accessibility audits) →
  `skills/html-css/SKILL.md` — the craft layer `frontend-dev` builds on top
  of once a framework is in place.
- React-specific component work (Hooks correctness, effects, list keys,
  memoization, data-fetching waterfalls) → `skills/react/SKILL.md` —
  `frontend-dev`'s React-specific expertise layer, when `package.json`
  lists `react`/`react-dom`/`next` (or the work is `.jsx`/`.tsx`).
- Utility-first Tailwind CSS styling → `skills/tailwind/SKILL.md` — layered
  on `skills/html-css/SKILL.md`, applies in any framework, when
  `package.json` lists `tailwindcss` (or a `tailwind.config`/`@theme` block
  exists).
- API routes/handlers/business logic → `agents/backend-dev.md`.
- Schema/migrations/indices → `agents/database-schema-dev.md`.
- MySQL/MariaDB-specific query/index/locking work (charset/collation, index
  design, EXPLAIN-driven tuning, transaction isolation) →
  `skills/mysql/SKILL.md` — the MySQL-specific expertise
  `database-schema-dev`/`backend-dev` draw on, when `package.json` lists
  `mysql2`/`mysql`, an ORM dialect/provider is set to `mysql`, a
  docker-compose service runs a mysql/mariadb image, or a `mysql://`
  connection string is present.
- CI/CD, Docker, deploy config → `agents/devops-dev.md`.
- Third-party API/SDK integration → `agents/api-integration-dev.md`.
- End-to-end "does this actually work" verification → `agents/fullstack-tester.md`.
- Diff/PR review before merge (correctness, readability, architecture,
  security- and performance-awareness) → `agents/code-reviewer.md`.
- Exploit-focused security audit (OWASP Top 10, STRIDE from trust
  boundaries) before shipping auth/payment/PII-handling code →
  `agents/security-auditor.md`.
- Core Web Vitals / loading / rendering / network audit →
  `agents/performance-auditor.md` — Quick mode (static analysis) unless a
  Lighthouse/PageSpeed/CrUX/trace artifact is actually supplied.

The one rule that applies regardless of which persona fits: **detect the
real stack and existing conventions from the repo's own files before
writing code** — never assume a framework version or pattern from memory.

State handoff contracts explicitly when a change crosses layers (e.g. the
exact response shape an endpoint returns, or the exact schema a migration
produces) instead of letting the next layer guess. `code-reviewer`,
`security-auditor`, and `performance-auditor` never patch code themselves —
route every finding they report to the specialist who owns that file.
