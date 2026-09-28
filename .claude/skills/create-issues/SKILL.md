---
name: create-issues
description: Create a tracker issue for a feature discussed during planning. One call per feature, one tracker issue created per call. No spec required. Sub-issues, if ever needed, are created explicitly via /split-issue — never here.
disable-model-invocation: true
---

# /create-issues `<name>`

## Purpose

Create the tracker issue for a feature discussed during planning. Called during (or shortly
after) `/plan-sprint` — one call per new `ROADMAP.md` row. The issue stands alone: no children,
no assumptions about hierarchy. If the scope later needs breaking down, that's `/split-issue`'s
job, invoked explicitly.

This command does **not** read specs, plans, or the modularity tree. Everything it needs is on
the ROADMAP row. The old post-spec-merge decomposition flow is gone — see
`docs/superpowers/specs/2026-09-09-create-issues-planning-time-design.md` for the reshape.

## Inputs

- `<name>` (required) — the feature name; must match a `Feature` value in `ROADMAP.md`. Fails
  if no such row exists.

## Steps

1. **Resolve the ROADMAP row** by `<name>`. If no row matches, exit with
   `no ROADMAP row for <name> — run /plan-sprint first to append it`. Read the row's
   `Feature`, `Priority`, `Level`, `Module`, `Milestone` columns.
2. **Determine target location** for the new issue:
   - `.claude/tracker.json` → `tracker.platform` picks the adapter.
   - For **plane**, the location is `tracker.projectId` — no repo selection needed.
   - For **github** and **gitlab**, the issue goes in a service repo. Read `workspace/INDEX.md`
     for the list of repos. If there is exactly one repo, use it. If there are multiple, prompt
     the user to pick one (default: the repo whose name matches the row's `Module` slug when
     one matches, otherwise no default). Do not attempt to infer from module front-matter —
     the modularity tree does not carry a `home_repo` field today.
3. **Create one tracker issue** via the adapter:
   - **Plane** — create a work item under `tracker.projectId` with `title = <feature>`, priority
     mapped from the row, and no parent (this is a top-level item). Assign to the milestone/cycle
     whose name matches the row's `Milestone` column, creating it if missing.
   - **GitHub** — create an issue in the resolved service repo with `title = <feature>`. Assign
     to the milestone whose name matches the row's `Milestone`, creating it if missing.
   - **GitLab** — create an issue in the resolved service repo with `title = <feature>`. Assign
     to the milestone whose name matches the row's `Milestone`, creating it if missing.

   The issue description is a two-line stub: `Feature: <name>` on line 1, `Planned in
   <sprint-name>` on line 2. No spec link, no scope block — those come later if a spec is
   written.
4. **Update `ROADMAP.md`.** Fill the row's `Issues` column with the created issue's link;
   leave `Link` blank (there is no epic/tracking-issue at this stage — `/roadmap-sync`
   manages that column during planning if a team uses epics).
5. **Print one-line summary** — `created <issue-id> at <url>` — and exit.

## What this command does NOT do

- Read specs, plans, or the modularity tree.
- Create sub-issues, child work items, or L3-derived shards. That is `/split-issue`'s job,
  invoked explicitly when needed.
- Suggest `/roadmap-sync` on completion. Per `docs/process/AGENTS.md`, that command is
  planning-only and callers should not chain into it.
- Run any pre-flight gate other than "the ROADMAP row exists."

## Config

Reads `.claude/tracker.json` → `tracker` (platform, org, projectId) to select the adapter in
step 3. Also reads `gitHost` when `tracker.platform` is `github` or `gitlab`, to know which org
the target service repo lives in. Reads `workspace/INDEX.md` to list candidate repos in step 2.
