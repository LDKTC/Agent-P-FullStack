# Efficient Agent Flow Design

## Goal

Make Agent-P FullStack faster and less token-intensive without weakening its
lane ownership, stack detection, explicit handoffs, or runtime verification.
Keep Claude Code, AGENTS.md-aware tools, GitHub Copilot, and Codex package
users on the same workflow.

## Decision

Adopt a canonical, tiered agent-flow skill and make every platform surface a
thin adapter to it. Keep the existing specialist roster and technology skills;
change how work is selected, sequenced, and how much context is passed.

## Architecture

### Canonical agent-flow skill

Add `skills/agent-flow/SKILL.md` as the single authoritative workflow for
task classification, dispatch, context budgets, handoffs, and escalation.
It will define three execution tiers:

1. **Direct**: one clearly-owned, low-risk unit of work. Adopt one specialist
   inline; do not create a coordination pass.
2. **Coordinated**: two or more dependent lanes. Obtain one compact stack
   briefing, set only the contracts needed at each seam, then dispatch in
   dependency order.
3. **Assured**: work involving authentication, payments, PII, public APIs,
   migrations, deployment, or a request for review/audit. Add only the
   corresponding verifier(s) after implementation.

The skill will retain the existing Chain-of-Responsibility ownership rules:
most-specific owner wins, one owner per unit, explicit escalation, and no
unroutable task silently absorbed. It will reference `dry-and-cor` and
`dev-testing` rather than reproduce their detailed rules.

### Briefing and handoff protocol

For unfamiliar repositories, run `stack-briefing` once before code changes.
Produce a maximum 250-word briefing containing only the active stack,
conventions, version guardrails, and relevant paths. Reuse it for the current
task. Re-run it only if a manifest, lockfile, database configuration, or
deployment configuration changes in scope.

Pass a compact handoff envelope between workers instead of a full transcript:

```
OWNER: <specialist>
INPUTS: <relevant file paths, compact stack briefing, preceding contract>
DELIVERS: <concrete files/decision/output>
CONTRACT: <only schema, API, UI, or configuration shape the next owner needs>
EVIDENCE: <test/probe/review evidence expected>
NEXT: <owner | none>
```

The coordinator passes only the fields that apply. A standalone UI change,
for example, receives no database or deployment context.

### Platform adapters

- `agents/fullstack-head.md` becomes a compact Claude Code coordinator that
  selects a tier, maintains a short routing record, and dispatches only
  required specialists.
- `AGENTS.md` and `.github/copilot-instructions.md` become concise routing
  adapters: use `agent-flow` first, then load the selected specialist and
  applicable technology skill.
- `README.md` and `CLAUDE.md` describe the canonical-source policy and the
  supported packaging workflow without restating operational rules.
- Existing specialist prompts retain their lane-specific procedures, but
  replace duplicated global routing, testing, and technology explanations
  with references to their canonical skills where this does not remove a
  necessary permission or role boundary.

### Mobbin-informed UI/UX research

Extend the UI/UX research path with Mobbin as an optional visual-reference
source for real app, web-app, and public-site flows. Mobbin supports searching
screens, UI elements, text, and full flows; its official documentation also
offers a remote MCP integration for coding agents. Use it only when available
and authorized in the host environment. Do not add credentials, require a
paid account, bypass a login/paywall, or install an external integration as
part of this plugin.

The researcher will use the narrowest query that describes the user's
problem, platform, and flow (for example, `web checkout address validation`),
then examine no more than five representative examples. It reports the
underlying interaction pattern in its own words, not screenshots, copied
visual details, or a named product's brand expression. Mobbin evidence informs
the visual/interaction recommendation; it never by itself makes a pattern
"proven".

Keep the existing quality bar:

1. confirm convergence across at least three independent products;
2. corroborate the decision with an appropriate authoritative UX or platform
   guideline;
3. reject inaccessible, deceptive, or brand-specific treatments even when
   they are common; and
