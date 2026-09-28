---
name: plan-sprint
description: Interactive sprint planning — reads the modularity tree, shows candidate scope (unexpanded L1/L2 placeholders + in-flight items), asks DM to pick sprint contents (from tree + ad-hoc), creates the tracker milestone, appends rows to ROADMAP.md, and prompts to bulk-create tracker issues via /create-issues. Grows the plan one sprint at a time. Supports --refine on the currently active sprint.
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
     (P0/P1/P2), `change_level` (C0/C1/C2). `Module` column will be left blank for these.
6. **Create the tracker milestone** via the tracker adapter (read
   `.claude/tracker.json` → `tracker.platform`):
   - **Plane** — create a cycle named `<sprint-name>`.
   - **GitHub** — create a milestone in the docs repo named `<sprint-name>`.
   - **GitLab** — create a milestone at the project (or group, per team convention) named `<sprint-name>`.
7. **Append rows to `ROADMAP.md`.** For each picked item, append a row to the table with columns:

       | <feature-title> | <priority> | <level> | <l2-slug or blank> | | | <sprint-name> | |

   Column order matches the schema in `docs/superpowers/specs/2026-08-12-sprint-driven-roadmap-design.md` Part 3:
   `Feature | Priority | Level | Module | Spec | Issues | Milestone | Link`. `Spec`, `Issues`, and
   `Link` start empty. `/create-issues` fills `Issues` at planning time (step 8 or standalone);
   `/new-spec` sets `Spec` at spec-write time; `/roadmap-sync` reconciles `Link` and `Milestone`
   during planning refresh.
8. **Prompt to bulk-create tracker issues.** After the rows are appended, prompt
   `Create tracker issues for these <N> features now? [Y/n]` (default `Y`). On `Y`, invoke
   `/create-issues <name>` once per newly appended row, in the order they were appended, one
   at a time — tracker item numbering depends on serialized creation, and parallelizing here
   scrambles the board. Print each command's one-line summary as it completes. On `n`, skip
   — the user can run `/create-issues <name>` per row later. If any invocation fails
   mid-loop, stop, print the failure and which rows still lack issues, and exit; the DM
   re-runs `/create-issues` per gap.
9. **Print next-steps summary.** For each row whose `Issues` column is now filled: `next:
   /new-spec <slug>` if a spec is needed (C1/C2), or `next: implement <slug>` if C0. For each
   row whose `Issues` column is still empty (step 8 skipped, or exited early on mid-loop
   failure): `next: /create-issues <slug>` first, then the `/new-spec` line if C1/C2.

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
