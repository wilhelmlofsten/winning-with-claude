# Winning with Claude — Layered Claude Code Architecture

A reusable blueprint for implementing Claude Code's layered context system in any project. Includes the architecture pattern, implementation templates, and a real-world case study.

---

## Why This Matters

Claude Code reads `CLAUDE.md` files automatically based on your working directory. Without them, Claude starts every session blind — it doesn't know your stack, conventions, commands, or architecture. With a well-structured setup, every team member gets a Claude that already understands the project.

The key insight: **give Claude the right context at the right time**. Too much context degrades quality. Too little means Claude guesses wrong.

---

## Architecture: Layered Context

```
CLAUDE.md                          ← Always loaded (~80-100 lines)
├── {package}/CLAUDE.md            ← Auto-loaded when working in that directory
├── .mcp.json                      ← MCP server config (gitignored, per-developer)
├── tools/mcp-{project}/           ← Custom MCP servers (e.g., GraphQL API)
└── .claude/
    ├── settings.json              ← Team-wide config (MCP enablement, hooks, permissions)
    ├── commands/                  ← Project-specific slash commands (invoked with /command-name)
    │   ├── create-page.md         ← Example: scaffold a Next.js page
    │   └── review-pr.md           ← Example: review branch against standards
    ├── skills/                    ← Reusable behavioral skills (loaded on demand)
    │   └── {skill-name}/
    │       ├── SKILL.md           ← Skill instructions + YAML frontmatter
    │       └── references/        ← Heavy reference content (loaded on demand)
    ├── memory/                    ← Shared team memory (checked in, loaded on demand)
    │   └── MEMORY.md              ← Gotchas, key paths, hard-won knowledge for the whole team
    └── docs/
        ├── architecture.md        ← Deep reference (read on demand)
        ├── workflows.md           ← Operational procedures
        ├── coding-standards.md    ← Full conventions
        ├── mcp-setup.md           ← MCP server setup guide
        └── {domain}/              ← Domain-specific deep guides
```

### Layer 1: Root `CLAUDE.md` (Always On)

**Loaded:** Every session, regardless of directory.

**Purpose:** Project identity, monorepo map, essential commands, conventions summary, domain dictionary, and pointers to deeper docs.

**Target:** ~80-100 lines. This is the most expensive layer — every token here is consumed on every interaction. Keep it tight.

**What belongs here:**

- One-line project description
- Directory map with one-liner per package
- Most-used commands (dev, test, lint)
- Convention summary (commit format, branch naming)
- Tool versions
- Domain-specific terms the team uses
- Pointers to package CLAUDE.md files and `.claude/docs/`

**What does NOT belong here:**

- Detailed API docs, component guides, or architectural deep-dives
- Long code examples
- Anything that only matters for one package

### Layer 2: Package `CLAUDE.md` Files (Auto-Loaded Per Directory)

**Loaded:** When Claude is working on files inside that directory.

**Purpose:** Stack details, package-specific commands, patterns, and conventions.

**Target:** ~40-80 lines per package.

**What belongs here:**

- Full tech stack with versions
- All package commands
- Project structure overview (`src/` layout)
- Key patterns (styling approach, testing philosophy, data access)
- Code style specifics (Prettier config, ESLint rules)

### Layer 3: Deep Docs (On Demand)

**Loaded:** Only when Claude reads them, typically guided by pointers in root CLAUDE.md.

**Purpose:** Comprehensive reference material — architecture, workflows, runbooks, guides.

**Target:** No line limit. These are full docs.

**Structure:** Organized in `.claude/docs/` with subfolders by domain. An `index.md` file serves as the central table of contents.

**Index file (`index.md`):** Lists every deep doc with its path and topics. The root CLAUDE.md points to this index instead of listing every file individually. This keeps the root file short and makes the doc structure self-describing.

**What belongs here:**

- `index.md` — central table of contents for all deep docs
- System architecture with flow diagrams
- Step-by-step operational workflows
- Detailed coding standards
- Domain-specific deep guides (GraphQL codegen, secrets management, CI/CD pipeline)

