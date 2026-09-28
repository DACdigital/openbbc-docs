# Reactive Issue Splitting (`/split-issue`)

**Date:** 2026-08-12
**Status:** Approved design, pre-implementation
**Depends on:** sub-projects (1), (2), and (3) of this initiative — the solution-architecture home,
the modularity skill, and the sprint-driven ROADMAP. This is sub-project (4) of five, and the last
design in this initiative.

## Purpose

Feature issues can grow larger than the team's PR-review appetite. Devs realise mid-implementation
that their draft PR will be too big to review; reviewers ask "please split this". Today the
process has no tool for that. Modularity's L3 nodes address it only for teams that plan the
decomposition upfront in the docs repo — many features never get that treatment, and PR-size
concerns are non-deterministic anyway.

This sub-project adds a **reactive** tool: `/split-issue`. It operates purely at the tracker level,
never touches `docs/`, and can be triggered by anyone (dev, reviewer, PL) at any time — before
implementation starts, mid-implementation, or during PR review.

## Design principle (locked)

**The direction of influence is one-way: `docs/` (the docs repo master plan) → tracker / service
repos. Never the reverse.** Every skill that touches both must respect this direction. Concretely:

- Docs-repo skills (`/modularize`, `/plan-sprint`, `/new-feature`, `/create-issues`) read docs state
  and write to `docs/` and/or the tracker.
- Tracker-side skills that respond to *implementation* concerns (`/split-issue`) may read
  tracker/PR state but **must never write to `docs/`**.

This means `/split-issue` cannot promote sub-issues to L3 tree nodes, cannot rename tree nodes,
cannot append to ROADMAP. Any planning-time decomposition that should live in the tree stays with
`/modularize`.

## Scope

- **In:** the new `/split-issue <issue-id> [--pr <pr-url>]` skill (initial + `--refine` mode);
  tracker adapter branches for Plane / GitHub / GitLab; a small tweak to `/create-issues` that
  offers to materialise L3 tree nodes as sub-issues at parent-issue-creation time (docs → tracker
  direction only); a mention in `docs/process/README.md` and `README.md`.
- **Out (YAGNI):** any hook or CI check that forces splitting based on PR size; splitting issues
  that already have sub-issues without `--refine`; automated sub-issue closure detection; a
  whole-project "audit for oversized issues" tool; cross-parent sub-issue rebalancing; any
  interaction with docs repo files (see design principle above); a `/close-sprint` helper.

## Writing principle

Concise, technical, complete. `/split-issue` is a tracker-side execution adapter; the docs
process is unchanged by its existence.

## Part 1 — Two independent decomposition flows

- **Proactive planning (tree-bound)** — the existing `/modularize <l2-slug>` from sub-project (2).
  Produces L3 tree nodes when DM/PL wants to record a decomposition. Optional per feature. Never
  runs automatically.
- **Reactive splitting (tracker-only)** — new `/split-issue`. Produces tracker sub-issues on
  demand. Never touches `docs/`.

The two flows do not conflict. A feature can be:

- Planned proactively (L3 tree → sub-issues at `/create-issues` time), OR
- Started as a single issue then split reactively (`/split-issue`), OR
- Delivered as a single issue with no splitting at all.

All three are valid; nothing is enforced.

## Part 2 — `/split-issue` skill

**Location:** `.claude/skills/split-issue/SKILL.md`. `disable-model-invocation: true` (user-invoked).

**Actors:** dev, reviewer, PL, DM — anyone with tracker access. No role gate.

**Inputs:**

- `<issue-id>` (required) — tracker-native id. Plane work-item id / GitHub `#NN` (or full URL) /
  GitLab `#NN` (or full URL).
- `--pr <pr-url>` (optional) — link to a draft/open PR. If provided, the skill reads the PR's
  file-list + short summary via the git-host MCP as extra grounding for scope. Useful when the
  trigger is "my in-flight PR is huge".

**Flow:**

1. **Resolve tracker platform** from `.claude/tracker.json` → `tracker.platform`.
2. **Fetch the issue** via the tracker adapter — title, description, existing sub-issues (if any).
3. **Refuse-with-hint on existing sub-issues** — if the issue already has sub-issues, refuse and
   print `use --refine to modify existing sub-issues`. Exit.
4. **Fetch PR context** — if `--pr` was passed, fetch the PR file-list and summary via git-host
   MCP. Skip on failure with a warning; continue without PR grounding.
5. **Auto-propose sub-issues.** Based on issue description + optional PR context, propose N
   sub-issues with `{title, summary}`. Sizing target: each sub-issue should map to a small-diff PR
   (this is the whole point). The proposal is level-3-shaped semantically without being tied to
   the tree.
6. **Q&A refine, one at a time.** For each proposed sub-issue, ask keep / edit / drop. Edits
   prompt only for `title`, `summary`.
7. **Add-more loop.** After walking every proposal, ask "any missing?". If yes, mini Q&A per new
   sub-issue.
