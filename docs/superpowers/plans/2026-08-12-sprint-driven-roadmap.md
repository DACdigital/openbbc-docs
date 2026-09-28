# Sprint-Driven ROADMAP + Phase 2 Rework — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Restructure onboarding from 8 phases to 7 (drop Phase 5), rework Phase 2 to make
estimate optional and stop seeding ROADMAP feature rows upfront, move NFRs + Assumptions into the
arch vault, add the `/plan-sprint` skill that grows ROADMAP one sprint at a time, and add the
`docs → tracker one-way direction` hard rule to AGENTS.md.

**Architecture:** Docs-only. Adds `/plan-sprint`; reshapes `/setup-project` Phase 2 (replacing
what sub-project 1 added); drops Phase 5; renumbers Phase 6/7 → 5/6; updates `/check-setup`,
`/new-feature`, `ROADMAP.md`, process docs, and README.

**Tech Stack:** Markdown, YAML front-matter.

**Commit policy:** Per standing instruction, do not commit the spec or plan. Implementation edits
may be committed.

**Spec:** `docs/superpowers/specs/2026-08-12-sprint-driven-roadmap-design.md`

**Prerequisites:** sub-projects (1) and (2) plans complete.

---

## File Structure

- **Create**:
  - `.claude/skills/plan-sprint/SKILL.md`
- **Modify**:
  - `.claude/skills/setup-project/SKILL.md` — Phase 2 full rework; Phase 5 removal; renumber Phase 6/7.
  - `.claude/skills/check-setup/SKILL.md` — swap ROADMAP-rows check for ROADMAP-shell check; add NFRs/Assumptions check.
  - `.claude/skills/new-feature/SKILL.md` — add reads of `nfrs.md`, `assumptions.md`, and modularity node README.
  - `ROADMAP.md` — new schema with `Module` column; NFRs + Assumptions removed.
  - `docs/process/README.md` — phase table, per-feature flow, tracker-adapter reference.
  - `docs/process/AGENTS.md` — add one-way-direction hard rule; mention `/plan-sprint`.
  - `README.md` — Quick start section + repo-layout tweaks.

---

## Task 1 — Create `/plan-sprint` skill

**Files:**
- Create: `.claude/skills/plan-sprint/SKILL.md`

- [ ] **Step 1: Write the skill file**

Create `.claude/skills/plan-sprint/SKILL.md` with this content:

