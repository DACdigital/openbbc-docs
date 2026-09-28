# Reactive Issue Splitting — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add the `/split-issue <issue-id> [--pr <pr-url>]` skill that decomposes a tracker issue
into sub-issues (tracker-only, no docs writes), and add the `/create-issues` tweak that offers to
create matching sub-issues from L3 tree nodes when they exist for the feature's L2.

**Architecture:** Docs-only. Adds `/split-issue`; extends `/create-issues`; mentions both in
process/README.md.

**Tech Stack:** Markdown, YAML front-matter.

**Commit policy:** Per standing instruction, do not commit the spec or plan. Implementation edits
may be committed.

**Spec:** `docs/superpowers/specs/2026-08-12-reactive-issue-splitting-design.md`

**Prerequisites:** sub-projects (1), (2), (3) plans complete.

**Design principle enforced by this plan:** Docs → tracker one-way direction. `/split-issue` never
writes to `docs/`. Any promote-to-tree operation would violate this; the spec removed that option
explicitly. This plan MUST NOT reintroduce it.

---

## File Structure

- **Create**:
  - `.claude/skills/split-issue/SKILL.md`
- **Modify**:
  - `.claude/skills/create-issues/SKILL.md` — small tweak to offer L3-derived sub-issues.
  - `docs/process/README.md` — mention `/split-issue` and extend the tracker-adapter table.

---

## Task 1 — Create `/split-issue` skill

**Files:**
- Create: `.claude/skills/split-issue/SKILL.md`

- [ ] **Step 1: Write the skill file**

Create `.claude/skills/split-issue/SKILL.md` with this content:

```markdown
---
name: split-issue
description: Reactive tracker-only issue splitting — decomposes a tracker issue into sub-issues on demand (dev sees their PR is too big; reviewer wants smaller diffs). Reads issue description + optional PR link for scope grounding, auto-proposes sub-issues, refines via Q&A, creates them in the tracker under the parent. Never writes to docs/.
disable-model-invocation: true
---

# /split-issue `<issue-id> [--pr <pr-url>] [--refine]`

## Purpose

Split a tracker issue into sub-issues when its scope is too large for a reviewable PR. Triggered
by anyone (dev mid-implementation, reviewer during review, PL). **Tracker-only**: never writes to
`docs/`, never touches the modularity tree, never creates arch-log entries. This respects the
`docs → tracker` one-way direction rule in `docs/process/AGENTS.md`.

For **proactive** planning-time decomposition (create L3 tree nodes for a feature *before*
starting), use `/modularize <l2-slug>` instead — that is the docs-side tool that writes to the
tree.

## Inputs

- `<issue-id>` (required) — tracker-native id. Plane work-item id / GitHub `#NN` (or full URL) /
  GitLab `#NN` (or full URL).
- `--pr <pr-url>` (optional) — link to a draft/open PR. If provided, the skill reads the PR's
  file-list and summary via git-host MCP as extra scope grounding.
- `--refine` (optional flag) — operate on the issue's existing sub-issues (add / rename / drop)
  instead of creating fresh ones.

## Steps

### Normal mode (no `--refine`)

1. **Resolve tracker platform** from `.claude/tracker.json` → `tracker.platform`.
2. **Fetch the issue** via the tracker adapter — title, description, existing sub-issues (if any).
3. **Refuse-with-hint on existing sub-issues.** If the issue already has sub-issues, print
   `use --refine to modify existing sub-issues` and exit.
4. **Fetch PR context** — if `--pr` was passed, fetch the PR's file list and summary via git-host
   MCP. On failure, warn and continue without PR grounding.
5. **Auto-propose N sub-issues.** Based on issue description + optional PR context, propose N
   sub-issues with `{title, summary}`. Each should be sized for a small-diff PR.
6. **Q&A refine, one at a time.** For each proposal, prompt keep / edit / drop. Edits prompt only
   for `title`, `summary`.
7. **Add-more loop.** After the last proposal, prompt "any missing?". If yes, mini Q&A per new
   sub-issue.
8. **Confirm parent.** Default parent is `<issue-id>`. User can override to a different tracker
   issue as parent.
