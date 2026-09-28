# Greenfield/Brownfield Onboarding Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Fix the Plane MCP config and turn the template's init into a single guided orchestrator that onboards greenfield and brownfield projects through the same 8-phase flow.

**Architecture:** The deliverables are Claude Code command/agent markdown files plus JSON/env config — there is no code test runner in this repo. Following the repo's established pattern (see `docs/superpowers/plans/2026-07-21-implement-template.md`), each task produces one focused file (or a tight cluster), and its "test" is a set of shell **acceptance checks** (`jq`, `grep -q`, `test -f`) that fail before the file is written/rewritten and pass after. `/setup-project` becomes an orchestrator that owns Phases 0,1,2,4,5 inline and delegates Phase 3→`/setup-workspace`, Phase 6→`/propagate-rules`, Phase 7→`/check-setup`.

**Tech Stack:** Markdown (command/agent/process docs), strict JSON (`.mcp.json`, `.claude/tracker.json`), dotenv (`.env.example`), git. MCP servers: Plane (`uvx plane-mcp-server`), GitHub/GitLab (`npx @modelcontextprotocol/server-*`).

**Source spec:** `docs/superpowers/specs/2026-07-22-greenfield-brownfield-onboarding-design.md` (authoritative). Base template design: `docs/superpowers/specs/2026-07-15-spec-driven-docs-template-design.md`.

## Global Constraints

Every task's requirements implicitly include these:

- **Writing principle:** concise and to the point for a technical audience — but complete. Omit ceremony, not substance. Markdown only; no marketing copy.
- **The process docs are the source of truth.** `.claude/` commands are the execution adapter; if they disagree with `docs/process/`, the process doc wins.
- **Placeholders, not fabrications.** Values filled at init use obvious placeholders (`<project>`, `<org>`, `…`), never invented specifics.
- **Env vars stay `${...}`** in `.mcp.json` — never hardcode secrets or the DAC slug/URL; DAC defaults are documented in comments only.
- **`.mcp.json` is strict JSON** — no comments; the shipped file keeps all three platform blocks, and `/setup-project` Phase 0 prunes to the chosen ones at init.
- **Commit style (user global rule):** one-liner commit messages, no body, and **never** a `Co-Authored-By` / attribution trailer.
- **Company facts** (only where natural): DAC.digital (DAC.Infomotion Sp. z o.o.), www.dac.digital.
- Work on branch `feat/implement-template` (current). One commit per task.

## File map

| File | Task | Action |
|------|------|--------|
| `.mcp.json` | 1 | Modify — fix Plane block |
| `.env.example` | 1 | Modify — Plane keys |
| `.gitignore` | 2 | Modify — anchor `repos/` → `/repos/` |
| `docs/repos/.gitkeep` | 2 | Create |
| `.claude/commands/setup-workspace.md` | 2 | Rewrite — Phase 3 |
| `.claude/commands/propagate-rules.md` | 3 | Create — Phase 6 |
| `.claude/commands/check-setup.md` | 4 | Create — Phase 7 |
| `.claude/commands/setup-project.md` | 5 | Rewrite — orchestrator (Phases 0–7) |
| `docs/process/README.md` | 6 | Modify — add Onboarding section |
| `README.md` | 7 | Modify — quick start + `uv` prereq |

Tasks 1–4, 6, 7 touch disjoint files and are order-independent. Task 5 (setup-project) references the command names/behaviors fixed by Tasks 2–4; its full content is given here, so it stays consistent regardless of order, but review it last.

---

### Task 1: Fix Plane MCP config + align env

**Files:**
- Modify: `.mcp.json`
- Modify: `.env.example`

**Interfaces:**
- Produces: env var names `PLANE_API_KEY`, `PLANE_WORKSPACE_SLUG`, `PLANE_BASE_URL` (consumed by `/setup-project` Phase 0 and `/check-setup`).

- [ ] **Step 1: Acceptance checks (run first — expect FAIL)**