### Layer 4: MCP Servers (Live Data)

**Files:** `tools/` directory + `.mcp.json` (gitignored, per-developer)

**Loaded:** On Claude Code startup. Claude gains tool-calling access to external data sources.

**Purpose:** Give Claude access to live project data — APIs, databases, design tools — so it can answer questions grounded in real data instead of guessing.

**Common MCP servers:**

- **GraphQL/REST API** — query your backend directly (events, users, content)
- **Figma** — read designs, generate code from design files, map components
- **Database** — query schemas, inspect data (read-only recommended)

**Setup pattern:**

1. Build or configure the MCP server (stdio for local tools, http for hosted services)
2. Create `.mcp.json` in repo root with server config and credentials (gitignored)
3. Share `.mcp.json` template in docs so teammates can set up their own
4. Enable servers in `.claude/settings.json` via `enabledMcpjsonServers`

**What belongs here:**

- Read-only tools that query project data (events, sessions, designs)
- Tools that help Claude understand the current state of the system
- PostToolUse hooks for workflow reminders (e.g., "run codegen after editing .graphql files")

**What does NOT belong here:**

- Write operations without explicit user confirmation
- Tools with production credentials in shared config
- Anything that could modify data without the developer's knowledge

### Layer 5: Slash Commands (Project Workflows)

**Directory:** `.claude/commands/`

**Loaded:** On demand — user types `/command-name` in Claude Code chat.

**Purpose:** Project-specific workflow guides that tell Claude *how* to perform a common task in this codebase. Each file is a markdown prompt injected at invocation time. Commands are self-contained — they reference exact file paths, project conventions, and CLI commands so Claude can execute the workflow without extra context.

**What belongs here:**

- Repetitive scaffolding tasks (new page, new component, new content type)
- Workflows with project-specific steps (codegen after GraphQL changes, SOPS before infra edits)
- Review workflows grounded in this project's standards

**What does NOT belong here:**

- Generic workflows with no project-specific content (use a skill instead)
- Commands that require Claude to hold state between steps — commands are stateless prompts

**Inline arguments with `$ARGUMENTS`:** Commands can accept inline arguments so users skip the first follow-up question. Add a **Usage** line and reference `$ARGUMENTS` in the relevant step:

```markdown
**Usage:** `/create-component [ComponentName]`

If `$ARGUMENTS` is non-empty, use it as the component name and skip asking for it in Step 1.
```

