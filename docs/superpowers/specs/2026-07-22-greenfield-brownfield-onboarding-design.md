# Greenfield/Brownfield Onboarding — Design

**Date:** 2026-07-22
**Status:** Approved design, pre-implementation
**Builds on:** `docs/superpowers/specs/2026-07-15-spec-driven-docs-template-design.md` (the base template design).

## Purpose

Make the template's init work identically for **greenfield** (create everything) and **brownfield**
(adopt what exists) projects. Today's `/setup-project` + `/setup-workspace` only configure ids and
create/clone repos; there is no repo detach/rename, no repo indexing, no tracker-project
create/fetch, no rule propagation into service repos, and no completeness check. This design turns
init into a single guided **orchestrator** that runs an 8-phase skeleton (Phase 0–7), branching
create-vs-fetch **per resource** so both paths share one flow, plus a proven Plane MCP fix.

## Scope

- **In:** the Plane MCP fix; `/setup-project` reworked as a phase 0→7 orchestrator; `/setup-workspace`
  extended with template/exemplar creation and brownfield indexing; two new commands
  (`/propagate-rules`, `/check-setup`); the per-repo index artifacts; the doc/config ripple.
- **Out (YAGNI):** git hooks/watchers for auto-refreshing indexes (refresh is command-driven); deep
  architecture/dependency maps (lightweight index only); full-scaffold repo creation (high-level
  config only); auto-populating child issues at init (gated per-feature by `/create-issues`); any new
  tracker/git-host platforms beyond Plane/GitHub/GitLab.

## Writing principle

Same as the base design: concise and to the point for a technical audience — but complete. The
process docs are the tool-neutral source of truth; `.claude/` commands are only the execution adapter.

## Part 1 — Plane MCP fix

The shipped `.mcp.json` Plane block (`npx @makeplane/mcp-server`, env `PLANE_API_TOKEN` /
`PLANE_WORKSPACE_SLUG`) does not connect. Replace it with the proven reference:

```json
"plane": {
  "command": "uvx",
  "args": ["plane-mcp-server", "stdio"],
  "env": {
    "PLANE_API_KEY": "${PLANE_API_KEY}",
    "PLANE_WORKSPACE_SLUG": "${PLANE_WORKSPACE_SLUG}",
    "PLANE_BASE_URL": "${PLANE_BASE_URL}"
  }
}
```

Ripple:
- `.env.example` — rename `PLANE_API_TOKEN` → `PLANE_API_KEY`; add `PLANE_BASE_URL`. Document the DAC
  defaults in comments: `PLANE_WORKSPACE_SLUG=dac-digital`, `PLANE_BASE_URL=https://plane.dac.digital`.
- Values stay `${...}` env vars (not hardcoded) so the template remains reusable; the DAC values are
  documented defaults, not baked in.
- Prerequisite note (README + `/setup-project` Phase 0): the Plane MCP server runs via `uvx`, so `uv`
  must be on PATH — it is **not** an npx package.

## Part 2 — Onboarding phase model

`/setup-project` becomes the **orchestrator**. It runs Phases 0→7 in order. Every phase is
**idempotent**: it detects "already done" and offers skip/redo rather than duplicating work. Each
resource phase branches **create (greenfield)** vs **fetch/adopt (brownfield)**; a project may mix
both (some repos new, some existing).

| Phase | Name | Owner | Create path (greenfield) | Fetch path (brownfield) |
|-------|------|-------|--------------------------|--------------------------|
| **0** | Access & connectivity | `/setup-project` inline | choose tracker + git-host + optional comms; write `.env` + `.mcp.json` (chosen blocks only); **probe each MCP server** and report ✅/❌ | same |
| **1** | Docs-repo identity | `/setup-project` inline | prompt for a **free-choice name**; rewrite `<project>` placeholders; rename local dir; detach template origin; **create + push** the docs repo on the git host | adopt the repo if it already exists on the host |
| **2** | Project context | `/setup-project` inline | fill `CLAUDE.md` context; ingest approved estimate; seed `ROADMAP.md` | same |
| **3** | Repos | `/setup-workspace` | create from **template or exemplar** (high-level config only, interactive) | clone + build `docs/repos/<repo>.md` index |
| **4** | Tracker project | `/setup-project` inline | **create** the tracker project | **fetch** existing; record `projectId` |
| **5** | Populate tracker | `/setup-project` inline | **architect chooses** level (default structure-only) | skip |
| **6** | Propagate rules | `/propagate-rules` | project-wide conventions stay here; repo/language-specific rules → each repo via PR | same |
| **7** | Completeness check | `/check-setup` | validate every phase's artifacts + connections | same |

