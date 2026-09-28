# Unified Arch Log + Layer Discipline — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Drop `docs/<feature>/decisions.md` from the template and unify all persisted decisions
under `docs/architecture/logs/`. Add a layer-discipline rule + heuristic. Extend `/new-feature`
with a Shape-A `current/`-update step that drafts the paired arch-log entry when applicable.

**Architecture:** Docs-only. Six files touched. No new skills or agents. Sub-projects (1)–(4) are
merged and provide the arch-log infrastructure this plan reshapes the surrounding template around.

**Tech Stack:** Markdown.

**Commit policy:** Per standing instruction, do not commit the spec or plan. Implementation edits
may be committed.

**Spec:** `docs/superpowers/specs/2026-08-12-unified-arch-log-design.md`

---

## File Structure

- Modify: `.claude/skills/new-feature/SKILL.md` — drop decisions.md seed; add current/ update step;
  restructure commit list.
- Modify: `.claude/skills/setup-project/SKILL.md` — extend the Phase 2 scaffold with the
  layer-discipline callout.
- Modify: `.claude/agents/architecture-log-review.md` — append the layer-discipline check.
- Modify: `docs/process/AGENTS.md` — remove `decisions.md entry format` section, remove
  `decisions.md is the one exception` bullet, add layer-discipline hard rule.
- Modify: `docs/process/README.md` — remove the `decisions.md` sentence in "Where each gate happens".
- Modify: `README.md` — drop `decisions.md` from the repo-layout line.

---

## Task 1 — Extend `architecture-log-review` agent with layer-discipline check

**Files:**
- Modify: `.claude/agents/architecture-log-review.md`

- [ ] **Step 1: Read the current Checks section**

Read `.claude/agents/architecture-log-review.md` and locate the `## Checks` section, which
currently lists 5 numbered checks.

- [ ] **Step 2: Append check 6**

Find the last check:

    5. **Append-only invariant.** Are past log entries under `docs/architecture/logs/` untouched? Any
       edit or delete to existing log entries is a fail.

Append immediately after (still within the same numbered list):

    6. **Layer discipline.** Is the change described by `Impact` at business / service / contract /
       deployment-topology level? Changes limited to dependency versions, docker-compose or infra
       image bumps, CI/lint/test tooling, or code-level refactors are NOT architecture and belong in
       a spec or PR description instead. If the `Impact` list contains only implementation-detail
       items, emit FAIL. If it mixes architecture and implementation-detail, emit PASS_WITH_ISSUES
       and instruct the author to split into an arch entry + a separate change.

- [ ] **Step 3: Verify**

Read the Checks section back. Confirm 6 checks total, layer discipline is check 6.

---

## Task 2 — Add layer-discipline hard rule to `docs/process/AGENTS.md`; remove decisions.md sections

**Files:**
- Modify: `docs/process/AGENTS.md`

- [ ] **Step 1: Remove the entire `decisions.md entry format` section**

Find in `docs/process/AGENTS.md`:

    ## `decisions.md` entry format

    Each feature keeps `docs/<feature-name>/decisions.md` as a running, **reverse-chronological** decision
    log — newest entry first. It records the *why* behind choices as they land (spec authoring, spec
    review, implementation), distinct from the spec (the *what*) and the PR history (the review audit
    trail).

    Entry format:

    ```
    ### YYYY-MM-DD — <decision>

    **Decision**: <what was decided, one or two sentences>
    **Context**: <what prompted the decision — the situation or question>
    **Rationale**: <why this choice, over the alternatives>
    **Alternatives rejected**: <what else was considered and why it was not chosen>
    ```

    Append a new entry at the top of the file for every material decision. Do not edit or delete past
    entries — the log is additive.

Delete the entire section (heading + prose + code block).

- [ ] **Step 2: Remove the `decisions.md is the one exception` hard rule bullet**

Find in the `## Hard rules` section:

    - **`decisions.md` is the one exception, and it IS committed.** It is a decision log, not a review
      verdict — record decisions there as they are made, in every feature folder.

Delete this bullet.

- [ ] **Step 3: Append the layer-discipline hard rule at the end of the `## Hard rules` bullet list**

