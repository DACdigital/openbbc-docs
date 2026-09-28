---
name: arch-review
description: "Post-approval sync — reconcile docs/architecture/current/ with an approved spec"
disable-model-invocation: true
---

# /arch-review `[<name>] [--recent N] [--apply] [--pr] [--exclude <spec>] [--codename <slug>]`

## Purpose

Reconcile `docs/architecture/current/` (L1 + L2 only) with what one or more approved specs commit
to. Runs **after** a spec is approved — the spec PR need not be merged yet, so sync work can happen
in parallel on its own branch and its own PR. Complements `/spec-review` (which judges the spec
itself, not `current/`).

This command **replaces** the earlier read-only `/arch-review` gate that commented on the spec PR.
That gate's rubric was merged into `/spec-review`, and this command now owns the current-sync job
instead.

Manual only. Not part of the pre-merge gate sequence. Recommended (not required) by `/spec-review`
after a PASS / PASS_WITH_ISSUES verdict on C1 or C2 specs.

## Inputs

- `<name>` (optional) — a file basename (with or without the `-design.md` suffix) in
  `docs/superpowers/specs/`. If supplied, only that spec is compared.
- `--recent N` (default 3) — if `<name>` omitted, take the N most recently modified specs
  matching `docs/superpowers/specs/*.md`.
- `--exclude <spec>` — drop a spec from the shortlist.
- `--codename <slug>` — override the auto-generated codename for the arch-log entry (default:
  `sync-<spec-slug>` for a single spec, `sync-<YYYY-MM-DD>-<N>-specs` for multiple).
- `--apply` (default off) — without this flag, the command emits the delta report only. With it,
  the command creates a branch, applies non-AMBIGUOUS edits, and writes the arch-log entry —
  **without committing**. The user reviews `git status` + `git diff` and commits manually.
- `--pr` (default off, only meaningful with `--apply`) — after applying, stage + commit + push +
  `glab pr create` for the arch sync PR. Without `--pr`, the branch stays local.

## Steps

1. **Locate the spec shortlist.** Either the `<name>` argument or the top-N most recently
   modified specs matching `docs/superpowers/specs/*.md`. Apply `--exclude <spec>` if passed. Print the final shortlist to the user and ask them to confirm before proceeding.
   Manual invocation is the sole trigger — the skill does NOT probe PR state, MR approval, or any
   other remote signal. Whoever ran `/arch-review` has already decided the specs are worth
   syncing against; the skill trusts that decision.
2. **Invoke the `architecture-review` subagent** (`.claude/agents/architecture-review`) with the
   confirmed spec shortlist and the full `docs/architecture/current/` L1 + L2 surface. The agent
   returns a delta report classified `CONTRACT_SHAPE` / `SCOPE_MOVE` / `AMBIGUOUS` per delta and an
   overall verdict of `IN SYNC` / `NEEDS UPDATE` / `AMBIGUOUS`.
3. **Print the delta report** in this session verbatim.
4. **Exit paths by verdict:**
   - `IN SYNC` — print `Nothing to sync. current/ already matches the approved spec(s).` and exit.
     No branch, no PR, no log entry.
   - `AMBIGUOUS` (no non-AMBIGUOUS deltas) — print the report and exit. No branch. The user must
     resolve the ambiguity manually (usually by editing the spec).
   - `NEEDS UPDATE` — proceed to step 5.
5. **If `--apply` NOT passed** — print `Report only. Pass --apply to create a branch and edit current/.`
   and exit. No branch.