Run:
```bash
jq -e '.mcpServers.plane.command == "uvx"' .mcp.json \
  && jq -e '.mcpServers.plane.args == ["plane-mcp-server","stdio"]' .mcp.json \
  && jq -e '.mcpServers.plane.env | has("PLANE_API_KEY") and has("PLANE_BASE_URL")' .mcp.json \
  && grep -q '^PLANE_API_KEY=' .env.example \
  && grep -q '^PLANE_BASE_URL=' .env.example \
  && ! grep -q 'PLANE_API_TOKEN' .env.example \
  && echo CHECKS_PASS
```
Expected now: no `CHECKS_PASS` (current file uses `npx`/`PLANE_API_TOKEN`).

- [ ] **Step 2: Write `.mcp.json`** (full content — keeps all three blocks; only Plane changes)

```json
{
  "mcpServers": {
    "plane": {
      "command": "uvx",
      "args": ["plane-mcp-server", "stdio"],
      "env": {
        "PLANE_API_KEY": "${PLANE_API_KEY}",
        "PLANE_WORKSPACE_SLUG": "${PLANE_WORKSPACE_SLUG}",
        "PLANE_BASE_URL": "${PLANE_BASE_URL}"
      }
    },
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_PERSONAL_ACCESS_TOKEN": "${GITHUB_TOKEN}"
      }
    },
    "gitlab": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-gitlab"],
      "env": {
        "GITLAB_PERSONAL_ACCESS_TOKEN": "${GITLAB_TOKEN}",
        "GITLAB_API_URL": "https://gitlab.com/api/v4"
      }
    }
  }
}
```

- [ ] **Step 3: Write `.env.example`** (full content)

```
# Copy to .env and fill in the values for the platforms chosen in .claude/tracker.json
# during /setup-project. Never commit the real .env file.

# --- Plane (tracker) ---------------------------------------------------
# API key from Plane workspace settings > API tokens.
PLANE_API_KEY=
# Workspace slug. DAC default: dac-digital
PLANE_WORKSPACE_SLUG=
# Plane base URL. DAC default: https://plane.dac.digital
PLANE_BASE_URL=

# --- GitHub (tracker and/or git host) -----------------------------------
# Personal access token (repo + issues scopes) or fine-grained token.
GITHUB_TOKEN=

# --- GitLab (tracker and/or git host) -----------------------------------
# Personal/project access token (api scope).
GITLAB_TOKEN=

# --- Comms (optional notifications) -------------------------------------
# Incoming webhook URL for a Slack channel.
SLACK_WEBHOOK_URL=
# Incoming webhook URL for a Discord channel.
DISCORD_WEBHOOK_URL=
```

- [ ] **Step 4: Run acceptance checks again (expect PASS)**

Run the Step 1 command. Expected: prints `CHECKS_PASS`.

- [ ] **Step 5: Commit**

```bash
git add .mcp.json .env.example
git commit -m "Fix Plane MCP config to uvx plane-mcp-server; align .env.example keys"
```

---

### Task 2: Rewrite /setup-workspace (Phase 3) + fix docs/repos ignore

**Files:**
- Modify: `.gitignore` (anchor the reference-checkout ignore so `docs/repos/` is committable)
- Create: `docs/repos/.gitkeep`
- Rewrite: `.claude/commands/setup-workspace.md`

**Interfaces:**
- Consumes: `.claude/tracker.json` → `gitHost` (from `/setup-project` Phase 0).
- Produces: per-repo index files `docs/repos/<repo>.md` and the `workspace/INDEX.md` overview table with columns `Repo | Type | Language | Branch | Last sync | Index` (consumed by `/check-setup` Task 4 and referenced by `/setup-project` Task 5).

- [ ] **Step 1: Acceptance checks (run first — expect FAIL)**

Run:
```bash
git check-ignore -q docs/repos/x.md && echo "STILL_IGNORED" || echo "ok-not-ignored"
test -f docs/repos/.gitkeep && echo "gitkeep-present" || echo "gitkeep-missing"
grep -q 'template' .claude/commands/setup-workspace.md \
  && grep -q 'exemplar' .claude/commands/setup-workspace.md \
  && grep -q 'docs/repos/<repo>.md' .claude/commands/setup-workspace.md \
  && grep -q -- '--refresh' .claude/commands/setup-workspace.md \
  && echo WS_PASS || echo WS_FAIL
```
Expected now: `STILL_IGNORED`, `gitkeep-missing`, `WS_FAIL`.

- [ ] **Step 2: Edit `.gitignore`** — change the unanchored reference-checkout rule so it only matches the top-level dir.