9. **Materialize sub-issues.** One tracker call per sub-issue via the adapter:
   - **Plane** — create a work item with `parent = <resolved-parent>`, using the parent's
     work-item type.
   - **GitHub** — create an issue in the same repo, attach as sub-issue via the `/sub_issues` API,
     inherit milestone from parent when set.
   - **GitLab** — create an issue in the same project, link to parent via the linked-issue
     relation (`relates_to`), inherit milestone from parent when set.

   Each sub-issue's description contains a link back to the parent issue and the Q&A summary.
10. **Print summary** — list of created sub-issue ids + URLs.

### `--refine` mode

1. **Resolve platform + fetch the parent issue and its existing sub-issues** as in normal mode
   steps 1–2.
2. **List existing sub-issues** as a numbered menu.
3. **Prompt for an operation** — `add | rename <n> | drop <n> | done`. Loop until `done`.
   - **add** — same mini-Q&A as normal mode step 7; create in tracker under the parent.
   - **rename <n>** — prompt for new title/summary; update via tracker adapter. Never touches
     `docs/`.
   - **drop <n>** — close the sub-issue in the tracker with reason `dropped during /split-issue
     --refine`. Do not delete the tracker item — closure preserves audit history.

## Config

Reads `.claude/tracker.json` → `tracker.platform` (for the tracker adapter) and `gitHost.platform`
(for the git-host MCP used when `--pr` is passed). **Writes only to the tracker and to nothing
under `docs/`.** Enforcing this is the design principle from
`docs/superpowers/specs/2026-08-12-reactive-issue-splitting-design.md` — do not reintroduce any
promote-to-tree flow.
```

- [ ] **Step 2: Verify**

Read the file back. Confirm:
- Front-matter has `disable-model-invocation: true`.
- Purpose explicitly states "tracker-only" and "never writes to `docs/`".
- Normal mode (10 steps) and `--refine` mode (3 steps) are present.
- Tracker adapter branches for Plane / GitHub / GitLab in step 9.
- Config section reinforces the docs-write ban.

- [ ] **Step 3: Commit (optional)**

Message: `Add /split-issue skill for reactive tracker-only issue decomposition`.

---

## Task 2 — Extend `/create-issues` with L3-derived sub-issues (opt-in)

**Files:**
- Modify: `.claude/skills/create-issues/SKILL.md`

- [ ] **Step 1: Read `/create-issues` current flow**

Read `.claude/skills/create-issues/SKILL.md` to understand its current structure. The change adds
one optional post-step: after creating the parent feature issue, check the modularity tree for L3
children and offer to create sub-issues from them.

- [ ] **Step 2: Insert a "Sub-issues from L3 tree (opt-in)" step**

Find the last step of the current create-issues flow (the one that finalizes issue creation and
prints the summary). Insert immediately before it:

```
N. **Offer L3-derived sub-issues (opt-in).** After the parent feature issue is created, look up
   whether L3 tree nodes exist for this feature:
   - Read the ROADMAP row for this feature (match by `Feature` column).
   - If the row's `Module` column is blank, skip this step — nothing to check; print the hint
     `next: run /split-issue <parent-id> later if the scope needs splitting` and continue.
   - If the `Module` slug is set, look for L3 subdirs under
     `docs/architecture/current/modularity/<...>/<module-slug>/` (walk the modularity dir to find
     the L2 node whose slug matches).
   - **No L3 subdirs found** — print the same `/split-issue` hint; continue.
   - **L3 subdirs found** — prompt `L3 nodes found under <l2-slug>. Create matching sub-issues
     from them? [y/N]`. Default N.
     - **y** — for each L3 subdir, create one sub-issue in the tracker with:
       - `parent = <parent-feature-issue-id>`,
       - `title` from the L3 node's `title` front-matter field,
       - `description` from the L3 node's Purpose section body plus a link back to the parent
         issue.
       Use the same tracker adapter branches as `/split-issue` (Plane parent, GitHub `/sub_issues`,
       GitLab linked-issue).
     - **n** — skip; L3 tree nodes stay as planning-only artifacts. Print the `/split-issue` hint.

