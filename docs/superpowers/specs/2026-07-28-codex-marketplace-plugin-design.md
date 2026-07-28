# Codex Marketplace Plugin Design

## Goal

Package Agent-P FullStack's five existing skills as a repository-local Codex marketplace plugin without changing the Claude Code plugin.

## Design

- Add `plugins/agent-p-fullstack/` as the Codex plugin root.
- Copy the existing `stack-briefing`, `html-css`, `react`, `tailwind`, and `mysql` skills into that package so the plugin is self-contained.
- Add a validated `.codex-plugin/plugin.json` describing the package and its skills directory.
- Add `.agents/plugins/marketplace.json` with an `agent-p-fullstack` marketplace entry pointing to `./plugins/agent-p-fullstack`.

## Boundaries

The existing `.claude-plugin/` manifests and `agents/` definitions remain unchanged. Codex receives the reusable skills; its existing root `AGENTS.md` continues to provide the cross-tool routing guidance.

## Validation

Run the Codex plugin validator against the packaged plugin, then inspect both JSON manifests and confirm the copied skills are present.