```markdown
---
name: plan-sprint
description: Interactive sprint planning — reads the modularity tree, shows candidate scope (unexpanded L1/L2 placeholders + in-flight items), asks DM to pick sprint contents (from tree + ad-hoc), creates the tracker milestone, and appends rows to ROADMAP.md. Grows the plan one sprint at a time. Supports --refine on the currently active sprint.
disable-model-invocation: true
---

# /plan-sprint `[--refine]`

## Purpose

Sprint-boundary ceremony for the DM. Grows `ROADMAP.md` one sprint at a time by picking a subset
of the modularity tree's L2 nodes plus any ad-hoc items, then creating the tracker milestone that
holds them. Never touches the tree.

## Actor

DM (Delivery Manager). No hard gate — anyone with tracker write access can invoke.

## Inputs

- `--refine` (optional flag) — operate on the currently active sprint (the one whose tracker
  milestone is still open) instead of planning a new one.

## Steps

### Normal mode (no `--refine`)

1. **Compute default sprint name** — read `ROADMAP.md` `Milestone` column and the tracker
   milestones to find the highest sprint index N; propose `Sprint <N+1>` as default. Prompt the
   user to confirm or override.
2. **Read the modularity tree** at `docs/architecture/current/modularity/`.
3. **Empty tree** — print
   `no tree yet; run /modularize first at root to populate services from the arch vault, then re-run /plan-sprint`
   and exit gracefully.
4. **Show candidate scope:**
   - **Placeholder L1s** (dirs with no L2 subdirs) — label "expand first (run /modularize <slug>)".
   - **Placeholder L2s** (dirs with no L3 subdirs) — label "ready to pick — L3 decomposition
     happens later per feature".
   - **In-flight items from prior sprint** — read ROADMAP rows whose `Milestone` matches the last
     open tracker milestone; label "carry over?".
5. **DM picks items:**
   - **From tree** — DM picks L2 slugs (comma-separated, or one at a time). For any picked L2
     whose L1 parent is unexpanded, prompt "L1 <parent-slug> is unexpanded; run /modularize
     <parent-slug> now? [y/N]". If y, spawn `/modularize <parent-slug>` inline. If n, defer this
     L2 pick with a warning.
   - **Ad-hoc** — prompt "any items not in the tree?". For each: `title`, `priority`
     (P0/P1/P2), `level` (L0/L1/L2). `Module` column will be left blank for these.
6. **Create the tracker milestone** via the tracker adapter (read
   `.claude/tracker.json` → `tracker.platform`):
   - **Plane** — create a cycle named `<sprint-name>`.
   - **GitHub** — create a milestone in the docs repo named `<sprint-name>`.
   - **GitLab** — create a milestone at the project (or group, per team convention) named `<sprint-name>`.
7. **Append rows to `ROADMAP.md`.** For each picked item, append a row to the table with columns:

       | <feature-title> | <priority> | <level> | <l2-slug or blank> | | | <sprint-name> | |

   Column order matches the schema in `docs/superpowers/specs/2026-08-12-sprint-driven-roadmap-design.md` Part 3:
   `Feature | Priority | Level | Module | Spec | Issues | Milestone | Link`. `Spec`, `Issues`, `Link`
   start empty and are filled later by `/new-feature`, `/create-issues`, `/roadmap-sync`.
8. **Print next-steps summary** — one line per row that needs a spec:
   `next: /new-feature <slug>`.

### `--refine` mode

1. **Find the active sprint** — the tracker milestone that is open and has the highest sprint
   index. If none open, exit with `no active sprint to refine — run /plan-sprint (no --refine) to
   start a new one`.
2. **List its ROADMAP rows.**
3. **Prompt for an operation** — `add | drop <n> | done`. Loop until `done`.
   - **add** — same picking flow as normal mode steps 4–5; append rows to ROADMAP and create
     tracker milestone assignments.
   - **drop <n>** — remove the row from ROADMAP; move the tracker item off the milestone
     (unassign, do not delete the issue itself).

## Config

Reads `.claude/tracker.json` → `tracker.platform` and (for docs-repo git host, for the milestone
adapter branch on GitHub/GitLab) `gitHost.platform`. Writes to `ROADMAP.md` and to the tracker via
the appropriate MCP.
```

- [ ] **Step 2: Verify**

Read the file back. Confirm:
- Front-matter has `disable-model-invocation: true`.
- Normal-mode (8 steps) and `--refine`-mode (3 steps) are present.
- Tracker-adapter branches match sub-project (3) spec Part 4.

- [ ] **Step 3: Commit (optional)**

Message: `Add /plan-sprint skill for growing ROADMAP one sprint at a time`.

---

## Task 2 — Rework `/setup-project` Phase 2 (fully) and drop Phase 5

**Files:**
- Modify: `.claude/skills/setup-project/SKILL.md`

- [ ] **Step 1: Read the current Phase 2 section**

Read `.claude/skills/setup-project/SKILL.md` from `### Phase 2` to the start of `### Phase 3`. Sub-project (1) added an
arch-adopt-or-scaffold step to Phase 2; this task rewrites Phase 2 fully.

- [ ] **Step 2: Replace Phase 2 with the new flow**

Find the entire `### Phase 2 — Project context` section (from the heading through step 4 seed
ROADMAP as amended by sub-project 1).

Replace with:

```
### Phase 2 — Project context

1. Fill the `CLAUDE.md` project-context section: project name, stack, tracker line (`platform`, `org`,
   `projectId`), git-host line (`platform`, `org`), comms line (if configured). Leave the service-repo
   list for Phase 3.
2. **Import solution architecture (adopt-or-scaffold).** Prompt the user: "Do you have an existing
   solution-architecture set to import?" [y/n].
   - **y (adopt)** — accept a local dir path, a git URL to clone from, or an Obsidian vault export
     path. Copy contents into `docs/architecture/current/`. Verify a `README.md` exists at the root;
     if not, prompt the user to name (or promote) one before continuing.
   - **n (scaffold)** — write a starter `docs/architecture/current/README.md` with sections
     `## Context`, `## Components`, `## Data`, `## Deployment`, `## Non-functional requirements`
     (linking to `nfrs.md`), `## Assumptions` (linking to `assumptions.md`), `## Open questions`,
     each with a placeholder line. Halt Phase 2 until the user has filled at least one section
     with real (non-placeholder) content.
   - **Log the initial state** — create
     `docs/architecture/logs/YYYY-MM-DD-init/README.md` with `Decision: initial architecture recorded`,
     `Driver: project setup`, `Impact: all of current/`, `Alternatives rejected: none` (see the log
     format in `docs/superpowers/specs/2026-08-12-solution-architecture-home-design.md` Part 3).