Find the last hard rule (the `docs → tracker` one-way-direction bullet added by sub-project 3):

    - **Docs → tracker / service repos is the only allowed direction of influence.** Skills that
      operate in the docs repo (`/modularize`, `/plan-sprint`, `/new-feature`, `/create-issues`) may
      read docs state and write to `docs/` and/or the tracker. Skills that respond to *implementation*
      concerns (`/split-issue`, service-repo automation) may read tracker/PR/service-repo state but
      MUST NOT write to `docs/`. The master plan (`docs/`) shapes execution; execution never mutates
      the master plan.

Append after it:

    - **Architecture layer discipline.** `docs/architecture/current/` describes the product at the
      business / service / contract level: business capabilities, services and their responsibilities,
      contracts between them (APIs, events, data ownership), and deployment topology at the
      boxes-and-arrows level. It does NOT describe implementation choices such as dependency
      versions, docker-compose image bumps, CI configuration, lint rules, framework upgrades that
      don't change any contract, code-level refactors, or small bug fixes. Rule of thumb: if the
      change would still matter to someone reading the product plan a year from now, it belongs in
      `current/`; otherwise it belongs in a spec, a PR description, or a code comment.
      `architecture-log-review` enforces this.

- [ ] **Step 4: Verify**

Read the whole `docs/process/AGENTS.md` end-to-end. Confirm:
- The `decisions.md entry format` section is gone.
- The `decisions.md is the one exception` bullet is gone.
- The layer-discipline bullet is the last item in the Hard rules list.
- No dangling references to `decisions.md` remain.

---

## Task 3 — Remove the `decisions.md` sentence from `docs/process/README.md`

**Files:**
- Modify: `docs/process/README.md`

- [ ] **Step 1: Locate the sentence in "Where each gate happens"**

Find:

    Nothing review-related is committed to the repo — PR comments and approvals are the audit trail, and
    they live in the PR history, not in a file. The one exception is `docs/<feature>/decisions.md`: it is
    a **decision log**, not a review verdict, and it is committed (see AGENTS.md for its format).

- [ ] **Step 2: Replace with a pointer to `docs/architecture/logs/`**

Replace the paragraph with:

    Nothing review-related is committed to the repo — PR comments and approvals are the audit trail, and
    they live in the PR history, not in a file. Architectural decisions that shape the product are the
    exception and DO live in the repo, under `docs/architecture/logs/YYYY-MM-DD-<codename>/README.md`,
    written alongside the `docs/architecture/current/` changes they document. See AGENTS.md for the
    log format and the `architecture-log-review` gate.

- [ ] **Step 3: Verify**

Read the "Where each gate happens" section back. Confirm the `decisions.md` sentence is gone and
the arch-log pointer replaces it.

---

## Task 4 — Drop `decisions.md` from `README.md` repo-layout

**Files:**
- Modify: `README.md`

- [ ] **Step 1: Update the repo-layout line for `docs/<feature>/`**

Find in the `## Repo layout` fenced block:

    docs/<feature>/        spec.md, decisions.md, optional plan.md — one folder per feature

Replace with:

    docs/<feature>/        spec.md, optional plan.md — one folder per feature

- [ ] **Step 2: Verify column alignment**

Confirm the description column still lines up with the surrounding rows (description starts at
column 23; the new content is shorter so alignment shifts left but internal structure is
preserved). If the visual alignment feels off compared to siblings, adjust trailing spaces.

- [ ] **Step 3: Verify**

Read the repo-layout block back.

---

## Task 5 — Add the layer-discipline callout to `/setup-project` Phase 2 scaffold

**Files:**
- Modify: `.claude/skills/setup-project/SKILL.md`

- [ ] **Step 1: Locate the scaffold in Phase 2 step 2**

Find in `.claude/skills/setup-project/SKILL.md` Phase 2 step 2 (the `n (scaffold)` sub-branch):

    - **n (scaffold)** — write a starter `docs/architecture/current/README.md` with sections
      `## Context`, `## Components`, `## Data`, `## Deployment`, `## Non-functional requirements`
      (linking to `nfrs.md`), `## Assumptions` (linking to `assumptions.md`), `## Open questions`,
      each with a placeholder line. Halt Phase 2 until the user has filled at least one section
      with real (non-placeholder) content.

- [ ] **Step 2: Extend the scaffold description to include the layer-discipline callout**

