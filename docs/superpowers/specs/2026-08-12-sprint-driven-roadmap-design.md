# Sprint-Driven ROADMAP + Phase 2 Rework

**Date:** 2026-08-12
**Status:** Approved design, pre-implementation
**Depends on:** sub-projects (1) and (2) of this initiative — the solution-architecture home
(`docs/superpowers/specs/2026-08-12-solution-architecture-home-design.md`) and the modularity skill
(`docs/superpowers/specs/2026-08-12-modularity-skill-design.md`). This is sub-project (3) of five.

## Purpose

Today `/setup-project` seeds `ROADMAP.md` at Phase 2 with feature rows from an approved estimate,
and Phase 5 populates the tracker with epics/milestones mirroring the seed. The result is a
plan-everything-upfront pattern: teams commit to a full backlog before the architecture is stable
or the tree is decomposed.

This sub-project inverts it. Architecture is the only mandatory intake at setup. Estimate becomes
optional. ROADMAP.md is seeded as a shell — no feature rows — and **grows one sprint at a time** via
a new `/plan-sprint` command. Modularization runs on user demand, not at setup. The result: the
project's plan grows as understanding grows.

## Scope

- **In:** phase restructure (8 → 7 phases); new Phase 2 flow (arch mandatory, estimate optional,
  no upfront ROADMAP rows, no setup-time modularization); Phase 5 removal; the `/plan-sprint` skill
  (initial + `--refine`); ROADMAP.md schema addition (`Module` column); relocation of NFRs +
  Assumptions to the arch vault; ripples to `/new-feature`, `/create-issues`, `/roadmap-sync`,
  `/check-setup`; the small ripple to sub-project (1)'s scaffold.
- **Out (YAGNI):** any per-sprint metadata inside the modularity tree; auto-scheduling from tree
  metadata; auto-decomposition at Phase 2; any escape hatch to populate a full backlog from an
  estimate (Q4 chose to drop this cleanly); sprint retros / velocity / burn-down; concurrent
  active sprints (support one at a time — parallel sprints work via naming, no special mechanics).

## Writing principle

Concise, technical, complete. The process docs remain the source of truth; `.claude/` skills are
the execution adapter.

## Part 1 — Phase restructure

Onboarding goes from 8 phases (0–7) to **7 phases (0–6)**. Phase 5 (Populate tracker) is dropped
entirely; Phase 6 (Rules) and Phase 7 (Check) renumber to 5 and 6.

| Was | New | Content |
|-----|-----|---------|
| 0 Access | 0 Access | unchanged |
| 1 Identity | 1 Identity | unchanged |
| 2 Context | **2 Context** | **REWORKED** — see Part 2 |
| 3 Repos | 3 Repos | unchanged (starter-packs discovery from prior work stays) |
| 4 Tracker | 4 Tracker | unchanged — create/fetch tracker project only |
| ~~5 Populate~~ | — | **DROPPED** — milestones/issues no longer created at setup |
| 6 Rules | 5 Rules | `/propagate-rules` — content unchanged, just renumbered |
| 7 Check | 6 Check | `/check-setup` — content updates in Part 6 |

## Part 2 — New Phase 2 flow

`.claude/skills/setup-project/SKILL.md` Phase 2 is rewritten to these steps:

1. **Fill CLAUDE.md project-context.** Unchanged from today (project name, stack, tracker line,
   git-host line, comms line if configured; service repos deferred to Phase 3).
2. **Import solution architecture** — invoke sub-project (1)'s adopt-or-scaffold flow. Result:
   `docs/architecture/current/README.md` exists with non-placeholder content; the initial arch-log
   entry (`docs/architecture/logs/YYYY-MM-DD-init/`) is written.
3. **Ingest estimate — OPTIONAL.** Prompt: "Do you have an approved estimate to import?" [y/n]:
   - **y** — extract only NFRs + Assumptions. Do **not** create ROADMAP feature rows and do **not**
     decompose the tree from the estimate. Any estimate feature-list content is preserved only
     inside the estimate export itself (the ingest is purely extractive for NFRs + Assumptions).
   - **n** — skip; NFRs + Assumptions files stay as scaffolded placeholders.