3. **Ingest estimate — OPTIONAL.** Prompt: "Do you have an approved estimate to import?" [y/n]:
   - **y** — extract only NFRs and Assumptions from the estimate. Do **not** create ROADMAP feature
     rows and do **not** decompose the tree from the estimate.
   - **n** — skip; NFRs + Assumptions files stay as scaffolded placeholders.
4. **Seed the arch-vault NFRs + Assumptions files.**
   - Write `docs/architecture/current/nfrs.md`. Populated from step 3 if an estimate was ingested;
     otherwise a scaffolded skeleton with sections `## Availability`, `## Security`, `## Performance`,
     `## Compliance`, each with a placeholder line.
   - Write `docs/architecture/current/assumptions.md` with the same treatment (scaffolded skeleton
     if no estimate).
   - Ensure `current/README.md` includes Markdown links to both files under `## Non-functional
     requirements` and `## Assumptions` headings.
5. **Seed `ROADMAP.md` as a shell:**

       # ROADMAP

       The living sprint plan: one row per feature/task in a planned sprint. Rows are
       added by `/plan-sprint` at each sprint boundary — not seeded upfront.

       | Feature | Priority | Level | Module | Spec | Issues | Milestone | Link |
       |---------|----------|-------|--------|------|--------|-----------|------|

       Live status lives in the tracker; `/status` reads it.

   Empty table (no rows). Do not add NFRs / Assumptions sections here — they moved to the arch vault
   in step 4.

Phase 2 stops here. No modularization is run at setup. The DM/PL iterates on `current/` (with
paired arch-log entries) until the architecture is stable, then runs `/modularize` at root, then
`/plan-sprint`.
```

- [ ] **Step 3: Remove Phase 5 entirely**

Find the entire `### Phase 5 — Populate tracker (new project only; skip when fetched)` section
(from the heading through the last item under it). Delete this whole section including its heading.

- [ ] **Step 4: Renumber Phase 6 → 5 and Phase 7 → 6**

Find and update:
- `### Phase 6 — Propagate rules` → `### Phase 5 — Propagate rules`
- `### Phase 7 — Completeness check` → `### Phase 6 — Completeness check`

- [ ] **Step 5: Update the Phases prose in the Purpose block if any refers to "eight phases"**

Find in Purpose:

```
One-time init **orchestrator**. Runs the eight onboarding phases (0–7) in order, working the same for
greenfield (create everything) and brownfield (adopt what exists) projects, branching create-vs-fetch
per resource. Owns Phases 0, 1, 2, 4, 5 inline; delegates Phase 3 to `/setup-workspace`, Phase 6 to
`/propagate-rules`, and Phase 7 to `/check-setup`.
```

Replace with:

```
One-time init **orchestrator**. Runs the seven onboarding phases (0–6) in order, working the same
for greenfield (create everything) and brownfield (adopt what exists) projects, branching
create-vs-fetch per resource. Owns Phases 0, 1, 2, 4 inline; delegates Phase 3 to
`/setup-workspace`, Phase 5 to `/propagate-rules`, and Phase 6 to `/check-setup`.
```

- [ ] **Step 6: Read the whole file end-to-end**

Confirm:
- Phase numbering: 0 → 1 → 2 → 3 → 4 → 5 → 6 (seven phases, no gaps).
- Phase 2 has the 5 new steps from step 2 above.
- Phase 5 (was "Populate tracker") is gone.
- Purpose says "seven onboarding phases (0–6)".

- [ ] **Step 7: Commit (optional)**

Message: `Rework /setup-project Phase 2 flow and drop Phase 5`.

---