### Phase 0 — Access & connectivity

Prompts for tracker platform (`plane`/`github`/`gitlab`), git-host platform (`github`/`gitlab`, may
equal the tracker), and an optional comms channel — as today. Then two additions:

1. Write `.mcp.json` containing **only** the chosen platform blocks (strict JSON, no comments — drop
   unused blocks). Confirm the exact server command/package per platform (GitHub's official MCP ships
   as a container/binary; Plane runs via `uvx`).
2. **Connectivity probe:** for each configured MCP server, make one cheap read call (list projects /
   whoami). Report ✅/❌ per server. A failed probe (missing token, `uvx` absent, wrong base URL)
   **stops the flow here** with a specific fix hint. This guardrail makes a broken tracker MCP (the
   original Plane breakage) impossible to miss before later phases depend on it.

Writes `.env` from `.env.example` (chosen platforms' keys, values blank) and the `tracker`/`gitHost`/
optional `comms` blocks of `.claude/tracker.json`.

### Phase 1 — Docs-repo identity

- Prompt for the project name **freely** (do not auto-derive `<project>-docs`).
- Rewrite `<project>` placeholders across `README.md`, `CLAUDE.md`, `ROADMAP.md`.
- Rename the working directory to the chosen name.
- `git remote remove origin` to detach from the template. Idempotent: skip if origin is already
  non-template.
- **Create-or-adopt** the repo on the configured git host and push. Depends on Phase 0 git-host access.

### Phase 2 — Project context

Fill the `CLAUDE.md` project-context section (name, stack, tracker/git-host/comms lines; service-repo
list left for Phase 3). Ingest the approved estimate (estimate MCP if available, else the user's
export) and seed `ROADMAP.md` (feature/epic rows + NFRs + assumptions), as in the base design.

### Phase 3 — Repos (`/setup-workspace`, extended)

Per repo, one of three routes:

- **New from template** — point at a git-host template repo; instantiate; the agent then inspects the
  pulled files and keeps only **high-level** config (toolchain, CI, quality gates, tool/language
  versions, Dockerfile if present), **asking interactively** when a file's usefulness is ambiguous. No
  application code. Agent rules are **not** copied here — they come from Phase 6.
- **New from exemplar** — point at an existing repo; same high-level-config extraction; nothing
  app-specific.
- **Existing** — clone into `workspace/<repo>/`; generate `docs/repos/<repo>.md` (stack, layout, entry
  points, build/test/run commands, detected conventions).

**Index artifacts (two-tier):**
- `workspace/INDEX.md` stays the lean **overview** table — `Repo | Type | Language | Branch | Last sync`
  — plus one column linking each row to its detail file. Committed (as today).
- `docs/repos/<repo>.md` holds per-repo **depth**. Committed to the docs repo so it is referenceable
  without cloning the service repo. Per-repo files keep refreshes scoped (no shared-file churn).

**Maintenance:** refresh-on-rerun — re-running `/setup-workspace` (or a `--refresh` flag) regenerates
the indexes and updates `INDEX.md` last-sync. No git hooks; staleness is visible via last-sync.

### Phases 4–5 — Tracker

- **Phase 4:** create a new tracker project (greenfield) or fetch an existing one (brownfield); record
  `projectId` in `.claude/tracker.json`.
- **Phase 5 (new project only):** prompt the **architect** for the population level:
  - **Structure only (default)** — create epics/milestones mirroring the ROADMAP features (priority,
    level). No child issues; those come per-feature via `/create-issues` after each spec PR merges,
    preserving the spec-driven gate.
  - **Full backlog** — epics + all child issues up front (bypasses the spec gate; offered, not default).
  - **Project only** — populate nothing; ROADMAP stays the backlog.

  Uses the tracker-adapter verb table already in `docs/process/README.md` (Plane cycles/epics, GitHub
  milestones/sub-issues, GitLab milestones/epics).

### Phase 6 — Propagate rules (`/propagate-rules`, new)

Two tiers:
- **Project-wide** conventions stay in `docs/conventions/` — single source, **not** copied into repos.
- **Repo/language-specific** rules are generated per repo (e.g. pytest + Playwright for a TS frontend;
  Maven/Spring specifics for a Java service), written into `workspace/<repo>/` on a branch, and opened
  as a **code PR** for human review/merge — respecting the process's code gate. Each generated ruleset
  includes a short pointer back to this repo's `docs/conventions/`. Existing `AGENTS.md`/`CLAUDE.md`/
  `.claude` content is **merged/appended, never clobbered**. Freshly-created empty greenfield repos may
  commit directly (nothing to review against).

### Phase 7 — Completeness check (`/check-setup`, new)

Read-only audit → a checklist report, each item ✅/⚠️/❌ with the exact command to fix a gap:
- config files present and non-placeholder (`.env`, `.mcp.json`, `.claude/tracker.json`);
- MCP connectivity probes green;
- docs repo detached from template + pushed to the git host;
- every workspace repo cloned and indexed (`docs/repos/<repo>.md` + `INDEX.md` row);
- tracker project reachable; ROADMAP seeded;
- per-repo rules propagated (PR opened/merged).

## File changes

**New:**
- `.claude/commands/propagate-rules.md` — Phase 6.
- `.claude/commands/check-setup.md` — Phase 7.
- `docs/repos/` — per-repo index files (ships with `.gitkeep`; committed).

**Changed:**
- `.mcp.json` — Plane block fixed (uvx) per Part 1.
- `.env.example` — `PLANE_API_KEY` + `PLANE_BASE_URL`; DAC defaults in comments.
- `.claude/commands/setup-project.md` — rewritten as the Phase 0→7 orchestrator (connectivity probe;
  rename/detach/create-push with free-choice name; create-or-fetch tracker project; interactive Phase 5
  population; delegates to `/setup-workspace`, `/propagate-rules`; ends with `/check-setup`).
- `.claude/commands/setup-workspace.md` — template/exemplar create routes (interactive high-level-config
  extraction); brownfield indexing into `docs/repos/<repo>.md`; `INDEX.md` link column; refresh-on-rerun.
- `docs/process/README.md` — short "Onboarding" subsection documenting the phase model (source of truth;
  commands are the adapter).
- `README.md` — quick start reflects the orchestrator + new phases; Plane `uv`/`uvx` prerequisite note.
- `.gitignore` — allow `docs/repos/` (committed).

## Acceptance

- Plane MCP block matches the proven reference; `.env.example` and docs align; `uvx` prerequisite noted.
- `/setup-project` runs Phases 0→7, each idempotent, branching create/fetch per resource; greenfield and
  brownfield both complete without a separate command path.
- Phase 0 probe reports per-server ✅/❌ and stops on failure with a fix hint.
- Phase 1 renames to a **user-chosen** name, detaches the template origin, and creates+pushes the docs
  repo (or adopts an existing one).
- `/setup-workspace` supports template/exemplar creation (high-level config only) and produces
  `docs/repos/<repo>.md` + an `INDEX.md` overview row per repo; re-run refreshes them.
- Phase 5 prompts the architect for the population level; default is structure-only (no child issues).
- `/propagate-rules` keeps project-wide conventions here and delivers repo/language-specific rules per
  repo via PR, merging (not clobbering) existing rules.
- `/check-setup` reports pass/fail per phase artifact with fix hints.

## Open items for implementation planning

- Exact `docs/repos/<repo>.md` section template (headings for stack/layout/entrypoints/commands/conventions).
- Per-platform verbs for **creating** a tracker project and epics/milestones (Plane/GitHub/GitLab) — the
  base doc covers issue/milestone verbs but not project creation.
- The connectivity-probe call per MCP server (which cheap read verifies auth without side effects).
- How `/setup-project` invokes the sub-commands (delegate vs inline the phase logic) and how a re-run
  resumes at the first incomplete phase.
- Interactive extraction heuristic for "high-level config" — the concrete allow/ask list of config files.