4. **Seed the arch-vault NFRs + Assumptions files:**
   - Write `docs/architecture/current/nfrs.md`. Populated from step 3 if an estimate was ingested;
     otherwise a scaffolded skeleton with sections such as `## Availability`, `## Security`,
     `## Performance`, `## Compliance`, each with a placeholder line.
   - Write `docs/architecture/current/assumptions.md` with the same treatment.
   - Update `current/README.md` to include Markdown links to both files under a `## Non-functional
     requirements` and `## Assumptions` heading. This satisfies the ripple to sub-project (1)
     without changing (1)'s core rules.
5. **Seed ROADMAP.md as a shell:**

       # ROADMAP

       The living sprint plan: one row per feature/task in a planned sprint. Rows are
       added by `/plan-sprint` at each sprint boundary — not seeded upfront.

       | Feature | Priority | Level | Module | Spec | Issues | Milestone | Link |
       |---------|----------|-------|--------|------|--------|-----------|------|

       Live status lives in the tracker; `/status` reads it.

   Empty table (no rows). NFRs and Assumptions sections are **removed** from ROADMAP.md (they moved
   in step 4).

Phase 2 stops here. **No modularization is run at setup.** DM/PL iterate on `current/` (each edit
paired with an arch-log entry via `arch-log-review`) and run `/modularize` at root only when they
consider the architecture stable enough to decompose.

## Part 3 — ROADMAP.md schema