## Task 3 — Update `/check-setup`

**Files:**
- Modify: `.claude/skills/check-setup/SKILL.md`

- [ ] **Step 1: Update the ROADMAP check**

Find (as amended by sub-project 1):

```
8. **ROADMAP seeded** — `ROADMAP.md` has real feature rows (not just the header/example). Fix:
   `/setup-project` (Phase 2).
```

Replace with:

```
8. **ROADMAP shell present** — `ROADMAP.md` exists with the header and the empty rows table using
   the columns `Feature | Priority | Level | Module | Spec | Issues | Milestone | Link`. Rows may
   be empty; this is expected before the first `/plan-sprint`. Fix: `/setup-project` (Phase 2).
```

- [ ] **Step 2: Add a new check for NFRs + Assumptions files in the arch vault**

Find:

```
9. **Rules propagated** — each service repo has agent rules referencing `docs/conventions/`. Fix:
   `/propagate-rules`.
```

Insert immediately before it:

```
9. **Arch-vault NFRs + Assumptions present** — `docs/architecture/current/nfrs.md` and
   `docs/architecture/current/assumptions.md` exist and are linked from `current/README.md`. Fix:
   `/setup-project` (Phase 2).
```

Then renumber the following:
- `9. **Rules propagated**` → `10. **Rules propagated**`

- [ ] **Step 3: Update the Output summary**

Find:

```
A checklist, one line each `✅|⚠️|❌ <check> — <detail> [fix: <command>]`, then a one-line summary
`N/9 checks pass`.
```

Replace `N/9` with `N/10`.

- [ ] **Step 4: Read file end-to-end and verify**

Confirm:
- Checks numbered 1 → 10, no gaps.
- Check 8 is "ROADMAP shell present".
- Check 9 is "Arch-vault NFRs + Assumptions present".
- Check 10 is "Rules propagated".
- Output says `N/10 checks pass`.

- [ ] **Step 5: Commit (optional)**

Message: `Update /check-setup for sprint-driven ROADMAP + NFRs in arch vault`.

---

## Task 4 — Extend `/new-feature` with context reads

**Files:**
- Modify: `.claude/skills/new-feature/SKILL.md`

- [ ] **Step 1: Locate the brainstorming step**

Find:

```
2. **Run `superpowers:brainstorming`** to explore intent, requirements, and design for `<name>`.
   `docs/process/AGENTS.md` overrides Superpowers' own defaults: the resulting `spec.md` MUST contain,
   in order, the required sections — Business value / Why, Level (L0/L1/L2), Scope (in / out),
   Contracts, Acceptance criteria, Risks & assumptions.
```

- [ ] **Step 2: Insert context reads before the brainstorming step**

Insert immediately before step 2:

```
Before starting brainstorming, load these files into context (skip any that don't exist):

- `docs/architecture/current/nfrs.md` — architectural constraints the spec must respect.
- `docs/architecture/current/assumptions.md` — project-level assumptions.
- If a ROADMAP row exists whose `Feature` or `Module` column matches `<name>`, read the
  corresponding modularity node's `README.md` (path derived from the `Module` slug under
  `docs/architecture/current/modularity/`). Use its `Purpose` and `Scope (in / out)` sections as
  seed material for the spec's Business value and Scope.
```

Then renumber existing steps 2 → 3, 3 → 4, ..., 8 → 9.

- [ ] **Step 3: Read the file end-to-end**

Confirm the new step is inserted before brainstorming and all subsequent steps are renumbered.

- [ ] **Step 4: Commit (optional)**

Message: `Extend /new-feature with arch-vault + modularity context reads`.

---

## Task 5 — Update `ROADMAP.md` schema

**Files:**
- Modify: `ROADMAP.md`

- [ ] **Step 1: Rewrite the file**

Overwrite `ROADMAP.md` with:

```markdown
# ROADMAP

The living sprint plan: one row per feature/task in a planned sprint. Rows are added by
`/plan-sprint` at each sprint boundary — not seeded upfront.

| Feature | Priority | Level | Module | Spec | Issues | Milestone | Link |
|---------|----------|-------|--------|------|--------|-----------|------|

Live status is **not** duplicated here — it lives in the tracker. The `Link` column points to the
tracker item (epic/milestone); `/roadmap-sync` keeps `Issues`, `Milestone`, and `Link` current, and
`/status` reads progress live from the tracker.
```