Replace with:

    - **n (scaffold)** — write a starter `docs/architecture/current/README.md`. First, insert this
      callout at the top of the file (before any section headings):

          > **Scope of this vault.** This describes the *shape of the product* — business
          > capabilities, services and their responsibilities, contracts (APIs / events / data
          > ownership), and deployment topology at the boxes-and-arrows level. It does not describe
          > implementation choices (dep versions, CI, lint, docker-compose image bumps, code-level
          > refactors). Rule of thumb: if a change matters a year from now to someone reading the
          > product plan, it belongs here; otherwise it belongs in a spec or a PR description.

      Then add sections `## Context`, `## Components`, `## Data`, `## Deployment`,
      `## Non-functional requirements` (linking to `nfrs.md`), `## Assumptions` (linking to
      `assumptions.md`), `## Open questions`, each with a placeholder line. Halt Phase 2 until the
      user has filled at least one section with real (non-placeholder) content.

Note: on the `y (adopt)` branch, the user's existing vault is copied in as-is; the callout is
NOT injected on adopt.

- [ ] **Step 3: Verify**

Read Phase 2 step 2 back. Confirm the callout is only on the scaffold branch.

---

## Task 6 — Rework `/new-feature` (drop decisions.md seed, add current/ update step)

**Files:**
- Modify: `.claude/skills/new-feature/SKILL.md`

- [ ] **Step 1: Update the front-matter `description`**

Find at the top of `.claude/skills/new-feature/SKILL.md`:

    description: Orchestrate brainstorming into docs/<name>/spec.md, seed decisions.md, propose a level, and open the spec PR

Replace with:

    description: Orchestrate brainstorming into docs/<name>/spec.md, propose a level, ask whether the feature changes docs/architecture/current/ and (if yes) write the paired arch-log entry, then open the spec PR

- [ ] **Step 2: Update the Purpose paragraph**

Find:

    Turn an idea into a reviewable feature spec. Orchestrates `superpowers:brainstorming` to produce
    `docs/<name>/spec.md`, seeds `docs/<name>/decisions.md`, proposes the L0/L1/L2 level, and optionally
    runs `superpowers:writing-plans` for a `plan.md`. Ends by opening the spec PR that `/spec-review` (and,
    for L2, `/arch-review` + `/contract-data-event-review`) will comment on.

Replace with:

    Turn an idea into a reviewable feature spec. Orchestrates `superpowers:brainstorming` to produce
    `docs/<name>/spec.md`, proposes the L0/L1/L2 level, asks whether the feature adds or changes
    anything in `docs/architecture/current/` and — if yes — writes the paired arch-log entry in the
    same PR. Optionally runs `superpowers:writing-plans` for a `plan.md`. Ends by opening the spec
    PR that `/spec-review` (and, for L2, `/arch-review` + `/contract-data-event-review`) will
    comment on.

- [ ] **Step 3: Remove the `decisions.md` seed step**