Replace the line:
```
repos/
```
with:
```
/repos/
```
(Anchored `/repos/` ignores only the top-level local reference checkouts, not `docs/repos/`.)

- [ ] **Step 3: Create `docs/repos/.gitkeep`** (empty file, so the committed index dir exists before any repo is indexed).

- [ ] **Step 4: Write `.claude/commands/setup-workspace.md`** (full content)

````markdown
---
description: Provision service repos into workspace/ (create from template/exemplar or clone existing), index each, and regenerate workspace/INDEX.md
---

# /setup-workspace

## Purpose

Onboarding Phase 3. Provisions service repos one at a time, for greenfield (create) and brownfield
(adopt) projects alike. New repos are created on the git host and seeded with **high-level config
only**; existing repos are cloned and **indexed** so Claude can navigate them quickly. All repos land
under `workspace/<repo>/`; `workspace/INDEX.md` and the per-repo `docs/repos/<repo>.md` files are
regenerated.

## Inputs

None required — interactive, repeated per repo. Reads `.claude/tracker.json` → `gitHost`. Run
`/setup-project` first if `gitHost` is missing or a placeholder. Optional `--refresh` argument: skip
provisioning; only re-index repos already cloned under `workspace/`.

## Steps

1. **Read `.claude/tracker.json` → `gitHost`** (platform, org). If missing or a placeholder, stop and
   tell the user to run `/setup-project` first.
2. **Loop over repos**, one at a time, until the user says they're done. For each, ask its **name**,
   **type** (frontend / backend / ML / infra / …), and **language/stack**, then pick a route:
   - **New from template** — ask for a git-host template repo; instantiate a new repo from it.
   - **New from exemplar** — ask for an existing repo to learn from; create an empty repo, then copy
     across only high-level config extracted from the exemplar (see below).
   - **Existing** — the repo already exists on the git host; clone it as-is.
3. **High-level config (new-repo routes only).** Keep only toolchain and quality config — never
   application code, dependency lockfiles, secrets, or content. Copy without asking:
   `.editorconfig`, `.gitattributes`, `.gitignore`, tool/language version pins (`.tool-versions`,
   `.nvmrc`, `.node-version`, `.sdkmanrc`), CI config (`.github/workflows/*`, `.gitlab-ci.yml`),
   linter/formatter config (eslint / prettier / ruff / checkstyle / pmd / spotless), `Dockerfile` /
   `.dockerignore`, `renovate.json`, `lombok.config`. For anything **not** on this list, ask the user
   whether it belongs before copying. Agent rules (`AGENTS.md` / `CLAUDE.md` / `.claude`) are **not**
   copied here — they come from `/propagate-rules`.
4. **Clone every repo** (created or existing) into `workspace/<repo>/`.
5. **Index each repo** → `docs/repos/<repo>.md` using the template below, by inspecting the checkout
   (stack, layout, entry points, build/test/run commands, detected conventions). `--refresh` runs only
   this step and step 6 over the already-cloned repos.
6. **Regenerate `workspace/INDEX.md`** — the overview table
   `Repo | Type | Language | Branch | Last sync | Index`, where `Index` links to
   `docs/repos/<repo>.md`, `Branch` is the checked-out branch, and `Last sync` is this run's date.
7. Remind the user: `workspace/*` is gitignored except `INDEX.md`; `docs/repos/*` **is** committed.

## Per-repo index template (`docs/repos/<repo>.md`)

    # <repo> — index

    **Type:** <frontend | backend | ML | infra | …>
    **Language / stack:** <e.g. Java 21 / Spring Boot>
    **Git host:** <platform>/<org>/<repo>
    **Last indexed:** <YYYY-MM-DD>

    ## Layout
    - <top-level dir> — <one-line purpose>

    ## Entry points
    - <bootstrap / main files or service entry modules>

    ## Commands
    - Build: <cmd>
    - Test: <cmd>
    - Run (local): <cmd>

    ## Conventions detected
    - <build tool, test framework, lint / format, CI, notable patterns>

## Config

Reads `.claude/tracker.json` → `gitHost` (platform + org). Does not touch `tracker` or `comms`.
````

- [ ] **Step 5: Run acceptance checks again (expect PASS)**

