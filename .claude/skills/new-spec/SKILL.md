---
name: new-spec
description: Initialize a feature spec from an existing arch node. Orchestrates superpowers:brainstorming into docs/superpowers/specs/YYYY-MM-DD-<name>-design.md, proposes the C0/C1/C2 change level, gates the push on /spec-review, and opens the spec PR carrying the passing verdict as its first comment. Blocks if the target capability is a novelty not yet in current/.
disable-model-invocation: true
---

# /new-spec `<name>` `[--issue <tracker-id>]`

## Purpose

Turn an idea into a reviewable feature spec, grounded in an existing node of the solution
architecture. Orchestrates `superpowers:brainstorming` to produce
`docs/superpowers/specs/YYYY-MM-DD-<name>-design.md`, proposes the C0/C1/C2 change level per
`docs/process/README.md`. Ends by opening the spec PR, gated: `/spec-review`
runs before the push against the local commit, and the branch is pushed and the PR opened only
when its verdict is `PASS` or `PASS_WITH_ISSUES`. The passing verdict is posted as the new PR's
first comment. Single unified gate covering structure, naming, architecture, and contracts.

This command does **not** write to `docs/architecture/current/` or `docs/architecture/logs/`.
If the target capability is a novelty not yet in the architecture, the command blocks and
tells the user to run `/arch-change <name>` (type: new) first.

## Inputs

- `<name>` (required) — kebab-case feature name (no folder). The skill stamps today's date
  at write time. Fails if a file matching
  `docs/superpowers/specs/*-<name>-design.md` already exists (offer to resume that feature
  instead).
- `--issue <tracker-id>` (optional) — a tracker ticket ID (e.g. `PROJ-42`) whose title and
  description seed the brainstorm and inform the arch-node mapping.

## Steps

1. **Resolve mapping** to a `current/` node — try each mode in order and stop at the first
   hit:
   - `<name>` argument matches a modularity slug (walk
     `docs/architecture/current/modularity/**/` and match by `id` front-matter or dir name)
     → mapped node = that slug.
   - `--issue <tracker-id>` present → pull the ticket via the configured tracker adapter
     (`.claude/tracker.json` → `tracker.platform`; `tracker` client per
     [Client resolution](../../../docs/process/README.md#client-resolution)), extract title and
     description; fuzzy-match the title against modularity node titles. Ticket body seeds the
     brainstorm regardless.
   - Prior conversation clearly names a module (only when the skill is invoked mid-
     conversation; not applicable in fresh sessions) → propose that mapping.
   - Exactly one plausible modularity node by fuzzy match on `<name>` → propose that mapping.
   - None of the above → treat as ambiguous.

2. **Confirm or ask**:
   - **Confidently mapped** → report and confirm:
     `"This spec will attach to <slug> (L<level>, <path>). Continue? [Y/n/change]"`.
   - **`change` selected** or **ambiguous** (zero or multiple candidates) → present a
     numbered list of existing modularity nodes plus a `none` option; user picks one.
   - **`none` chosen** or **no candidates at all** → **BLOCK** and print:

         "This looks like a novelty. Run /arch-change <name> (type: new) first to add it to
         docs/architecture/current/, then re-run /new-spec <name>."

     Exit without writing any files.

3. **Load context**:
   - Mapped node's `README.md` + parent chain (walk `parent` back to root).
   - `docs/architecture/current/nfrs.md`.
   - `docs/architecture/current/assumptions.md`.
   - Tracker ticket body if `--issue` was passed.
   - If a ROADMAP row matches `<name>`, its `Priority` and `Milestone` columns.

4. **Run `superpowers:brainstorming`** with the required-sections override from
   `docs/process/AGENTS.md`. The resulting `spec.md` MUST contain, in order:
   Business value / Why, Change level (C0/C1/C2), Scope (in / out), Contracts, Acceptance
   criteria, Risks & assumptions.

5. **Mid-brainstorm drift safeguard**: if the design surfaces net-new arch not covered by the
   mapped node — e.g., proposes a whole new service, or a new contract on a different service
   — halt with the same block message as step 2 and exit without writing files.

6. **Write `docs/superpowers/specs/YYYY-MM-DD-<name>-design.md`** (with today's date) containing
   the six required sections.

7. **Propose the change level (C0/C1/C2)** per `docs/process/README.md`'s change
   classification, with a one-line justification. Write it into the spec's
   "Change level (C0/C1/C2)" section. Ambiguous → pick higher (hard rule). Never route
   through the C0 fast lane if the change touches DB, API, events, security, tenant
   isolation, or an upgrade.

8. **Implementation plan stays local.** If the author runs `superpowers:writing-plans`, the plan
   is a working note under the gitignored `docs/superpowers/plans/`. It is never committed, in
   any repo.

9. **Commit locally**: create a branch and commit only the newly-written
   `docs/superpowers/specs/YYYY-MM-DD-<name>-design.md`, never a plan file. Do
   not push yet — the push is gated by step 10.

10. **Spec gate, before the push.** Invoke `/spec-review`'s subagent against the committed spec.
    There is no PR yet, so it runs in its **pre-push mode** and returns its verdict into this
    session instead of commenting.

    - **`FAIL`** → **the push does not happen.** No PR is opened, nothing is posted as a comment,
      and no comms notification is sent. Print the verdict and its full findings to the user,
      unedited and unabridged, and stop. Do not apply the findings yourself — the author decides
      what to change, then amends the commit and re-runs from step 10. Iterating here is the
      point: this is where a spec is allowed to fail as many times as it needs to, at zero cost
      to the PR's comment history.
    - **`PASS` / `PASS_WITH_ISSUES`** → push the branch, obtain the `gitHost` client per
      [Client resolution](../../../docs/process/README.md#client-resolution), open a PR against this repo's default
      branch titled after `<name>`, and **post the captured verdict verbatim as that PR's first
      comment**. The pre-push run returned the verdict into this session rather than commenting
      because no PR existed yet; opening one is what makes it postable, so posting it is not
      optional — a spec PR never exists without its `/spec-review` verdict on it. Report any
      `PASS_WITH_ISSUES` findings to the user as well; they are non-blocking, and the human PL
      still approves and merges.

11. If a comms channel is configured **and** step 10 posted a `PASS` / `PASS_WITH_ISSUES`
    verdict, obtain the `comms` client per [Client resolution](../../../docs/process/README.md#client-resolution) and
    post a short notification that the spec PR for `<name>` is open. A `FAIL` notifies
    nobody.

## Config

Reads `.claude/tracker.json` → `tracker` (only when `--issue` is used), `gitHost` (the docs repo
lives on `gitHost`; used to open the spec PR and post its verdict comment in step 10) and `comms`
(optional notification). Clients for each role are obtained per
[Client resolution](../../../docs/process/README.md#client-resolution).

## What this command never does

- Never writes to `docs/architecture/current/` or `docs/architecture/logs/`.
- Never runs `/modularize`.
- Never proceeds past step 2 when the target capability is a novelty; always blocks with the
  instruction to run `/arch-change` first.
- Never pushes **ahead of the spec gate**. The push is what step 10 gates, not something this
  command withholds: the moment `/spec-review` returns `PASS` or `PASS_WITH_ISSUES`, pushing and
  opening the PR is exactly what happens next. What never happens is a push while
  `/spec-review`'s verdict is `FAIL` — no push, no PR, no comment, no notification, and the
  local commit from step 9 is the only artifact of that run.
- Never posts a `FAIL` verdict anywhere. Failing findings are reported to the author in the
  session only.
- Never edits the spec to satisfy `/spec-review` findings on its own — it reports them and stops.