One new column added: **`Module`** — the modularity-tree slug this row derives from, when
applicable. Blank when the row is an ad-hoc item from planning discussion (bug, tech-debt, ops task
not in the tree, client ask that doesn't fit cleanly). All other columns unchanged from today.

Column order: `Feature | Priority | Level | Module | Spec | Issues | Milestone | Link`.

## Part 4 — `/plan-sprint` skill

New skill at `.claude/skills/plan-sprint/SKILL.md`. `disable-model-invocation: true` (user-invoked
only). Owned by the DM role.

**Flow:**

1. **Determine sprint name.** Compute the next sprint name (`Sprint <N+1>` by default, where N is
   the highest sprint index seen in existing tracker milestones and ROADMAP rows). Prompt the user
   to confirm or override.
2. **Read the modularity tree** at `docs/architecture/current/modularity/`.
3. **Empty tree** — warn `no tree yet; run /modularize first at root to populate services from the
   arch vault, then re-run /plan-sprint`, and exit gracefully with no side effects.
4. **Non-empty tree — show candidate scope:**
   - Placeholder L1 nodes (no L2 subdirs): "expand-first" candidates.
   - Placeholder L2 nodes (no L3 subdirs): "ready to pick — L3 decomposition happens later per
     feature".
   - In-flight items from the prior sprint (ROADMAP rows whose `Milestone` isn't yet closed on the
     tracker): shown so DM decides whether to carry them over.
5. **DM picks items.** Two picking flows:
   - **From tree** — DM selects L2 slugs. For any picked L2 whose L1 parent is still unexpanded, the
     skill **offers** to run `/modularize <l1-slug>` inline (spawns the modularity skill). If DM
     declines, the L2 pick is deferred with a warning; the sprint proceeds without that item.
   - **Ad-hoc** — for items not in the tree, prompt: `title`, `priority` (P0/P1/P2), `level`
     (L0/L1/L2). `Module` column left blank for these.
6. **Create the tracker milestone** for this sprint via the tracker adapter. Sprint name maps to
   milestone name. Adapter branches on `.claude/tracker.json` → `tracker.platform`:
   - **Plane** — create a cycle whose name is the sprint name.
   - **GitHub** — create a milestone in the docs repo.
   - **GitLab** — create a milestone at the project or group level per convention.
7. **Append ROADMAP.md rows** for every picked item. `Module` filled from tree slug when picked
   from tree; blank for ad-hoc. `Milestone` set to the sprint name. `Spec`, `Issues`, `Link` start
   empty; they're populated later by `/new-feature`, `/create-issues`, and `/roadmap-sync`.
8. **Print next-steps summary** — one line per row that needs a spec:
   `next: /new-feature <slug>`.

**`--refine` mode.** `/plan-sprint --refine` re-runs on the currently active sprint (the one whose
milestone is open on the tracker) to add / drop items mid-flight:

- **Add** — same picking flow as steps 4–5; new rows appended; milestone assignment created.
- **Drop** — remove a row from ROADMAP; move the tracker item off the milestone (do not delete the
  issue itself if one exists — that stays for audit).

**No sprint close command in this spec** — closing a sprint is a tracker-side action (close the
milestone). `/roadmap-sync` reflects the closure into ROADMAP. If teams want a `/close-sprint`
helper later, it can be added; punted for YAGNI.

## Part 5 — Ripples to existing commands

- **`/new-feature <slug>`.** No shape change. Two small additions:
  - If a ROADMAP row exists whose `Feature` or `Module` matches `<slug>`, the skill reads that
    modularity node's `README.md` (via `Module` slug) as extra context for the spec's Purpose
    section.
  - Reads `docs/architecture/current/nfrs.md` and `assumptions.md` and passes their content to
    brainstorming as extra context, so the spec author sees the architectural constraints.
- **`/create-issues`.** Unchanged in shape. Runs per-feature after each spec merges. Reads the
  ROADMAP row's `Milestone` to know which sprint the issues belong to.
- **`/roadmap-sync`.** Unchanged in shape. Still keeps `Issues`, `Milestone`, `Link` columns
  current from the tracker. Operates on a smaller, sprint-scoped ROADMAP now.
- **`/check-setup`.** Content updates (see Part 6).
- **`/setup-project`.** Phase 2 rewrite (Part 2); Phase 5 removal; Phase 6 → 5, Phase 7 → 6.
- **`/status`.** Unchanged.
- **`/spec-review`, `/arch-review`, `/contract-data-event-review`.** Unchanged.

## Part 6 — `/check-setup` updates

Modify `.claude/skills/check-setup/SKILL.md`:

- **Remove** the check "ROADMAP seeded — has real feature rows".
- **Replace with** "ROADMAP shell present" — `ROADMAP.md` exists with a header and the empty rows
  table (rows may be empty; header and column set must match Part 3).
- **Add** "Arch-vault NFRs + Assumptions present" — `docs/architecture/current/nfrs.md` and
  `assumptions.md` exist and are linked from `current/README.md`.
- Existing arch checks from sub-project (1) already fire (arch home present, initial log entry
  present).

Total check count goes from 7 (today) → 9 after sub-project (1) → **10 after this sub-project**.

## Part 7 — Ripples to other sub-projects

- **Sub-project (1) — Solution-architecture home.** Small ripple: the scaffolded starter
  `current/README.md` now suggests two additional files (`nfrs.md`, `assumptions.md`) with sections
  and links. Sub-project (1)'s arch-log rule is unchanged and still covers changes to these files.
  When implementing (1)+(3) together, fold this into (1)'s implementation plan.
- **Sub-project (2) — Modularity skill.** No ripple. `/modularize` runs on user demand as
  specified; it doesn't know about sprints or ROADMAP.
- **Sub-project (4) — Sub-issue sharding (not yet designed).** Consumes L3 nodes per feature; runs
  post-spec-merge alongside `/create-issues`. Sprint-driven ROADMAP doesn't block it.

## Part 8 — Ripple summary (files this spec's implementation will touch)

- **Add:** `.claude/skills/plan-sprint/SKILL.md`.
- **Modify:** `.claude/skills/setup-project/SKILL.md` (Phase 2 rewrite, Phase 5 removal, renumber);
  `.claude/skills/check-setup/SKILL.md` (Part 6 updates); `.claude/skills/new-feature/SKILL.md`
  (Part 5 additions); `docs/process/README.md` (phase table, per-feature flow mention, tracker
  adapter table cross-reference); `docs/process/AGENTS.md` (mention `/plan-sprint` alongside
  `/new-feature` in the hard-rules preamble if useful — else no change); `README.md` (repo layout
  and Quick-start section — mention the new command and the new flow).

## Open questions

None blocking implementation. Two low-priority follow-ups:

- Whether `/plan-sprint` should also create the tracker's parent epic when the sprint contains
  multiple related items. Punt — current adapters and `/create-issues` handle grouping fine.
- Whether ROADMAP should render a "sprint separator" row between milestones for readability. Punt
  — the `Milestone` column already groups visually via sort.