This step is docs → tracker only; it reads from `docs/` (the tree) and writes to the tracker.
Nothing in `docs/` is modified.
```

Replace the leading `N.` with the actual next step number based on the file's current numbering.

- [ ] **Step 3: Verify**

Read the file back. Confirm:
- The new step is inserted before the summary/finalization step.
- The step's guard on `Module` column being blank is present.
- The step never writes to `docs/`.

- [ ] **Step 4: Commit (optional)**

Message: `Extend /create-issues to offer L3-derived sub-issues opt-in`.

---

## Task 3 — Mention `/split-issue` in process README + adapter table

**Files:**
- Modify: `docs/process/README.md`

- [ ] **Step 1: Update the Per-feature flow diagram**

Find:

```
select feature (from ROADMAP)
  → /new-feature  → docs/<feature>/spec.md  → spec PR
  → /spec-review  → verdict on PR
  → [L2] /arch-review + /contract-data-event-review → verdicts on same PR
  → human approves + merges spec PR         ← GATE
  → /create-issues → child issues in service repos (pluggable tracker)
  → /roadmap-sync  → ROADMAP links spec + issues + tracker; manages tracker milestones/epics
  → Devs implement in workspace/<repo>, code PRs reviewed & merged
  → all child issues close → feature done → ROADMAP updated
```

Replace with:

```
select feature (from ROADMAP)
  → /new-feature  → docs/<feature>/spec.md  → spec PR
  → /spec-review  → verdict on PR
  → [L2] /arch-review + /contract-data-event-review → verdicts on same PR
  → human approves + merges spec PR         ← GATE
  → /create-issues → parent issue in tracker; opt-in sub-issues from L3 tree if present
  → /roadmap-sync  → ROADMAP links spec + issues + tracker; manages tracker milestones/epics
  → Devs implement in workspace/<repo>, code PRs reviewed & merged
     ↳ /split-issue <id> if a dev/reviewer wants smaller PRs mid-flight (tracker-only)
  → all child issues close → feature done → ROADMAP updated
```

- [ ] **Step 2: Extend the tracker-adapter table with sub-issue mechanics**

Find the tracker-adapter table:

```
| Concept | Plane | GitHub | GitLab |
|---------|-------|--------|--------|
| Ticket | work item | issue | issue |
| Grouping by timebox | cycle (≈ milestone) | milestone | milestone |
| Grouping by theme | epic | milestone + sub-issues, or a tracking issue | epic |
| Sub-ticket | work item with parent | sub-issue | issue linked to epic |
```

Replace the last row with a fuller sub-issue row:

```
| Sub-ticket | work item with `parent` | sub-issue attached via `/sub_issues` API | issue linked via `relates_to` |
```

- [ ] **Step 3: Verify**

Read the modified sections back. Confirm the flow diagram includes the `/split-issue` line and the
adapter table's sub-ticket row uses the API-level verbs.

- [ ] **Step 4: Commit (optional)**

Message: `Document /split-issue and sub-issue mechanics in process README`.

---

## Self-review checklist

- **Spec coverage:**
  - Part 1 (two decomposition flows) — Task 1 covers reactive; proactive already lives in
    sub-project (2)'s modularize skill. ✅
  - Part 2 (`/split-issue` skill flow) — Task 1. ✅
  - Part 3 (`--refine` mode) — Task 1, `--refine` mode section. ✅
  - Part 4 (tracker adapter branches) — Task 1, step 9 + Task 3, step 2. ✅
  - Part 5 (`/create-issues` tweak) — Task 2. ✅
  - Part 6 (ripples) — Tasks 1, 2, 3. ✅
  - Part 7 (ripples to sub-projects 1/2/3) — none needed in this plan; sub-project (3) already
    added the ROADMAP `Module` column which Task 2 reads. ✅
  - Part 8 (non-goals) — the plan does NOT create any promote-to-tree flow, honoring the design
    principle. ✅
- **Placeholder scan:** none.
- **Type consistency:** `/split-issue`, `parent`, `sub-issue` terminology, tracker platform
  branch names — consistent across tasks.

No gaps found. Plan is ready.