6. **If `--apply` IS passed:**
   1. Compute the codename: `--codename` value if given; else `sync-<spec-slug>` (single spec) or
      `sync-<YYYY-MM-DD>-<n>-specs` (multiple).
   2. Verify current working tree is clean (`git status --porcelain` empty). If not, abort with a
      message asking the user to commit or stash first.
   3. Create the branch: `git checkout -b arch/<codename>` from the current HEAD.
   4. For each `CONTRACT_SHAPE` and `SCOPE_MOVE` delta in the report, apply the "Recommended edit"
      to the referenced `<file>:<section>` inside `docs/architecture/current/`. Never touch
      `docs/conventions/`, `docs/process/`, `docs/superpowers/`, `docs/repos/`, or any spec. Skip
      every `AMBIGUOUS` delta (they stay in the report only).
   5. Write **one** new arch-log entry at
      `docs/architecture/logs/<YYYY-MM-DD>-<codename>/README.md` using the log template below.
   6. Run `git status` + `git diff --stat` and print both.
7. **If `--pr` IS passed** — stage the changed files (`git add docs/architecture/current/
   docs/architecture/logs/<YYYY-MM-DD>-<codename>/`), commit with message
   `arch: sync current/ with <spec-name>[, +N more]`, push (`git push -u origin arch/<codename>`),
   and open the PR (`glab pr create --title "arch: sync current/ with <spec-name>[, +N more]"`
   with a body summarizing the deltas applied and the source spec shortlist).
   Otherwise, print `Branch arch/<codename> created and edited but NOT committed. Review with
   git status / git diff, then commit and push manually.` and exit.

## Arch-log template

The single log entry written on `--apply` is a full `docs/architecture/logs/<YYYY-MM-DD>-<codename>/
README.md` that satisfies `architecture-log-review`'s required fields. Emit it verbatim (fill in
placeholders):

```markdown
---
date: <YYYY-MM-DD>
codename: <codename>
kind: sync
---

# Sync — align current/ with <primary-spec-name>[, +N more]

## Driver
Post-approval sync of `docs/architecture/current/` with the following spec(s):
- `<spec-path>`
(one line per spec in the shortlist)

## Decision
Apply the deltas listed below to `docs/architecture/current/`. AMBIGUOUS deltas (if any) are
recorded here for human resolution but were NOT applied.

## Rationale
Approved spec is the design contract; `current/` is the architecture-of-record and must reflect it.
Per project rule: L2 modularity nodes are inspirational — specs are the impl contract; sync applies
CONTRACT_SHAPE deltas without altering module Purpose, and only introduces new L1/L2 surface for
SCOPE_MOVE deltas.

## Alternatives rejected
- Leaving current/ stale until the next `/modularize` run — rejected: drift accumulates silently and
  a new dev reading current/ builds against the wrong contract.
- Auto-applying all deltas including AMBIGUOUS ones — rejected: silent overrides of prior arch-log
  decisions require human review.

## Impact

### CONTRACT_SHAPE deltas applied (no purpose change)
- `<file>:<section>` — <one-line change summary>
(or "None.")

### SCOPE_MOVE deltas applied (new surface / ownership / boundary)
- `<file>:<section>` — <one-line change summary>
(or "None.")

### AMBIGUOUS deltas NOT applied — human review required
- `<file>:<section>` — spec sentence: "<exact quote>". Why ambiguous: <one line>.
(or "None.")

## Links
- Source spec(s): <spec-path>[, spec PR link], ...
- Related prior arch-log entries: <log-path> — extends | supersedes | confirms
  (or "None.")
```

## Guardrails

- **Surface cap.** Only touch files inside `docs/architecture/current/` and the new log entry under
  `docs/architecture/logs/<YYYY-MM-DD>-<codename>/`. Never edit `docs/conventions/`,
  `docs/process/`, `docs/superpowers/`, `docs/repos/`, or any spec file.
- **Never lower a documented arch decision.** If `current/` already reflects a documented decision
  from a prior arch-log entry and the spec doesn't explicitly override it, that's `AMBIGUOUS` in the
  agent's report — do not apply.
- **Superseded specs.** Newer spec wins; older listed as `SUPERSEDED — no delta computed`.
- **No commit without user review.** `--apply` stops after `git diff --stat`. Commit happens either
  in `--pr` flow (after push) or manually.

## Config

Reads `.claude/tracker.json` → `comms` (optional notification only). Uses `glab` for PR resolution
(step 1) and PR creation (step 7).
