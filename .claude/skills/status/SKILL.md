---
name: status
description: Read-only dashboard of per-feature spec state, child-issue counts, and milestone progress from the tracker
disable-model-invocation: true
---

# /status

## Purpose

Read-only delivery dashboard. For every feature in `ROADMAP.md`, shows spec state (draft / in review /
merged), open/closed child-issue counts, and milestone progress — pulled live from the tracker. Writes
nothing; if the numbers look stale, run `/roadmap-sync` first.

## Inputs

- None required — reports on every feature row in `ROADMAP.md`. Optionally accepts `<name>` to report
  on just one feature.

## Steps

1. **Read `ROADMAP.md`** for the feature list (or just `<name>` if given) and each row's `Issues`/
   `Milestone`/`Link` values.
2. **Spec state**: check whether a file matches `docs/superpowers/specs/*-<feature>-design.md`
   and whether its PR is open, has verdicts posted, or is merged (draft / in review / merged).
3. **Branch on `tracker.platform`** to pull live counts for the row's linked items:
   - **plane** → work-item states under the feature's epic; cycle progress if a cycle is linked.
   - **github** → issue/sub-issue open/closed counts; milestone progress if one is linked.
   - **gitlab** → issue open/closed counts under the epic; milestone progress if one is linked.
4. **Print a table**: `Feature | Spec state | Issues (open/closed) | Milestone progress`, one row per
   feature, plus each feature's `Priority` and `Level` from `ROADMAP.md` for context.

## Config

Reads `.claude/tracker.json` → `tracker` (platform, org, projectId) to select the adapter in step 3.
Does not read `gitHost` or `comms`. Writes nothing — this command is read-only.