Run the Step 1 command. Expected: `ok-not-ignored`, `gitkeep-present`, `WS_PASS`.

- [ ] **Step 6: Commit**

```bash
git add .gitignore docs/repos/.gitkeep .claude/commands/setup-workspace.md
git commit -m "Extend /setup-workspace: template/exemplar create, brownfield indexing, INDEX overview; commit docs/repos"
```

---

### Task 3: Create /propagate-rules (Phase 6)

**Files:**
- Create: `.claude/commands/propagate-rules.md`

**Interfaces:**
- Consumes: `docs/conventions/*` (master, project-wide); `workspace/<repo>/` checkouts; `docs/repos/<repo>.md` (stack, from Task 2); `.claude/tracker.json` → `gitHost`.
- Produces: per-repo agent rules delivered via code PR; a rules-present signal `/check-setup` looks for.

- [ ] **Step 1: Acceptance checks (run first — expect FAIL)**

Run:
```bash
test -f .claude/commands/propagate-rules.md \
  && grep -q 'project-wide' .claude/commands/propagate-rules.md \
  && grep -q 'repo/language-specific' .claude/commands/propagate-rules.md \
  && grep -qi 'merge' .claude/commands/propagate-rules.md \
  && grep -qi 'PR' .claude/commands/propagate-rules.md \
  && echo PR_PASS || echo PR_FAIL
```
Expected now: `PR_FAIL` (file absent).

- [ ] **Step 2: Write `.claude/commands/propagate-rules.md`** (full content)

````markdown
---
description: Generate repo/language-specific agent rules from docs/conventions/ into each service repo via a code PR
---

# /propagate-rules

## Purpose

Onboarding Phase 6. Keeps the **project-wide** conventions in this docs repo as the single source, and
delivers the **repo/language-specific** slice into each service repo. Rules land via a **code PR** in
the service repo (respecting the code gate), merging with — never clobbering — any existing agent rules.

## Inputs

None required — interactive. Reads `docs/conventions/`, the `workspace/<repo>/` checkouts, their
`docs/repos/<repo>.md` indexes, and `.claude/tracker.json` → `gitHost`. Run `/setup-workspace` first so
the repos exist locally.

## Steps

1. **Read the master conventions** in `docs/conventions/` (architecture, naming, api, persistence,
   events, testing). These stay here — they are the project-wide source of truth and are **not** copied
   into repos.
2. **For each `workspace/<repo>/`**, derive only the **repo/language-specific** rules — the concrete,
   stack-bound guidance the cross-repo conventions imply for this repo (e.g. pytest + Playwright E2E for
   a TypeScript frontend; Maven/Spring layering + PMD rules for a Java service). Use the repo's
   `docs/repos/<repo>.md` index for its stack.
3. **Compose the repo's agent rules** — an `AGENTS.md` (and/or `.claude/` rules) containing:
   - a short **pointer** back to this docs repo's `docs/conventions/` for the project-wide standards;
   - the repo/language-specific rules from step 2.
4. **Merge, don't clobber.** If the repo already has `AGENTS.md` / `CLAUDE.md` / `.claude` content,
   append and reconcile rather than overwrite; preserve the team's existing rules.
5. **Deliver via PR.** Write the changes on a branch in `workspace/<repo>/` and open a code PR on the
   git host for human review/merge. A freshly-created, empty greenfield repo (no history to review
   against) may commit directly to its default branch instead.
6. **Report** one line per repo: the PR URL (or a direct-commit note) and whether rules were merged into
   existing ones or newly created.

## Config

Reads `.claude/tracker.json` → `gitHost`. Does not touch `tracker` or `comms`.
````

- [ ] **Step 3: Run acceptance checks again (expect PASS)**

Run the Step 1 command. Expected: `PR_PASS`.

- [ ] **Step 4: Commit**

```bash
git add .claude/commands/propagate-rules.md
git commit -m "Add /propagate-rules: repo/language-specific rules into service repos via PR"
```

---

### Task 4: Create /check-setup (Phase 7)

**Files:**
- Create: `.claude/commands/check-setup.md`

**Interfaces:**
- Consumes: `.claude/tracker.json`, `.env`, `.mcp.json`, `workspace/INDEX.md`, `docs/repos/<repo>.md`, `ROADMAP.md`; read-only tracker/git-host MCP probes.
- Produces: a `N/7 phases complete` checklist report (no writes).