NFRs and Assumptions sections that were previously at the bottom of ROADMAP are **removed** —
they've moved to `docs/architecture/current/nfrs.md` and `docs/architecture/current/assumptions.md`.

- [ ] **Step 2: Verify**

Read `ROADMAP.md` back. Confirm:
- Header + intro paragraph.
- Empty rows table with 8 columns.
- Closing paragraph about live status.
- No NFRs / Assumptions sections.

- [ ] **Step 3: Commit (optional)**

Message: `Update ROADMAP schema with Module column; remove NFRs/Assumptions (moved to arch vault)`.

---

## Task 6 — Update `docs/process/README.md`

**Files:**
- Modify: `docs/process/README.md`

- [ ] **Step 1: Update the phase table**

Find in `docs/process/README.md` `## Onboarding (one-time)` section:

```
| Phase | What it does |
|-------|--------------|
| 0 Access | choose tracker / git-host / comms; write `.env` + `.mcp.json`; probe each MCP server (must be green to proceed) |
| 1 Identity | pick a project name; rewrite placeholders; detach the template; create/adopt + push the docs repo |
| 2 Context | fill `CLAUDE.md`; import solution architecture (adopt-or-scaffold); ingest the estimate; seed `ROADMAP.md` |
| 3 Repos | `/setup-workspace` — create (template/exemplar, high-level config only) or clone + index existing |
| 4 Tracker | create or fetch the tracker project |
| 5 Populate | new project only — PL (or delegate) picks structure-only (default) / full backlog / project-only |
| 6 Rules | `/propagate-rules` — project-wide conventions stay here; repo/language-specific rules → each repo via PR |
| 7 Check | `/check-setup` — audit every phase artifact |
```

Replace with:

```
| Phase | What it does |
|-------|--------------|
| 0 Access | choose tracker / git-host / comms; write `.env` + `.mcp.json`; probe each MCP server (must be green to proceed) |
| 1 Identity | pick a project name; rewrite placeholders; detach the template; create/adopt + push the docs repo |
| 2 Context | fill `CLAUDE.md`; import solution architecture (adopt-or-scaffold); ingest estimate (optional; NFRs+Assumptions only); seed `ROADMAP.md` shell |
| 3 Repos | `/setup-workspace` — create (template/exemplar, high-level config only) or clone + index existing |
| 4 Tracker | create or fetch the tracker project |
| 5 Rules | `/propagate-rules` — project-wide conventions stay here; repo/language-specific rules → each repo via PR |
| 6 Check | `/check-setup` — audit every phase artifact |
```

- [ ] **Step 2: Update prose that refers to "eight phases"**

Find:

```
Before feature work, `/setup-project` runs an eight-phase init that works the same for greenfield
(create everything) and brownfield (adopt what exists), branching create-vs-fetch per resource. Every
phase is idempotent; a re-run resumes at the first incomplete phase.
```

Replace with:

```
Before feature work, `/setup-project` runs a seven-phase init that works the same for greenfield
(create everything) and brownfield (adopt what exists), branching create-vs-fetch per resource. Every
phase is idempotent; a re-run resumes at the first incomplete phase.
```

- [ ] **Step 3: Add a section describing sprint-driven ROADMAP growth**

Find `## Per-feature flow` heading. Insert a new section immediately before it:

```
## Sprint-driven ROADMAP growth

`ROADMAP.md` starts empty after setup and grows one sprint at a time. At sprint boundary the DM
runs `/plan-sprint`, which reads the modularity tree, shows candidate scope, asks the DM to pick
the sprint's contents (from tree L2 nodes and/or ad-hoc items), creates the tracker milestone, and
appends rows to `ROADMAP.md`. No feature rows are seeded at project setup and no tracker milestones
are pre-created.

See `.claude/skills/plan-sprint/SKILL.md` for the flow. ROADMAP rows may originate from the tree
(with a `Module` slug) or from planning discussion (blank `Module`) — the tree is a helper, not the
only source of ROADMAP entries.
```

- [ ] **Step 4: Verify**

Read the modified sections back. Confirm:
- Phase table: 7 rows (0 through 6).
- Prose says "seven-phase".
- New "Sprint-driven ROADMAP growth" section is present.