4. give `frontend-dev` a compact, generic implementation brief rather than
   raw Mobbin images or a screen-by-screen reproduction request.

The revised UI/UX report adds a `MOBBIN REFERENCE` line containing the access
method (`MCP`, browser, or unavailable), a sanitized query, number of examples
reviewed, named pattern convergence, and source URLs/identifiers where the
user may access them. It explicitly states when Mobbin was unavailable so
the result never implies visual research that did not occur. `fullstack-head`
requests this evidence only for a meaningful interaction, layout, or flow
decision; it is not an automatic step for routine component implementation.

Mobbin sources:

- [Mobbin documentation overview](https://docs.mobbin.com/overview) confirms
  it is a UI/UX reference library and documents the remote MCP/API access
  modes and availability.
- [Mobbin MCP](https://mobbin.com/mcp) demonstrates searches over real
  product screens and flows for coding-agent design research.
- [Mobbin changelog](https://mobbin.com/changelog) documents searchable
  apps, screens, UI elements, flows, interaction views, and site sections.

### Codex package and synchronization

Create the repository-local Codex marketplace package described by the
existing Codex design documents. Its generated skill contents are a complete
copy of the canonical `skills/` directory, including `agent-flow`; the
root `AGENTS.md` remains the non-subagent adapter for projects that use it.

Add a deterministic PowerShell sync script which:

1. validates the expected source skills,
2. replaces only the package `skills/` directory with an exact copy of root
   `skills/`,
3. validates the Codex plugin manifest, and
4. fails if a tracked package skill differs from its source counterpart.

The script is the sole supported way to refresh packaged skills. This keeps
the package portable while avoiding hand-maintained divergence. It must not
modify the Claude plugin manifest or its existing marketplace entry.

## Changes in Scope

- Add the canonical `agent-flow` skill and its Codex-packaged copy.
- Add the optional, evidence-bounded Mobbin research protocol to the UI/UX
  researcher, the department-head dispatch criteria, and the frontend handoff
  contract.
- Tighten the department-head and cross-platform adapters around the tiered
  protocol.
- Remove duplicated global instructions where the canonical skill already
  governs them, retaining platform-specific details and safety boundaries.
- Create and validate the complete Codex marketplace plugin and deterministic
  package sync script.
- Update manifest metadata, installation documentation, counts, and internal
  links to reflect the new skill and package.
- Add lightweight structural checks for source/package parity, required
  references, and valid JSON/YAML frontmatter.

## Non-Goals

- Remove or merge specialist roles.
- Lower security, testing, schema, or authorization requirements.
- Make every task use multiple agents or mandatory research.
- Replace the existing Claude Code plugin or change its installation route.
- Introduce external services, telemetry, model-specific token metering, or
  application runtime code.

## Error Handling and Safety

The flow routes unknown work as out of scope rather than guessing. A missing
or stale stack briefing forces a refreshed briefing before a code-changing
task proceeds. The package sync validates all destinations before replacing
them and fails on missing/extra/different skills rather than silently leaving
a partial package. Verification remains proportional: direct work uses its
owner's evidence, coordinated work verifies its changed vertical slice, and
assured work includes only risk-matched reviewers.

## Validation

- Run the Codex plugin validator after generating the package.
- Run the parity check after synchronization and intentionally corrupt a
  packaged test copy in a temporary worktree to prove drift is detected.
- Parse every JSON manifest and every skill's YAML frontmatter.
- Search adapters and specialist prompts for obsolete duplicated dispatch
  procedures and verify links point to the canonical flow.
- Review a direct task, a schema-to-API-to-UI task, and an auth-related task
  against the tier selection and handoff templates.

## Expected Outcome

The routine context for a direct task is one routing adapter, one specialist,
and only the technology skills its files trigger. Multi-layer work gains a
single compact briefing and bounded handoffs rather than repeated full
instructions. Codex receives the same canonical skills as the source plugin,
with sync and validation making any future drift visible immediately.
