# Codex Marketplace Plugin Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make the Agent-P FullStack skills installable through a repo-local Codex marketplace.

**Architecture:** A self-contained plugin package under `plugins/agent-p-fullstack` holds a copy of the five existing skills and a Codex manifest. A repository marketplace manifest under `.agents/plugins` discovers it through a local relative source path.

**Tech Stack:** Codex plugin JSON manifests and Markdown skills.

## Global Constraints

- Preserve the existing Claude Code plugin and agent definitions unchanged.
- Use plugin identifier `agent-p-fullstack` consistently for the folder, plugin manifest, and marketplace entry.
- Validate the final plugin with `validate_plugin.py`.

---

### Task 1: Package and validate the Codex plugin

**Files:**
- Create: `plugins/agent-p-fullstack/.codex-plugin/plugin.json`
- Create: `plugins/agent-p-fullstack/skills/stack-briefing/SKILL.md`
- Create: `plugins/agent-p-fullstack/skills/html-css/SKILL.md`
- Create: `plugins/agent-p-fullstack/skills/react/SKILL.md`
- Create: `plugins/agent-p-fullstack/skills/tailwind/SKILL.md`
- Create: `plugins/agent-p-fullstack/skills/mysql/SKILL.md`
- Create: `.agents/plugins/marketplace.json`

**Interfaces:**
- Consumes: The five skill directories already maintained under `skills/`.
- Produces: A `skills: "./skills/"` plugin manifest and a marketplace entry whose source is `./plugins/agent-p-fullstack`.

- [ ] **Step 1: Scaffold the repository-local package and marketplace entry**

Run: `python C:\\Users\\User\\.codex\\skills\\.system\\plugin-creator\\scripts\\create_basic_plugin.py agent-p-fullstack --path .\\plugins --marketplace-path .\\.agents\\plugins\\marketplace.json --marketplace-name agent-p-fullstack --with-skills --with-marketplace`

Expected: A plugin root, default manifest, and marketplace JSON are created.

- [ ] **Step 2: Copy the five source skill directories into the package**

Run: `Copy-Item -Recurse skills\\stack-briefing,skills\\html-css,skills\\react,skills\\tailwind,skills\\mysql plugins\\agent-p-fullstack\\skills`

Expected: The package contains the five original `SKILL.md` files.

- [ ] **Step 3: Replace scaffold metadata with Agent-P metadata**

Set the manifest version to `1.1.0`, include the existing LDKTC author identity, reference `./skills/`, and provide complete non-placeholder interface metadata.

- [ ] **Step 4: Validate the package**

Run: `python C:\\Users\\User\\.codex\\skills\\.system\\plugin-creator\\scripts\\validate_plugin.py plugins\\agent-p-fullstack`

Expected: Exit code 0 and a validation-success message.

- [ ] **Step 5: Inspect the marketplace resolution inputs**

Run: `Get-Content .agents\\plugins\\marketplace.json; Get-ChildItem plugins\\agent-p-fullstack\\skills -Directory`

Expected: The single `agent-p-fullstack` entry points at `./plugins/agent-p-fullstack`, and all five skill directories appear.