- [ ] **Step 1: Acceptance checks (run first — expect FAIL)**

Run:
```bash
test -f .claude/commands/check-setup.md \
  && grep -qi 'read-only' .claude/commands/check-setup.md \
  && grep -q 'connectivity' .claude/commands/check-setup.md \
  && grep -q 'docs/repos/<repo>.md' .claude/commands/check-setup.md \
  && grep -q 'projectId' .claude/commands/check-setup.md \
  && echo CS_PASS || echo CS_FAIL
```
Expected now: `CS_FAIL` (file absent).

- [ ] **Step 2: Write `.claude/commands/check-setup.md`** (full content)

````markdown
---
description: Read-only audit that every onboarding phase completed — config, MCP connectivity, repos, tracker, rules
---

# /check-setup

## Purpose

Onboarding Phase 7. Read-only completeness audit. Reports ✅ / ⚠️ / ❌ per phase artifact with the exact
command to fix each gap. Changes nothing.

## Inputs

None. Reads the config files, `workspace/`, `docs/repos/`, `ROADMAP.md`, and (read-only) the
tracker/git-host MCPs.

## Checks

1. **Config present & non-placeholder** — `.claude/tracker.json` has real `tracker` + `gitHost` values
   (no `plane|github|gitlab` / `...` placeholders); `.env` exists with the chosen platforms' keys set;
   `.mcp.json` contains only the chosen platform block(s). Fix: `/setup-project` (Phase 0).
2. **MCP connectivity** — probe each configured MCP server with one read call; report ✅ / ❌. Fix:
   check `.env` tokens and `uv` on PATH (Plane), then re-run `/setup-project`.
3. **Docs-repo identity** — `git remote -v` origin is not the template, and the docs repo exists on the
   git host. Fix: `/setup-project` (Phase 1).
4. **Repos cloned & indexed** — every repo has a `workspace/<repo>/` checkout, a `docs/repos/<repo>.md`
   index, and a row in `workspace/INDEX.md`. Fix: `/setup-workspace` (or `--refresh`).
5. **Tracker project** — the configured `tracker.projectId` resolves on the tracker. Fix:
   `/setup-project` (Phase 4).
6. **ROADMAP seeded** — `ROADMAP.md` has real feature rows (not just the header/example). Fix:
   `/setup-project` (Phase 2).
7. **Rules propagated** — each service repo has agent rules referencing `docs/conventions/`. Fix:
   `/propagate-rules`.

## Output

A checklist, one line each `✅|⚠️|❌ <check> — <detail> [fix: <command>]`, then a one-line summary
`N/7 phases complete`.

## Config

Reads `.claude/tracker.json` (`tracker` + `gitHost`). Read-only; writes nothing.
````

- [ ] **Step 3: Run acceptance checks again (expect PASS)**

Run the Step 1 command. Expected: `CS_PASS`.

- [ ] **Step 4: Commit**

```bash
git add .claude/commands/check-setup.md
git commit -m "Add /check-setup: read-only onboarding completeness audit"
```

---

### Task 5: Rewrite /setup-project as the Phase 0–7 orchestrator

**Files:**
- Rewrite: `.claude/commands/setup-project.md`

**Interfaces:**
- Consumes: `.env.example` keys (Task 1), the behaviors of `/setup-workspace` (Task 2), `/propagate-rules` (Task 3), `/check-setup` (Task 4).
- Produces: `.claude/tracker.json` (full), `.mcp.json` (pruned to chosen platforms), `.env`, a filled `CLAUDE.md` project-context section, a seeded `ROADMAP.md`, and `tracker.projectId`.

- [ ] **Step 1: Acceptance checks (run first — expect FAIL)**

Run:
```bash
for kw in "Phase 0" "Phase 1" "Phase 7" "Connectivity probe" "project name" \
          "git remote remove origin" "Create-or-adopt" "structure only" \
          "/setup-workspace" "/propagate-rules" "/check-setup"; do
  grep -q "$kw" .claude/commands/setup-project.md || { echo "MISSING: $kw"; }
done; echo "checks done"
```
Expected now: several `MISSING:` lines (current file predates the orchestrator).