- [ ] **Step 5: Commit (optional)**

Message: `Update process README for 7-phase onboarding and sprint-driven ROADMAP`.

---

## Task 7 — Add one-way-direction rule and `/plan-sprint` mention to AGENTS.md

**Files:**
- Modify: `docs/process/AGENTS.md`

- [ ] **Step 1: Append the one-way-direction hard rule**

Find the last hard rule (added by sub-project 1's plan):

```
- **The solution-architecture home is mandatory.** From Phase 2 onward,
  `docs/architecture/current/README.md` must exist with non-placeholder content, and at least one
  entry must exist in `docs/architecture/logs/`. `/check-setup` reports the absence of either as a
  fail.
```

Append after it:

```
- **Docs → tracker / service repos is the only allowed direction of influence.** Skills that
  operate in the docs repo (`/modularize`, `/plan-sprint`, `/new-feature`, `/create-issues`) may
  read docs state and write to `docs/` and/or the tracker. Skills that respond to *implementation*
  concerns (`/split-issue`, service-repo automation) may read tracker/PR/service-repo state but
  MUST NOT write to `docs/`. The master plan (`docs/`) shapes execution; execution never mutates
  the master plan.
```

- [ ] **Step 2: Verify**

Read the Hard rules section back. Confirm the new rule appears as the last bullet.

- [ ] **Step 3: Commit (optional)**

Message: `Add docs → tracker one-way-direction hard rule to AGENTS.md`.

---

## Task 8 — Update `README.md`

**Files:**
- Modify: `README.md`

- [ ] **Step 1: Add `/plan-sprint` to the commands list**

Find:

```
- `/modularize [<slug>]` — decompose the solution architecture into a three-level tree of module
  nodes; runs on demand per node.
```

Insert immediately after:

```
- `/plan-sprint` — DM sprint-planning ceremony: pick items from the modularity tree + ad-hoc
  discussion, create the tracker milestone, append `ROADMAP.md` rows.
```

- [ ] **Step 2: Update the Quick start section prose**

Find:

```
1. `/setup-project` — Phase 0–7: configure + probe access, rename/detach this repo, fill context, seed
   `ROADMAP.md`, provision repos, create/fetch the tracker project, propagate rules, and audit the result.
```

Replace with:

```
1. `/setup-project` — Phase 0–6: configure + probe access, rename/detach this repo, fill context,
   import solution architecture, seed the `ROADMAP.md` shell, provision repos, create/fetch the
   tracker project, propagate rules, and audit the result.
```

- [ ] **Step 3: Update the Repo layout section**

Find:

```
docs/architecture/    this project's target solution architecture (current/) + change log (logs/)
```

Confirm it's already there from sub-project 1. If not, add it.

- [ ] **Step 4: Verify**

Read the updated `README.md` sections back. Confirm changes.

- [ ] **Step 5: Commit (optional)**

Message: `Update README for sprint-driven flow and /plan-sprint`.

---

## Self-review checklist

- **Spec coverage:**
  - Part 1 (phase restructure 8 → 7) — Task 2. ✅
  - Part 2 (new Phase 2 flow) — Task 2. ✅
  - Part 3 (ROADMAP schema — Module column) — Tasks 1 (write), 5 (schema file). ✅
  - Part 4 (`/plan-sprint` skill) — Task 1. ✅
  - Part 5 (ripples to `/new-feature`, `/create-issues`, `/roadmap-sync`) — Task 4 (`/new-feature`);
    `/create-issues` tweak is deferred to sub-project (4) plan; `/roadmap-sync` unchanged in shape
    (no task needed here). ✅
  - Part 6 (`/check-setup` updates) — Task 3. ✅
  - Part 7 (ripples to sub-projects 1, 2) — Task 2 adds NFRs/assumptions link in scaffold (ripple
    to sub-project 1). ✅
  - Part 8 (ripple summary) — Tasks 1–8 cover it. ✅
- **Placeholder scan:** none. Every step has exact text.
- **Type consistency:** `/plan-sprint`, `docs/architecture/current/nfrs.md`,
  `docs/architecture/current/assumptions.md`, `Module` column — consistent across tasks.

No gaps found. Plan is ready.
