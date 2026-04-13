# Winning with Claude

A reusable blueprint for structuring Claude Code context in any project — and a skill to set it up interactively.

## What's inside

**`winning-with-claude.md`** — The full guide. Covers the 8-layer context architecture (CLAUDE.md, deep docs, MCP servers, slash commands, skills, memory, settings), when to use each layer, implementation templates, and a real-world case study.

**`SKILL.md`** — An installable Claude Code skill (`claude-project-setup`). Once installed, it guides you through setting up the layered context system for any project — short path (essentials in minutes) or long path (full walkthrough layer by layer).

## Install the skill

Place the files at:

```
.claude/skills/claude-project-setup/
├── SKILL.md
└── references/
    └── winning-with-claude.md
```

Then ask Claude: *"Set up Claude Code for this project"* — the skill triggers automatically.

## What the guide covers

- **Why it matters** — how layered context prevents Claude from starting every session blind
- **8-layer architecture** — what each layer is, when it loads, and what belongs in it:
  - Layer 1: Root `CLAUDE.md` — always loaded, project identity and pointers (~80-100 lines)
  - Layer 2: Package `CLAUDE.md` — auto-loaded per directory, stack and conventions
  - Layer 3: Deep docs — on-demand reference in `.claude/docs/`
  - Layer 4: MCP servers — live data access (APIs, databases, Figma)
  - Layer 5: Slash commands — project-specific workflows invoked with `/command-name`
  - Layer 6: Skills — reusable behavioral patterns that trigger automatically
  - Layer 7: Memory — cross-session persistence (shared team + user-local)
  - Layer 8: Settings — team-wide config, permissions, and hooks
- **How much do you need?** — decision tree for single-package, monorepo, and large multi-team setups
- **How it works in practice** — what Claude sees in common scenarios (working in a package, asking about deployment, querying live data)
- **9 implementation steps** — from gathering source material to verifying the setup works
- **Maintenance guide** — when and who updates each layer, staleness checks
- **7 core principles** — less is more in Layer 1, distill don't duplicate, let it grow organically
- **Case study: EventApp** — full implementation across a Next.js + Strapi + Playwright + OpenTofu monorepo, with every file listed and the source material it was distilled from

## Read the guide

Open `winning-with-claude.md` for the full architecture, decision trees, templates, and case study. It works as standalone reading even without the skill.