- [ ] **Step 2: Write `.claude/commands/setup-project.md`** (full content)

````markdown
---
description: One-time init orchestrator — runs onboarding Phases 0–7 for greenfield or brownfield projects
---

# /setup-project

## Purpose

One-time init **orchestrator**. Runs the eight onboarding phases (0–7) in order, working the same for
greenfield (create everything) and brownfield (adopt what exists) projects, branching create-vs-fetch
per resource. Owns Phases 0, 1, 2, 4, 5 inline; delegates Phase 3 to `/setup-workspace`, Phase 6 to
`/propagate-rules`, and Phase 7 to `/check-setup`.

## Inputs

None required — fully interactive. Every phase is **idempotent**: detect "already done" from its
artifacts and offer skip/redo. On re-run, resume at the first incomplete phase.

## Phases

### Phase 0 — Access & connectivity

1. Prompt for the **tracker** (`plane` | `github` | `gitlab`) + org + `projectId` (leave blank if it
   will be created in Phase 4); the **git host** (`github` | `gitlab`) + org (may equal the tracker —
   Plane is issues-only, so repos still need a git host); an **optional comms** channel (`slack` |
   `discord`) + id (skip the block entirely if declined).
2. Write `.claude/tracker.json` (`tracker`, `gitHost`, optional `comms`) per the base design schema.
3. Write `.mcp.json` with **only** the chosen tracker and (if different) git-host blocks — strict JSON,
   no comments, drop unused blocks. Plane runs via `uvx plane-mcp-server stdio` (needs `uv` on PATH);
   confirm the exact server command/package per platform (GitHub's official MCP may ship as a
   container/binary rather than an npm package).
4. Write `.env` from `.env.example`, keeping the chosen platforms' keys, values left blank.
5. **Connectivity probe** — for each configured MCP server, make one cheap read call and report ✅ / ❌:
   Plane → list workspace projects; GitHub → get the authenticated user / list org repos; GitLab → get
   the current user / list group projects. On ❌, **stop** with a specific fix hint (missing token, `uv`
   not installed, wrong `PLANE_BASE_URL`). Do not proceed until every configured server is green.

### Phase 1 — Docs-repo identity

1. Prompt for the **project name** freely — do **not** auto-append `-docs`.
2. Rewrite `<project>` placeholders across `README.md`, `CLAUDE.md`, `ROADMAP.md`.
3. Rename the working directory to the chosen name.
4. Detach the template: `git remote remove origin`. Idempotent — skip if origin is already non-template.
5. **Create-or-adopt** the repo on the git host (`gitHost` from Phase 0) and push. If a repo of that
   name already exists on the host, adopt it (set origin, push) instead of creating a new one.

### Phase 2 — Project context

1. Fill the `CLAUDE.md` project-context section: project name, stack, tracker line (`platform`, `org`,
   `projectId`), git-host line (`platform`, `org`), comms line (if configured). Leave the service-repo
   list for Phase 3.
2. **Ingest the approved estimate** — via the estimate MCP if available (list/get the approved
   estimation), else ask the user for the export — pulling the feature/epic list, priorities, NFRs, and
   documented assumptions.
3. **Seed `ROADMAP.md`** — one row per feature/epic in the `Feature | Priority | Level | Spec | Issues |
   Milestone | Link` table (`Level` and later columns start empty). Add short "Non-functional
   requirements" and "Assumptions" sections below the table.

### Phase 3 — Repos

Run **`/setup-workspace`** — creates repos from a template/exemplar (high-level config only) or clones
existing ones, and indexes each into `docs/repos/<repo>.md`.

### Phase 4 — Tracker project

Create or fetch the tracker project, then record its id in `.claude/tracker.json` → `tracker.projectId`:

| Platform | "Tracker project" is | Create | Fetch |
|----------|----------------------|--------|-------|
| Plane | a Plane project | create a project (name = the docs project name) via the Plane MCP | list projects, match by name/id |
| GitHub | the git repo + its milestones | ensure the repo exists (from Phase 1/3); no separate object | the existing repo |
| GitLab | the project (repo); epics at group level | ensure the project/group exists | the existing project |

### Phase 5 — Populate tracker (new project only; skip when fetched)