Then `/create-component MyButton` passes `MyButton` directly — saving one round-trip per invocation. Use this for the primary name/path argument (the first thing you'd ask for).

**Command file structure:** Pure markdown. No frontmatter required for basic commands. The content is the prompt — write it as instructions Claude should follow, with numbered steps, code examples, exact paths, and verification commands.

```markdown
# Command Name

One sentence describing what this does.

## Step 1 — Gather requirements

Ask the user for X, Y, Z.

## Step 2 — Find reference

Check `path/to/existing/` for similar examples.

## Step 3 — Create the files

Create `path/to/new/file.ts` with this content:
\`\`\`typescript
// ...
\`\`\`

## Step N — Verify

Run: `npm run typecheck`
```

### Layer 6: Skills (Reusable Behavioral Patterns)

**Directory:** `.claude/skills/{skill-name}/` or shared via `~/.claude/skills/`

**Loaded:** On demand — Claude loads a skill when the task matches the skill's description. Unlike commands (which users invoke explicitly), skills trigger automatically based on what Claude is doing.

**Purpose:** Encode reusable judgment and workflows that apply across projects. Commands are project-specific ("scaffold a page in *this* codebase"). Skills are general ("how to write a plan before coding" or "how to debug systematically"). Skills work across Claude.ai, Claude Code, and the API.

**Structure:**

```
.claude/skills/my-skill/
├── SKILL.md          ← Required. YAML frontmatter + instructions.
└── references/       ← Optional. Heavy content loaded on demand (keeps SKILL.md lean).
```

**SKILL.md frontmatter must include:**
- `name` — kebab-case folder name
- `description` — Under 1024 chars, no XML. Must state WHAT it does AND WHEN to use it (trigger phrases). Include negative triggers ("Do NOT use for...") to prevent unwanted activation.

**Three-level progressive disclosure:**
1. YAML frontmatter → always in system prompt (Claude decides whether to load the skill)
2. SKILL.md body → loaded when relevant (keep under 5,000 words)
3. `references/` files → loaded on demand for heavy content

**Skill vs command — when to use which:**

| | Commands | Skills |
|--|----------|--------|
| Trigger | User types `/command-name` | Claude auto-loads based on task |
| Scope | Project-specific (exact paths, conventions) | Reusable across projects |
| Content | Step-by-step workflow with project details | Behavioral pattern or judgment |
| Example | `/create-component` (knows your CVA setup) | `systematic-debugging` (works anywhere) |

**Key gotchas:**
- Always use `superpowers:writing-skills` skill when authoring a new skill — it validates structure before deploy
- Use `allowed-tools` frontmatter to restrict which tools the skill can invoke (important for safety-sensitive skills)
- Skills are rigid (follow exactly) or flexible (adapt principles) — the skill itself should state which

**What belongs here:**
- Process skills: how to approach a task (brainstorming, debugging, planning, code review)
- Expertise skills: domain knowledge encoded as behavior (accessibility rules, security checklist)
- Multi-MCP coordination: sequences that span multiple tools (e.g., Figma → data → component)

**What does NOT belong here:**
- Project-specific steps that only make sense in one codebase (use commands instead)
- Content that should live in CLAUDE.md (architecture, conventions, commands reference)

### Layer 7: Memory (Cross-Session Persistence)

Two memory locations serve different purposes — use both:

**Shared memory (in repo):** `.claude/memory/MEMORY.md`

Checked into the repo. Contains team-wide gotchas, key file paths, and hard-won knowledge every developer needs. Referenced from root `CLAUDE.md`'s session-start instruction so Claude loads it on demand. This is the right place for project knowledge that should be consistent across the whole team.

**User-local memory:** `~/.claude/projects/{project-slug}/memory/` (outside the repo)

**Loaded:** `MEMORY.md` is automatically injected into every session for that project. Lines beyond 200 are truncated, so keep it tight.

**Purpose:** Persist personal preferences, in-progress context, or developer-specific notes that don't belong in the shared repo — e.g., local setup quirks, personal workflow preferences.

**What belongs here:**

- Gotchas that aren't in any doc (e.g., "restart Strapi after adding a content type or migrations won't run")
- Known failure modes with their fixes (e.g., "stale NextAuth session after Strapi restart — sign out and back in")
- Frequently-needed file paths that don't belong in `CLAUDE.md`
- A mini docs index so Claude knows where to look for deep context

**What does NOT belong here:**

- Session-specific context (current task, in-progress state)
- Information already in `CLAUDE.md` or package docs — don't duplicate
- Speculative or unverified conclusions

**File structure:**

```
~/.claude/projects/{project-slug}/memory/
├── MEMORY.md        ← Always loaded. Keep under 200 lines.
├── patterns.md      ← Optional: coding patterns discovered during work
└── debugging.md     ← Optional: debugging insights and known failure modes
```

Create additional topic files only when a section in `MEMORY.md` outgrows it — don't pre-create empty stubs.

**To find your project slug:** it's the repo path with `/` replaced by `-` (e.g., `/Users/you/myproject` → `-Users-you-myproject`).

### Layer 8: Settings (Team Configuration)

**File:** `.claude/settings.json`

**Purpose:** Shared Claude Code behavior for the whole team. Enables MCP servers, configures hooks, and manages permissions. Start minimal and add rules as friction points emerge.

**Common settings:**

- `enabledMcpjsonServers` — Which MCP servers to connect on startup
- `hooks.PostToolUse` — Workflow reminders triggered by tool actions
- `permissions.allow` — Commands everyone approves of (e.g., `Bash(npm run test)`)
- `permissions.deny` — Commands that should never run (e.g., reading secrets files)

---

## How Much Do You Need?

Not every project needs the full setup. Use this decision tree:

**Single-package project (e.g., one Next.js app):**

- Root `CLAUDE.md` — yes, always
- Package CLAUDE.md files — no (only one package)
- `.claude/docs/` — 1-2 files if complex, skip if simple
- MCP servers — only if Claude needs live API/design data
- Skills — only if the team has recurring workflows worth encoding (start with shared `superpowers` skills)
- `.claude/settings.json` — only if team uses Claude Code

**Monorepo with 2-3 packages:**

- Root `CLAUDE.md` — yes
- Package CLAUDE.md files — yes, one per package
- `.claude/docs/` — start with 3 core docs (architecture, workflows, coding-standards)
- MCP servers — recommended if you have a backend API or use Figma
- Skills — recommended: adopt shared process skills (brainstorming, debugging, planning) + add project-specific ones as judgment calls emerge
- `.claude/settings.json` — empty skeleton

**Large monorepo (4+ packages, multiple teams):**

- Root `CLAUDE.md` — yes
- Package CLAUDE.md files — yes, one per package
- `.claude/docs/` — 3 core docs + domain subfolders with deep guides
- MCP servers — yes, for API, design tools, and any live data sources
- Skills — yes: process skills team-wide, domain skills per package, multi-MCP coordination skills
- `.claude/settings.json` — configure team-wide permissions

---

## How It Works in Practice

| Scenario                       | What Claude sees                                                                 |
| ------------------------------ | -------------------------------------------------------------------------------- |
| New session in project root    | Root `CLAUDE.md` only                                                            |
| Working in `frontend/src/`     | Root `CLAUDE.md` + `frontend/CLAUDE.md`                                          |
| "How does deployment work?"    | Claude reads root pointers, then fetches `.claude/docs/architecture.md`          |
| "How do I add a content type?" | Claude reads root pointers, then fetches `.claude/docs/backend/content-types.md` |
| "What events exist?"           | Claude calls `query_events` MCP tool, returns live data from Strapi              |
| "Implement this Figma frame"   | Claude calls `get_design_context` MCP tool, reads design, generates code         |

### Example: The RPI Pipeline Skill

**Scenario:** User says "Let's add speaker filtering to the sessions page."

**What happens:**
1. The `research-plan-implement` skill triggers automatically (skill description matches: "complex multi-file feature")
2. Claude presents the RPI routing table: "This looks complex. Follow this path:"

```
Stage        Command          Output                    Next step
─────────────────────────────────────────────────────────────────
Research     /rpi-research → .thoughts/research/       ← Review findings
Design       /rpi-design   → .thoughts/designs/        ← Approve approach
Plan         /rpi-plan     → .thoughts/plans/          ← Review phases
Implement    /rpi-implement → code + tests + commits   ← Done
Verify       /rpi-verify   → .thoughts/reviews/        ← Optional quality check
Archive      /rpi-archive  → .thoughts/archive/        ← Clean up after shipping
```

Each stage produces a reviewable artifact in `.thoughts/` before proceeding to the next. You can inspect, redirect, or skip stages as needed. Only `/rpi-implement` writes code.

---

## Implementation Steps

### Step 1: Gather Source Material

Collect these artifacts from the project:

- All `package.json` files (scripts, dependencies)
- Tool version configs (`mise.toml`, `.nvmrc`, `.tool-versions`)
- Existing documentation
- CI/CD configuration
- Linter/formatter configs
- Directory structure

### Step 2: Write Root CLAUDE.md

Start with this template, keep under 100 lines:

```markdown
# {{PROJECT_NAME}} — {{ONE_LINE_DESCRIPTION}}

{{2-3 sentence project description.}}

## Monorepo Structure

- `{{package1}}/` — {{framework}} ({{one-liner}})
- `{{package2}}/` — {{framework}} ({{one-liner}})

## Key Commands

\`\`\`sh
{{most_used_dev_command}} # Start development
{{most_used_test_command}} # Run tests
\`\`\`

### {{Package 1}} (`cd {{package1}}/`)

\`\`\`sh
{{commands}}
\`\`\`

### {{Package 2}} (`cd {{package2}}/`)

\`\`\`sh
{{commands}}
\`\`\`

## Tool Versions

- {{tool_list_with_versions}}

## Conventions

- **Commits:** {{commit_format}}
- **Branches:** {{branch_format}}
- **PRs:** {{pr_rules}}
- **TypeScript:** {{ts_approach}}
- **Testing:** {{testing_approach}}

## Domain Dictionary

| Term     | Meaning        |
| -------- | -------------- |
| {{term}} | {{definition}} |

## Environments

- **{{env1}}** — deployed from `{{branch1}}`
- **{{env2}}** — deployed from `{{branch2}}`

## Documentation

All deep documentation is indexed in `.claude/docs/index.md`. Consult the index to find the right doc for any topic.

**At session start:** Check `.claude/docs/index.md` when you need deeper context beyond what's in this file and the package CLAUDE.md files.

**After significant changes:** If your work affects architecture, workflows, conventions, or any topic covered in `.claude/docs/`, update the relevant doc and the index if a new doc was added.

## Package-Level Context

See `{{package1}}/CLAUDE.md`, `{{package2}}/CLAUDE.md` for package-specific guidance.
```

**For single-package projects**, drop the "Monorepo Structure" and "Package-Level Context" sections. Put everything in one `CLAUDE.md`.

### Step 3: Write Package CLAUDE.md Files

One per package/workspace. Template:

```markdown
# {{Package Name}} — {{Framework}}

## Stack

- **Framework:** {{name}} {{version}}
- **Language:** {{language}}, {{mode}} mode
- **{{category}}:** {{library/tool}}

## Commands

\`\`\`sh
{{command}} # {{description}}
\`\`\`

## Project Structure (`src/`)

\`\`\`
{{directory_tree}}
\`\`\`

## Key Patterns

- **{{Pattern name}}:** {{Brief explanation}}

## Code Style

- **Formatter:** {{tool}} ({{key settings}})
- **Linter:** {{tool}} ({{key rules}})

## Testing

- **Framework:** {{tool}}
- **Philosophy:** {{approach}}
```

### Step 4: Create Deep Docs

Create `.claude/docs/` with subdirectories matching your packages. Prioritize docs that answer recurring questions.

**Start with these three:**

1. `architecture.md` — System overview, data flows, deployment
2. `workflows.md` — How to do common tasks (setup, deploy, add features)
3. `coding-standards.md` — Full conventions reference

**Then add domain-specific guides** based on what the team asks about most:

```markdown
# {{Topic}} Guide

## Overview

{{What this covers and why.}}

## {{Section}}

{{Content — be specific, include commands and code examples.}}

## References

- `{{path}}` — {{what it contains}}
```

### Step 5: Create Settings

Create `.claude/settings.json` — start empty and add rules incrementally as friction points emerge:

```json
{}
```

Common settings to add over time:

- Tired of approving `npm run test`? Add to `permissions.allow`
- Claude keeps reading `.env` files? Add to `permissions.deny`
- Have MCP servers? Add to `enabledMcpjsonServers`
- Want workflow reminders? Add `PostToolUse` hooks

### Step 6: Create Slash Commands (Optional)

If your project has repetitive development workflows with project-specific steps, encode them as slash commands in `.claude/commands/`.

**When to create a command:**
- You find yourself explaining the same workflow to Claude repeatedly
- A workflow has project-specific paths, tools, or sequences Claude needs to know
- Scaffolding tasks where "always check for an existing example first" matters

**How to create a command:**

1. Create `.claude/commands/<command-name>.md`
2. Write the file as a step-by-step prompt Claude will follow
3. Include: exact file paths, the right CLI commands, conventions reminders, and a verification step
4. Test by typing `/command-name` in a Claude Code session

**Good candidates for commands:**

- `/create-page`, `/create-component` — scaffolding with project conventions baked in
- `/add-graphql-query` — workflows that involve codegen steps after file creation
- `/new-strapi-content-type`, `/new-strapi-plugin` — backend scaffolding with safety warnings for PII types
- `/add-e2e-test` — test scaffolding that references your real test patterns
- `/review-pr` — review checklist specific to your stack and standards
- `/commit` — story-driven commits that group changes logically and extract issue numbers from branch names
- `/publish-pr` — push branch and open an MR with the project template pre-filled
- `/infra-change` — workflows with safety steps (secrets handling, dry runs before apply)

### Step 7: Set Up MCP Servers (Optional)

If your project has a backend API, design tool, or other live data source, connect it via MCP so Claude can query real data. See **Layer 4** above for architecture guidelines.

**GraphQL/REST API server (stdio):**

1. Create a TypeScript MCP server in `tools/mcp-{{project}}/` using `@modelcontextprotocol/sdk`
2. Define tools that wrap your API queries (keep them read-only)
3. Build with `tsc` and reference the compiled JS in `.mcp.json`

**Hosted services (http):**

Add directly to `.mcp.json` — no build step needed:

```json
{
  "mcpServers": {
    "figma": {
      "type": "http",
      "url": "https://mcp.figma.com/mcp"
    }
  }
}
```

**Configuration:**

- `.mcp.json` goes in repo root — gitignore it (contains credentials)
- Document the template and setup steps in `.claude/docs/mcp-setup.md`
- Enable servers in `.claude/settings.json`:

```json
{
  "enabledMcpjsonServers": ["{{server-name}}"]
}
```

**PostToolUse hooks** can remind developers of workflow steps:

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [
          {
            "type": "command",
            "command": "bash -c 'if echo \"$CLAUDE_TOOL_INPUT\" | grep -qE \"\\.graphql\"; then echo \"Remember to run codegen\"; fi'"
          }
        ]
      }
    ]
  }
}
```

### Step 8: Create the Documentation Index

Create `.claude/docs/index.md` — a central table of contents listing every deep doc with its path and topics covered. Template:

```markdown
# Documentation Index

