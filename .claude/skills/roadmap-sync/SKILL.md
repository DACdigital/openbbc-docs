---
name: roadmap-sync
description: Refresh ROADMAP.md issue/tracker links and manage tracker milestones/epics. Planning-time only — invoke during /plan-sprint, sprint close-out, or when the user explicitly asks; never suggest as a follow-up to /new-spec, /create-issues, or other feature-scoped work.
disable-model-invocation: true
---

# /roadmap-sync

## Purpose

Keep `ROADMAP.md` current with the tracker. Refreshes the `Issues`, `Milestone`, and `Link` columns
for every feature row, and creates/updates tracker milestones/epics as needed. Read/write on
`ROADMAP.md`; read-only against the tracker itself (it doesn't create or close issues — that's
`/create-issues` and the service-repo devs).

## When to invoke

Planning contexts only:

- Inside `/plan-sprint` at sprint open/close, to reconcile ROADMAP with the tracker.
- When the user explicitly asks (`/roadmap-sync`, "sync the roadmap", "refresh ROADMAP links").
- During a periodic planning review where ROADMAP has drifted from tracker state.

Do **not** suggest this command as a follow-up to `/new-spec`, `/create-issues`, `/new-feature`,
`/spec-review`, `/arch-review`, or any other feature-scoped work. Those flows already write the
ROADMAP columns they own; a proactive `/roadmap-sync` suggestion there is noise and pushes the
user toward a planning-time operation from a non-planning context. See
`docs/process/AGENTS.md` — "Do not suggest /roadmap-sync outside planning."

## Inputs

- None required — syncs every row in `ROADMAP.md`. Optionally accepts `<name>` to scope the sync to
  one feature.

## Steps

1. **Read `ROADMAP.md`** and, for each row (or just `<name>` if given), read its existing `Issues` and
   `Link` values to find the tracker items already associated with it.
2. **Branch on `tracker.platform`**:
   - **plane** → refresh child work-item states under the feature's epic; ensure a cycle exists for
     the feature's target milestone/timebox if `Milestone` names one, creating it if missing.
   - **github** → refresh sub-issue/tracking-issue states; ensure a milestone exists for the feature's
     target release if `Milestone` names one, creating it if missing.
   - **gitlab** → refresh linked-issue states under the feature's epic; ensure a milestone exists
     for the feature's target release if `Milestone` names one, creating it if missing.
3. **Rewrite `Issues`** with current links/counts, `Milestone` with the resolved milestone/cycle name,
   and `Link` with the epic/tracking-item URL — for every row touched.
4. Leave `Feature`, `Priority`, `Level`, and `Spec` untouched — those are owned by `/plan-sprint`
   (seeding) and `/new-spec` (change level), not this command.

## Config

Reads `.claude/tracker.json` → `tracker` (platform, org, projectId) to select the adapter in step 2.
Does not read `gitHost` or `comms`.