Find:

    5. **Seed `docs/<name>/decisions.md`** with one entry (today's date, "spec drafted") using the format
       in `docs/process/AGENTS.md` (`### YYYY-MM-DD — <decision>` / Decision / Context / Rationale /
       Alternatives rejected).

Delete this entire step. Renumber the subsequent steps (6→5, 7→6, 8→7).

- [ ] **Step 4: Insert the "Propose current/ update" step as new step 5**

After the (already renumbered) step 4 "Write `docs/<name>/spec.md`" and before the renumbered
"Propose the level" step, insert a new step 5:

    5. **Propose `current/` update.** Ask the user: "Does this feature add or change anything in
       `docs/architecture/current/`?" Use the matches from step 2's context-load as reference
       ("here is what already exists related to this feature").
       - **If yes** — mini Q&A: which files or pages under `current/` to add or modify, what
         content. Write the diff. Draft the paired arch-log entry at
         `docs/architecture/logs/YYYY-MM-DD-<name>/README.md` with `Driver: new feature: <name>`,
         `Decision: introduce <name> into product shape`, `Rationale` captured from the Q&A,
         `Alternatives rejected` (list dropped proposals), `Impact` (list of `current/` files
         touched), `Links` (`docs/<name>/spec.md`).
       - **If no** — skip. No `current/` write, no arch-log entry. Spec references existing
         `current/` content instead.

Renumber subsequent steps accordingly (the pre-existing "Propose the level" becomes step 6, etc.).

- [ ] **Step 5: Update the `Open the spec PR` step's commit list**

Find (was step 7, now step 8 after Task 6 Steps 3–4 renumbering):

    8. **Open the spec PR**: create a branch, commit `docs/<name>/{spec.md,decisions.md[,plan.md]}`, push,
       and open a PR against this repo's default branch titled after `<name>`. This is what
       `/spec-review` and the L2 review commands will comment on.

Replace the commit-list clause:

    8. **Open the spec PR**: create a branch, commit `docs/<name>/{spec.md[,plan.md]}` and (if step 5
       wrote to them) any changed files under `docs/architecture/current/` plus the new
       `docs/architecture/logs/YYYY-MM-DD-<name>/README.md`, push, and open a PR against this repo's
       default branch titled after `<name>`. This is what `/spec-review` and the L2 review commands
       will comment on.

- [ ] **Step 6: Read the whole file end-to-end**

Confirm:
- Numbering runs 1 → 9 with no gaps.
- Step 5 is "Propose current/ update".
- No `decisions.md` references remain anywhere in the file.
- Front-matter description matches the new intent.

- [ ] **Step 7: Grep for stray decisions.md references across the six touched files**

Run:

    grep -n "decisions.md" .claude/skills/new-feature/SKILL.md .claude/skills/setup-project/SKILL.md .claude/agents/architecture-log-review.md docs/process/AGENTS.md docs/process/README.md README.md

Expected: no matches. If any remain, fix them inline.

---

## Task 7 — Single commit

**Files:**
- Modify: all six files touched above.

- [ ] **Step 1: Sanity check the diff**

Run `git diff --stat` and confirm the six expected files are modified (no extras, no untracked
files staged).

- [ ] **Step 2: Stage and commit**

Stage explicitly (do NOT `git add -A` — that would sweep in spec/plan files under
`docs/superpowers/`):

    git add \
      .claude/skills/new-feature/SKILL.md \
      .claude/skills/setup-project/SKILL.md \
      .claude/agents/architecture-log-review.md \
      docs/process/AGENTS.md \
      docs/process/README.md \
      README.md

Commit with message (heredoc):

    Implement sub-project (5): unified arch log + layer discipline

    Drop docs/<feature>/decisions.md from the template. All decisions
    worth persisting land in docs/architecture/logs/. Feature-level
    decisions belonged there all along — the arch log covers the same
    ground with a stronger enforcement (arch-log-review) and a clearer
    scope.

    Add a layer-discipline hard rule to AGENTS.md and a matching check
    to the architecture-log-review agent: docs/architecture/current/
    describes business capabilities, services, contracts, deployment
    topology — not dep-version bumps, CI, lint, docker-compose image
    tweaks, or code-level refactors. Rule of thumb: if it matters a
    year from now to someone reading the product plan, it belongs in
    current/.

    Extend /new-feature: after writing the spec, ask whether the
    feature adds or changes anything in current/. If yes, write the
    diff + draft the paired arch-log entry in the same PR. If no,
    only the spec is written. Removes the decisions.md seeding step.

    Extend /setup-project Phase 2: on the scaffold branch, inject a
    Scope-of-this-vault callout at the top of the starter
    current/README.md so vault owners see the layer-discipline
    heuristic every time they open the entry point.

    Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>

- [ ] **Step 3: Verify the commit**

Run `git show --stat HEAD` and confirm exactly six files changed.

---

## Self-review checklist

- **Spec coverage:**
  - Part 1 (drop decisions.md) — Tasks 2, 3, 4, 6. ✅
  - Part 2 (layer discipline rule + heuristic + agent check) — Tasks 1, 2, 5. ✅
  - Part 3 (`/new-feature` Shape A rework) — Task 6. ✅
  - Part 4 (`/check-setup` — no change) — no task; explicitly documented as no-op. ✅
  - Part 5 (ripples) — Tasks 1–6 cover all six files. ✅
  - Part 6 (migration guidance) — informative only, no task needed. ✅
- **Placeholder scan:** none. Every step lists the exact text to change and exact replacement.
- **Type consistency:** `docs/architecture/logs/`, `docs/architecture/current/`, the log filename
  convention `YYYY-MM-DD-<codename>`, and the `<name>` slug used by `/new-feature` are consistent
  across every task.

No gaps found. Plan is ready.