Ask the **architect** which population level they want:
- **Structure only (default)** — create epics/milestones mirroring the ROADMAP features (priority,
  level). **No child issues** — those come per-feature via `/create-issues` after each spec PR merges,
  preserving the spec-driven gate.
- **Full backlog** — epics + all child issues up front (bypasses the spec gate; offer, don't default).
- **Project only** — populate nothing; ROADMAP stays the backlog.

Use the tracker-adapter verbs in `docs/process/README.md` (Plane cycles/epics, GitHub
milestones/sub-issues, GitLab milestones/epics).

### Phase 6 — Propagate rules

Run **`/propagate-rules`** — project-wide conventions stay here; repo/language-specific rules go to each
service repo via PR.

### Phase 7 — Completeness check

Run **`/check-setup`** and show the report. Then print next steps: `/new-feature <name>` to start the
first feature.

## Config

Writes `.claude/tracker.json` in full (`tracker`, `gitHost`, optional `comms`) — this command is the
source of that file — plus `.mcp.json` and `.env`. On re-run, reads the existing files to detect prior
setup and resume at the first incomplete phase.
````

- [ ] **Step 3: Run acceptance checks again (expect PASS)**

Run the Step 1 command. Expected: `checks done` with **no** `MISSING:` lines.

- [ ] **Step 4: Cross-reference check** — the three delegated command files exist so the links resolve.

Run:
```bash
for f in setup-workspace propagate-rules check-setup; do
  test -f ".claude/commands/$f.md" && echo "ok $f" || echo "MISSING FILE $f"
done
```
Expected: three `ok` lines. (If any is missing, complete Tasks 2–4 first.)

- [ ] **Step 5: Commit**

```bash
git add .claude/commands/setup-project.md
git commit -m "Rewrite /setup-project as Phase 0-7 greenfield/brownfield orchestrator"
```

---

### Task 6: Document onboarding in the process doc

**Files:**
- Modify: `docs/process/README.md` (add an "Onboarding (one-time)" section before "## Per-feature flow")

**Interfaces:**
- Produces: the tool-neutral description of the phase model (source of truth the commands adapt).

- [ ] **Step 1: Acceptance checks (run first — expect FAIL)**

Run:
```bash
grep -q 'Onboarding (one-time)' docs/process/README.md \
  && grep -q 'docs/repos/<repo>.md' docs/process/README.md \
  && echo PROC_PASS || echo PROC_FAIL
```
Expected now: `PROC_FAIL`.

- [ ] **Step 2: Insert this section** into `docs/process/README.md` immediately **before** the line `## Per-feature flow`:

```markdown
## Onboarding (one-time)

Before feature work, `/setup-project` runs an eight-phase init that works the same for greenfield
(create everything) and brownfield (adopt what exists), branching create-vs-fetch per resource. Every
phase is idempotent; a re-run resumes at the first incomplete phase.

| Phase | What it does |
|-------|--------------|
| 0 Access | choose tracker / git-host / comms; write `.env` + `.mcp.json`; probe each MCP server (must be green to proceed) |
| 1 Identity | pick a project name; rewrite placeholders; detach the template; create/adopt + push the docs repo |
| 2 Context | fill `CLAUDE.md`; ingest the estimate; seed `ROADMAP.md` |
| 3 Repos | `/setup-workspace` — create (template/exemplar, high-level config only) or clone + index existing |
| 4 Tracker | create or fetch the tracker project |
| 5 Populate | new project only — architect picks structure-only (default) / full backlog / project-only |
| 6 Rules | `/propagate-rules` — project-wide conventions stay here; repo/language-specific rules → each repo via PR |
| 7 Check | `/check-setup` — audit every phase artifact |

`docs/conventions/` are project-wide (this repo). Repo/language-specific rules live in each service
repo, delivered by `/propagate-rules`. Per-repo navigation indexes live in `docs/repos/<repo>.md`.
```

- [ ] **Step 3: Run acceptance checks again (expect PASS)**

Run the Step 1 command. Expected: `PROC_PASS`.

- [ ] **Step 4: Commit**

```bash
git add docs/process/README.md
git commit -m "Document the 8-phase onboarding model in the process doc"
```

---

### Task 7: Update README quick start + Plane prerequisite

**Files:**
- Modify: `README.md`

**Interfaces:**
- Consumes: the command surface from Tasks 2–5.

- [ ] **Step 1: Acceptance checks (run first — expect FAIL)**

Run:
```bash
grep -qi 'uv' README.md \
  && grep -q 'orchestrat' README.md \
  && grep -q '/check-setup' README.md \
  && grep -q 'docs/repos/' README.md \
  && echo RM_PASS || echo RM_FAIL
```
Expected now: `RM_FAIL`.

- [ ] **Step 2: Edit the "## Prerequisites" section** — add a bullet after the tracker/git-host bullet:

```markdown
- MCP runtimes: the Plane MCP server runs via `uvx` — install [uv](https://docs.astral.sh/uv/) if you
  use Plane. The GitHub/GitLab MCP servers run via `npx` (Node). Missing runtimes are caught by the
  Phase 0 connectivity probe in `/setup-project`.
```

- [ ] **Step 3: Replace the "## Quick start" section** with:

```markdown
## Quick start

`/setup-project` is the **orchestrator** — it runs the full eight-phase onboarding (0–7) and works the
same whether you're starting fresh (greenfield) or adopting existing repos/tracker (brownfield):

1. `/setup-project` — Phase 0–7: configure + probe access, rename/detach this repo, fill context, seed
   `ROADMAP.md`, provision repos, create/fetch the tracker project, propagate rules, and audit the result.

The phase commands are also runnable on their own, and are re-runnable/idempotent:

- `/setup-workspace` — provision service repos (create from template/exemplar or clone) and index them.
- `/propagate-rules` — push repo/language-specific rules into the service repos via PR.
- `/check-setup` — read-only audit of onboarding completeness.

Then `/new-feature <name>` — brainstorm and write `docs/<name>/spec.md`, and open a spec PR.
```

- [ ] **Step 4: Edit the "## Repo layout" block** — add these two lines (keep the existing ones):

```
docs/repos/            per-repo navigation indexes (committed): stack, layout, entry points, commands
.claude/commands/      /setup-project (orchestrator), /setup-workspace, /propagate-rules, /check-setup, /new-feature, ...
```

(Replace the existing `.claude/commands/` line with the one above so it lists the new commands.)

- [ ] **Step 5: Run acceptance checks again (expect PASS)**

Run the Step 1 command. Expected: `RM_PASS`.

- [ ] **Step 6: Commit**

```bash
git add README.md
git commit -m "Update README: orchestrator quick start, new phase commands, uv prerequisite"
```

---

## Self-review

**Spec coverage** — every spec section maps to a task:
- Part 1 Plane fix (`.mcp.json`, `.env.example`, `uv` prereq) → Tasks 1, 7.
- Phase 0 access + connectivity probe → Task 5.
- Phase 1 identity (free-choice name, detach, create/adopt) → Task 5.
- Phase 2 context (CLAUDE.md, estimate, ROADMAP) → Task 5.
- Phase 3 repos (template/exemplar high-level config, brownfield index, two-tier INDEX) → Task 2.
- Phase 4 tracker create/fetch (per-platform verbs) → Task 5.
- Phase 5 architect-chosen population → Task 5.
- Phase 6 propagate rules (two-tier, PR, merge-not-clobber, pointer) → Task 3.
- Phase 7 completeness check → Task 4.
- Process-doc onboarding section → Task 6.
- `docs/repos/` committed (gitignore fix) → Task 2 (the spec said "allow docs/repos"; the real cause is the unanchored `repos/` rule matching `docs/repos/` — Task 2 anchors it to `/repos/`).

**Placeholder scan** — no "TBD/TODO/implement later". `<project>`/`<repo>`/`<org>`/`…` are intentional template placeholders per the Global Constraints, not plan gaps. Every command file's full content is written out.

**Type/name consistency** — command names (`/setup-project`, `/setup-workspace`, `/propagate-rules`, `/check-setup`) match across the orchestrator (Task 5), the process doc (Task 6), the README (Task 7), and the file map. The `INDEX.md` column set `Repo | Type | Language | Branch | Last sync | Index` and the `docs/repos/<repo>.md` path are identical in Tasks 2, 4, 5, 6. Env var names (`PLANE_API_KEY`, `PLANE_WORKSPACE_SLUG`, `PLANE_BASE_URL`) match across Tasks 1 and 5.
