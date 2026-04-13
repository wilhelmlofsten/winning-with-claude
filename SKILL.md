---
name: claude-project-setup
description: Use when setting up Claude Code for a new project, configuring CLAUDE.md files, or asking how to structure AI context for a codebase. Covers the layered context architecture: CLAUDE.md, .claude/docs/, settings, slash commands, skills, and MCP servers.
---

# Claude Project Setup

A complete blueprint for implementing Claude Code's layered context system in any project.

## When to Use

- Starting a new project and want Claude to understand it from day one
- Existing project where Claude keeps asking the same questions or guessing wrong
- Team onboarding Claude and want consistent behaviour across developers
- Wondering what belongs in `CLAUDE.md` vs `.claude/docs/` vs a skill vs a command

## What This Covers

The full layered context architecture has 8 layers:

| Layer | What                                                                           | When loaded                                     |
| ----- | ------------------------------------------------------------------------------ | ----------------------------------------------- |
| 1     | Root `CLAUDE.md`                                                               | Every session                                   |
| 2     | Package `CLAUDE.md` files                                                      | When working in that directory                  |
| 3     | Deep docs (`.claude/docs/`)                                                    | On demand, via pointers                         |
| 4     | MCP servers                                                                    | On startup (live data access)                   |
| 5     | Slash commands (`.claude/commands/`)                                           | When user types `/command-name`                 |
| 6     | Skills (`.claude/skills/`)                                                     | Auto-loaded based on task                       |
| 7     | Memory (`.claude/memory/` in repo + `~/.claude/projects/…/memory/` user-local) | On demand (shared) / every session (user-local) |
| 8     | Settings (`.claude/settings.json`)                                             | Team-wide config and hooks                      |

## Core Principle

**Give Claude the right context at the right time.** Too much context in Layer 1 degrades quality. Too little means Claude guesses wrong. The goal is a tight root `CLAUDE.md` (~80-100 lines) that points to deeper docs Claude can fetch on demand.

## Quick Decision Guide

**Not sure what layer something belongs in?**

- Always needed, every session → root `CLAUDE.md`
- Only needed in one package/directory → package `CLAUDE.md`
- Detailed reference (architecture, runbooks) → `.claude/docs/`
- Repetitive workflow with project-specific steps → `.claude/commands/`
- Reusable judgment across projects → `.claude/skills/`
- Live data from APIs/design tools → MCP server
- Team-wide gotchas / shared knowledge → `.claude/memory/MEMORY.md` (in repo)
- Personal/local gotchas between sessions → `~/.claude/projects/…/memory/`

Full implementation guide, templates, and a real-world case study: see `references/winning-with-claude.md`.

## Guided Setup Flow

When the user wants to set up Claude Code for a project, follow this interactive flow exactly. Do not skip questions or assume answers.

### Step 1 — Choose setup depth

Ask:

> "Do you want a **short setup** or a **long setup**?
>
> - **Short** — essentials only: root `CLAUDE.md` + settings skeleton. Gets you running in minutes.
> - **Long** — guided walkthrough: I'll go through each layer and ask whether you want it for your project."

---

### Short Setup path

Collect only what's needed for the two essentials:

1. Ask: "What is the project name and one-line description?"
2. Ask: "Is this a single-package project or a monorepo? If monorepo, list the packages."
3. Ask: "What are the most-used dev and test commands?"
4. Ask: "Any conventions to lock in now — commit format, branch naming, TypeScript strictness?"

Then generate:

- Root `CLAUDE.md` from the template in `references/winning-with-claude.md` (Step 2 template)
- `.claude/settings.json` as an empty `{}`

Tell the user: "You can add more layers any time — run this skill again and choose Long Setup."

---

### Long Setup path

Read `references/winning-with-claude.md` first, then walk through each layer below. For each one, present a one-sentence summary and ask: **"Do you want to set this up?"** Wait for a yes/no before moving on. Only generate files for layers the user confirms.

**Layer 1 — Root `CLAUDE.md`** _(always recommend yes)_

> "The always-loaded context file. Gives Claude your project identity, commands, conventions, and pointers to deeper docs. Recommended for every project."
> If yes: ask for project name, description, package list, key commands, conventions, tool versions, environments.

**Layer 2 — Package `CLAUDE.md` files**

> "Per-directory context files, auto-loaded when Claude works inside that package. Useful for monorepos or projects with distinct frontend/backend/infra areas."
> If yes: for each package, ask for stack, commands, project structure, key patterns, code style, testing approach.

**Layer 3 — Deep docs (`.claude/docs/`)**

> "On-demand reference docs for architecture, workflows, coding standards. Claude fetches these when it needs detail beyond what's in CLAUDE.md."
> If yes: ask which docs to start with (architecture, workflows, coding-standards) and whether there are domain-specific areas needing guides.

**Layer 4 — MCP servers**

> "Connects Claude to live data — your backend API, Figma designs, database. Claude can query real data instead of guessing."
> If yes: ask what data sources exist (GraphQL/REST API, Figma, database) and whether they want a custom stdio server or hosted http services.

**Layer 5 — Slash commands (`.claude/commands/`)**

> "Project-specific workflow guides invoked with `/command-name`. Encode repetitive tasks like scaffolding a page, running codegen, or reviewing a PR."
> If yes: ask what repetitive workflows the team does (scaffolding, deploy steps, review checklists) and generate command stubs for each.

**Layer 6 — Skills (`.claude/skills/`)**

> "Reusable behavioral patterns that auto-trigger based on task. Unlike commands, skills work across projects — e.g. systematic debugging, planning before coding."
> If yes: ask whether they want shared process skills (from superpowers) and whether there are any project-specific judgment calls worth encoding.

**Layer 7 — Memory**

> "Persists gotchas and hard-won knowledge. Shared memory (in repo) covers team-wide knowledge; user-local memory covers personal/setup-specific notes."
> If yes: ask what the most important gotchas or failure modes are, and generate `.claude/memory/MEMORY.md` stubs for both shared and user-local.

**Layer 8 — Settings (`.claude/settings.json`)**

> "Team-wide Claude Code config: MCP server enablement, PostToolUse hooks, permissions. Start minimal and grow it from real friction."
> If yes: ask if they have MCP servers to enable, commands to pre-approve, commands to block, or workflow reminders to hook.

---

After all layers: summarise what was created, list any layers skipped, and remind the user to run Step 9 (Verify the Setup) from `references/winning-with-claude.md`.