8. **Confirm parent.** Default parent is `<issue-id>`. User can override to a different parent
   (rare — supported for cases where the "split" reveals the work belongs under a sibling issue).
9. **Materialise sub-issues in the tracker.** One tracker call per sub-issue via the adapter.
   Each sub-issue is created with `parent = <resolved-parent>`, title from Q&A, description
   containing a link back to the parent issue and the free-text summary from Q&A.
10. **Print summary** — list of created sub-issue ids + URLs.

**No spec files written. No ROADMAP updates. No tree files touched. No arch-log entry.** The skill
lives entirely at the tracker / git-host layer.

## Part 3 — `--refine` mode

`/split-issue <issue-id> --refine` re-runs on the issue's existing sub-issues.

- **Add** — same mini-Q&A as Part 2 step 7; new sub-issue created in tracker under the parent.
- **Rename** — edit title / summary of an existing sub-issue in the tracker.
- **Drop** — close the sub-issue in the tracker with reason `dropped during /split-issue --refine`.
  Do not delete the tracker item — the closure preserves audit history.
- **Move** — not supported. Rare enough to handle as drop + add. If a sub-issue is genuinely
  under the wrong parent, the manual tracker UI is the right escape hatch.

`--refine` never touches docs either. Same principle applies.

## Part 4 — Tracker adapter branches

Following the pattern in `docs/process/README.md`:

| Platform | Sub-issue mechanic |
|----------|--------------------|
| **Plane** | Create a work item with `parent = <issue-id>`. Use the parent's work-item type by default. |
| **GitHub** | Create an issue in the same repo, then attach as a sub-issue of `<issue-id>` via the `/sub_issues` API. Milestone inherited from parent when set. |
| **GitLab** | Create an issue in the same project. Link to parent via the linked-issue relation (`relates_to` or `blocks` — the specific verb configured in `docs/process/README.md`'s adapter section). Milestone inherited from parent when set. |

Adapter selection reads `.claude/tracker.json` → `tracker.platform`. All three write only to the
tracker.

## Part 5 — `/create-issues` tweak

Small addition to `.claude/skills/create-issues/SKILL.md` — this is the *docs → tracker* direction
and is therefore allowed:

- After creating the parent issue for a feature, look for L3 tree nodes under the feature's L2.
  Resolution: read ROADMAP.md, find the row for this feature, take the `Module` slug (from
  sub-project (3)), locate `docs/architecture/current/modularity/<...>/<module-slug>/` and check
  for L3 subdirs.
- **If none exist** — do nothing extra. Parent issue created; no sub-issues. Print one hint:
  `next: run /split-issue <parent-id> later if the scope needs splitting`.
- **If L3 nodes exist** — prompt `L3 nodes found under <l2-slug>. Create matching sub-issues from
  them? [y/N]`. Default N.
  - **y** — create one sub-issue per L3 node. Title from the node's `README.md` `title`
    front-matter; description from the node's `Purpose` body. Same tracker adapter used by
    `/split-issue`.
  - **n** — skip. L3 nodes remain as planning-only artifacts in the tree.
- **If ROADMAP row has blank `Module`** — no L3 tree to inspect. Same as "none exist" — parent
  issue only.

The tweak never fails a `/create-issues` run — sub-issue materialisation is always optional.

## Part 6 — Ripples

- **Add:** `.claude/skills/split-issue/SKILL.md`.
- **Modify:** `.claude/skills/create-issues/SKILL.md` (Part 5 tweak);
  `docs/process/README.md` (mention `/split-issue` in the per-feature flow; extend the tracker
  adapter table with the sub-issue-mechanic column above; state the docs → tracker one-way
  principle explicitly if not already there); `README.md` (add `/split-issue` to the commands
  list).
- **Not modified:** `docs/process/AGENTS.md` — no new hard rule. Splitting is voluntary; there is
  nothing to enforce.

## Part 7 — Ripples to other sub-projects

- **(1) Solution-architecture home** — none. `/split-issue` never touches `current/` or `logs/`.
- **(2) Modularity skill** — none. `/split-issue` does not reuse `/modularize`'s node-creation
  code; it does not read tree files; it does not write tree files.
- **(3) Sprint-driven ROADMAP** — the `/create-issues` tweak reads ROADMAP's `Module` column
  (already added by (3)). No other interaction.

## Part 8 — Non-goals recap

- No PR-size CI hook or auto-trigger for splitting. It's always a human call.
- No promotion of sub-issues to L3 tree nodes. Removed on principle (docs → tracker only).
- No sub-issue spec files. Sub-issues live in the tracker, not `docs/`.
- No cross-parent sub-issue rebalancing.
- No sub-issue-per-file heuristics — proposals are semantic, based on issue description and
  optional PR grounding.
- No `/close-sprint` — closing a sprint is a tracker action.

## Open questions

None blocking implementation.