Central index for all Claude deep documentation. Consult this file to find the right doc for any topic.

## {{Category}}

| Doc      | Path          | Topics                                   |
| -------- | ------------- | ---------------------------------------- |
| {{Name}} | `{{path}}.md` | {{keyword1}}, {{keyword2}}, {{keyword3}} |
```

Then update root CLAUDE.md to point to the index and add behavioral instructions:

```markdown
## Documentation

All deep documentation is indexed in `.claude/docs/index.md`. Consult the index to find the right doc for any topic.

**At session start:** Check `.claude/docs/index.md` when you need deeper context beyond what's in this file and the package CLAUDE.md files.

**After significant changes:** If your work affects architecture, workflows, conventions, or any topic covered in `.claude/docs/`, update the relevant doc and the index if a new doc was added.
```

This makes Claude self-maintaining — it knows where to find docs and when to update them.

### Step 9: Verify the Setup

Start a new Claude Code session and test these:

1. **Root context:** Ask "What is this project?" — Claude should know the project name, stack, and structure
2. **Package context:** Work on a file inside a package, ask about its conventions — Claude should know package-specific patterns
3. **Deep doc discovery:** Ask "How do I deploy?" — Claude should find and read the relevant `.claude/docs/` file
4. **Conventions:** Ask about commit format or branch naming — Claude should give the correct format
5. **Commands:** Ask "How do I run tests?" — Claude should give the right command for the current package
6. **MCP tools (if configured):** Ask about live data (e.g., "What events exist?") — Claude should call the MCP tool and return real data

If any of these fail, the CLAUDE.md content is incomplete, the index pointers are missing, or the MCP server isn't connected.

---

## Maintenance

### When to Update

- **Dependency upgrade** — update versions in package CLAUDE.md
- **New package added** — create a new package CLAUDE.md, add to root index
- **Convention change** — update root CLAUDE.md and coding-standards.md
- **New CI/CD stage** — update architecture.md and pipeline guide
- **New deep doc added** — add to `.claude/docs/index.md` (or Claude won't find it)

### Who Updates

The person who makes the change updates the CLAUDE.md. Same principle as updating tests — if you change the behavior, update the docs.

### Staleness Check

Periodically verify (e.g., per sprint):

- Do the commands in root CLAUDE.md still work?
- Do the tool versions match `package.json` / `mise.toml`?
- Are all deep docs listed in `.claude/docs/index.md`?
- Has any package been added or removed without a CLAUDE.md?

---

## Principles

1. **Less is more in Layer 1.** Every line in root CLAUDE.md costs tokens on every interaction. Be ruthless about what earns its place there.

2. **Distill, don't duplicate.** CLAUDE.md files should be Claude-optimized summaries, not copies of existing docs. Point to the full docs for details.

3. **Keep it current.** Stale CLAUDE.md files are worse than no CLAUDE.md — Claude will confidently give wrong answers. Update when the project changes.

4. **Let it grow organically.** Start with the three core deep docs. Add domain-specific guides only when the team repeatedly needs them.

5. **Source from artifacts.** The best CLAUDE.md content comes from `package.json`, configs, and CI files — not from memory. Read the actual files.

6. **Index everything.** If a deep doc isn't listed in root CLAUDE.md, Claude won't find it. The root file is the table of contents.

7. **Settings evolve.** Don't pre-configure permissions. Start empty, add rules from real friction.

---

## Case Study: EventApp

EventApp is a white-label event platform (Next.js 16 + Strapi 5 + Playwright + OpenTofu on AWS). Here's what the full implementation looks like.

### Files Created

| Layer    | File                               | Purpose                                                               |
| -------- | ---------------------------------- | --------------------------------------------------------------------- |
| Root     | `CLAUDE.md`                        | Project identity, commands, conventions, domain dictionary, doc index |
| Package  | `frontend/CLAUDE.md`               | Next.js + React 19 stack, GraphQL codegen, Radix UI, Vitest           |
| Package  | `backend/CLAUDE.md`                | Strapi 5, content types, DB operations, email system                  |
| Package  | `e2e/CLAUDE.md`                    | Playwright config, browser matrix, dev server setup                   |
| Package  | `infra/CLAUDE.md`                  | OpenTofu, AWS modules, SOPS secrets, mise commands                    |
| Deep     | `.claude/docs/architecture.md`     | System overview, auth/media/email flows, CI/CD pipeline               |
| Deep     | `.claude/docs/workflows.md`        | Local setup, event creation, DB ops, deployment flow                  |
| Deep     | `.claude/docs/coding-standards.md` | TypeScript, commits, PRs, testing, formatting, linting                |
| Deep     | `.claude/docs/frontend/`           | Component patterns, GraphQL guide, auth patterns, testing guide       |
| Deep     | `.claude/docs/backend/`            | Content types, plugin dev, email system, data seeding                 |
| Deep     | `.claude/docs/infra/`              | AWS modules, secrets runbook, disaster recovery, pipeline guide       |
| MCP      | `tools/mcp-eventapp/`              | GraphQL MCP server — 5 tools for querying live Strapi data            |
| MCP      | `.claude/docs/mcp-setup.md`        | MCP setup guide (GraphQL + Figma), `.mcp.json` template               |
| Commands | `.claude/commands/create-page.md` | Scaffold Next.js App Router page with auth, data, loading/error states |
| Commands | `.claude/commands/create-component.md` | Scaffold React component (CVA + Radix UI + Vitest test)           |
| Commands | `.claude/commands/add-graphql-query.md` | Add GraphQL query end-to-end with codegen                        |
| Commands | `.claude/commands/new-strapi-content-type.md` | Scaffold full Strapi 5 content type (4 files)              |
| Commands | `.claude/commands/new-strapi-plugin.md` | Scaffold Strapi 5 plugin with optional admin UI                  |
| Commands | `.claude/commands/review-pr.md`    | Review branch changes against project standards                       |
| Commands | `.claude/commands/fix-review.md`   | Read MR comments, categorise, plan and implement fixes                |
| Commands | `.claude/commands/commit.md`       | Story-driven commits grouped by logical increments                    |
| Commands | `.claude/commands/publish-pr.md`   | Push branch and open GitLab MR with project template                  |
| Commands | `.claude/commands/add-e2e-test.md` | Scaffold Playwright E2E test for a user flow                          |
| Commands | `.claude/commands/infra-change.md` | Guide OpenTofu + SOPS change safely                                   |
| Commands | `.claude/commands/add-db-migration.md` | Schema change end-to-end: content type → migration → seed → codegen |
| Commands | `.claude/commands/rpi-*.md`        | RPI pipeline: research, design, plan, implement, verify, archive      |
| Skill    | `.claude/skills/research-plan-implement/` | Trigger skill — routes complex work into the right RPI path    |
| Skill    | `.claude/skills/eventapp-reviewer/` | Domain-parallel PR reviewer — dispatches up to 6 specialist sub-agents (GraphQL, Strapi, auth, frontend, E2E, DX) in parallel; each returns structured findings with confidence signal |
| Skill    | `.claude/skills/eventapp-parallel-research/` | Parallel RPI researcher — dispatches 3 domain expert agents (frontend, backend, infra) simultaneously for multi-domain features; synthesises output into a `.thoughts/research/` artifact |
| Skill    | `.claude/skills/eventapp-consistency-auditor/` | Full-stack consistency auditor — checks schema↔codegen, frontend queries↔Strapi fields, auth flow integrity, and env var propagation |
| Commands | `.claude/commands/audit-consistency.md` | New `/audit-consistency` command — identify what changed, dispatch consistency auditor, present findings with remediation steps |
| Memory   | `.claude/memory/MEMORY.md` | Shared team memory — Strapi/GraphQL gotchas, auth quirks, ECS drain delay, SOPS workflow, key file paths |
| Commands | `.claude/commands/review-pr.md` | **Updated** — Steps 3–7 replaced with a single `eventapp-reviewer` skill dispatch (parallel sub-agents replace the manual checklist) |
| Commands | `.claude/commands/rpi-research.md` | **Updated** — Added Step 0 multi-domain gate: if feature spans 2+ domains, dispatch `eventapp-parallel-research` instead of single-threaded investigation |
| Docs     | `docs/ai-conventions.md`          | Task-oriented onboarding guide for using Claude Code on the project   |
| Settings | `.claude/settings.json`            | MCP server enablement + PostToolUse hooks (codegen, Strapi restart, SOPS re-encrypt, Tofu plan reminder) |
| Memory   | `~/.claude/projects/{slug}/memory/MEMORY.md` | Cross-session project gotchas: auth expiry, ECS drain delay, codegen failures, SOPS workflow |

### Source Material Used

| Source Type      | Examples                                  | Distilled Into                             |
| ---------------- | ----------------------------------------- | ------------------------------------------ |
| Package configs  | `package.json`, `mise.toml`               | Commands, dependencies, tool versions      |
| Existing docs    | `docs/index.md`, `docs/devops.md`         | Architecture, workflows, domain dictionary |
| Convention docs  | `docs/getting-started/ways-of-working.md` | Coding standards, conventions              |
| CI/CD config     | `.gitlab-ci.yml`                          | Pipeline guide, deployment flow            |
| Linter/formatter | ESLint, Prettier configs                  | Code style sections                        |
| Source code      | `src/api/`, `src/components/`             | Content type and component inventories     |
