---
name: setup-project
description: One-time init orchestrator — runs onboarding Phases 0–6 for greenfield or brownfield projects
disable-model-invocation: true
---

# /setup-project

## Purpose

One-time init **orchestrator**. Runs the seven onboarding phases (0–6) in order, working the same
for greenfield (create everything) and brownfield (adopt what exists) projects, branching
create-vs-fetch per resource. Owns Phases 0, 1, 2, 4 inline; delegates Phase 3 to
`/setup-workspace`, Phase 5 to `/propagate-rules`, and Phase 6 to `/check-setup`.

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
2. Rewrite the template's name placeholders to the chosen name: the `\<project\>-docs` title token in
   `README.md` (replace the whole token — the chosen name is used verbatim, with no forced `-docs`
   suffix) and the `<project>` field in `CLAUDE.md`. (`ROADMAP.md` has no name placeholder.)
3. Rename the working directory to the chosen name.
4. Detach the template: `git remote remove origin`. Idempotent — skip if origin is already non-template.
5. **Create-or-adopt** the repo on the git host (`gitHost` from Phase 0) and push. If a repo of that
   name already exists on the host, adopt it (set origin, push) instead of creating a new one.

### Phase 2 — Project context

1. Fill the `CLAUDE.md` project-context section: project name, stack, tracker line (`platform`, `org`,
   `projectId`), git-host line (`platform`, `org`), comms line (if configured). Leave the service-repo
   list for Phase 3.
2. **Import solution architecture (adopt-or-scaffold).** Prompt the user: "Do you have an
   existing solution-architecture set to import?" [y/n].

   - **y (adopt / brownfield)** — accept a local dir path, a git URL to clone from, or an
     Obsidian vault export path. Copy contents into `docs/architecture/current/` verbatim,
     preserving the original layout. Then **delegate to `/migrate-arch`** to reshape into the
     schema (see the `/migrate-arch` skill for the flow — it opens a PR against the current
     branch). Phase 2 completes when `/migrate-arch`'s PR is either merged or the user opts to
     defer review; either way, `_migration-quarantine/` and unfilled `ARCH_GAP`s become the
     project's punch list.

   - **n (scaffold / greenfield)** — scaffold all 20 required files per the schema at
     `@.claude/skills/check-setup/arch-schema.md`. For each required file:
     1. Create the file with the title heading (system name for `README.md`; the file's role
        for others).
     2. Write every required section heading in the specified order.
     3. Under each required section, write one `ARCH_GAP` marker in the standard format:

            <!-- ARCH_GAP: required section unfilled.
                 Section: <section name>.
                 Fill with: <hint from schema>.
                 See: .claude/skills/check-setup/arch-schema.md#<anchor> -->

     4. Do not pre-fill any content beyond scaffolding — consistency with brownfield migration
        (both greenfield and brownfield land in "gap-filled fail state" the same way).

     **Single-table files** — `glossary.md`, `bizbok/stakeholders.md`, and
     `bizbok/information-map.md` declare `Required content` (a single table), not
     `Required sections`. For these, place one `ARCH_GAP` marker where the table would go
     and put the required column list in its `Fill with:` hint. The `Section:` field names
     the table's role (e.g. `Section: Stakeholder catalog table`).

     For `ddd/contexts/`, scaffold one placeholder file `ddd/contexts/example.md` following
     the same 4-step procedure using the required sections from
     `@.claude/skills/check-setup/arch-schema.md#ddd-context-file`. The human renames this
     file on first real context.

     Lens READMEs (`bizbok/README.md`, `ddd/README.md`, `c4/README.md`) get a 2–3 sentence
     explainer stub instead of `ARCH_GAP` — the schema tells the scaffolder what each lens
     is for, so this content is known at setup time. Include the cross-links to the other
     two lens READMEs.

     **Do NOT scaffold** anything under `modularity/` (populated later by `/modularize`)
     or `estimation/` (optional; not gated at setup).

   - **Log the initial state** — create
     `docs/architecture/logs/YYYY-MM-DD-initial-scaffold/README.md` with a real block:

           # initial-scaffold — adopt schema-driven arch scaffold

           **Date**: YYYY-MM-DD
           **Codename**: initial-scaffold

           **Driver**: project setup — /setup-project Phase 2

           **Decision**: adopt the schema-driven arch scaffold defined in
           .claude/skills/check-setup/arch-schema.md

           **Rationale**: every project starts from the same 20-file skeleton so downstream
           skills (/new-feature, /spec-review, /modularize) can rely on a stable shape.

           **Alternatives rejected**:
           - Freestyle (any layout) — rejected: makes tooling brittle and prevents cross-project
             navigation.

           **Impact**:
           - + docs/architecture/current/** (20 required files scaffolded)
           - + docs/architecture/logs/YYYY-MM-DD-initial-scaffold/README.md

           **Links**:
           - .claude/skills/check-setup/arch-schema.md
           - docs/superpowers/specs/2026-09-18-arch-current-schema-design.md
3. **Ingest estimate — OPTIONAL.** Prompt: "Do you have an approved estimate to import?" [y/n]:
   - **y** — extract NFRs and Assumptions from the estimate and write them directly into
     `docs/architecture/current/nfrs.md` and `docs/architecture/current/assumptions.md`,
     replacing the `ARCH_GAP` markers left by step 2's scaffold. The extracted content fills
     the schema's required sections (Availability / Performance / Security / Compliance /
     Observability for `nfrs.md`; Scope assumptions / Design decisions (locked) / Open
     questions for `assumptions.md`); any section the estimate doesn't cover keeps its
     `ARCH_GAP` marker. Do **not** create ROADMAP feature rows and do **not** decompose the
     tree from the estimate.
   - **n** — skip; NFRs + Assumptions files stay as scaffolded placeholders.
4. **Seed `ROADMAP.md` as a shell:**

       # ROADMAP

       The living sprint plan: one row per feature/task in a planned sprint. Rows are
       added by `/plan-sprint` at each sprint boundary — not seeded upfront.

       | Feature | Priority | Level | Module | Spec | Issues | Milestone | Link |
       |---------|----------|-------|--------|------|--------|-----------|------|

       Live status lives in the tracker; `/status` reads it.

   Empty table (no rows). Do not add NFRs / Assumptions sections here — they live in the arch
   vault (`docs/architecture/current/nfrs.md` and `assumptions.md`) scaffolded in step 2.

Phase 2 stops here. Every scaffolded file has one `ARCH_GAP` marker per required section, so
`/check-setup` (Phase 6) will report the full punch list. This is expected — the report *is* the
list of gaps to fill. No modularization is run at setup. The DM/PL iterates on `current/` (with
paired arch-log entries per `/new-feature`) until the architecture is stable, then runs
`/modularize` at root, then `/plan-sprint`.

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

### Phase 5 — Propagate rules

Run **`/propagate-rules`** — project-wide conventions stay here; repo/language-specific rules go to each
service repo via PR.

### Phase 6 — Completeness check

Run **`/check-setup`** and show the report. Then print next steps: `/new-feature <name>` to add the
first capability to the solution architecture, followed by `/new-spec <name>` to write its spec.

## Config

Writes `.claude/tracker.json` in full (`tracker`, `gitHost`, optional `comms`) — this command is the
source of that file — plus `.mcp.json` and `.env`. On re-run, reads the existing files to detect prior
setup and resume at the first incomplete phase.
