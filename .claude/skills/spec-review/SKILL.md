---
name: spec-review
description: Run the spec-review agent against docs/superpowers/specs/*-<name>-design.md and route the verdict. PASS/PASS_WITH_ISSUES is posted as a comment on the open spec PR; FAIL goes back to the author in-session and is never posted. Runs with or without an open PR — no PR selects pre-push mode.
disable-model-invocation: true
---

# /spec-review `<name>`

## Purpose

Run the `spec-review` subagent against `docs/superpowers/specs/*-<name>-design.md` (a glob
because the filename carries the write-time date prefix) and route its verdict: `PASS` /
`PASS_WITH_ISSUES` is posted as a comment on the open spec PR, `FAIL` is reported to the author in
the invoking session and posted nowhere. This is the **single** gate check for C1 and C2 specs —
the human PL still approves and merges; this command only produces the verdict that informs that
decision. The verdict has four sub-verdicts (Structure, Naming, Architecture, Contracts);
dimensions that don't apply to the change are marked N/A.

Runs in either of two modes, decided by whether a spec PR exists:

- **pre-push mode** (no PR yet) — how `/new-spec` step 11 invokes it. The verdict comes back into
  the calling session and gates the push; there is nothing to comment on.
- **on-PR mode** (open PR found) — invoked by hand on an already-open spec PR, e.g. after the
  author pushed a fix. A passing verdict is commented; a `FAIL` still is not.

## Inputs

- `<name>` (required) — the feature name (the `<name>` slug used at `/new-spec` time). Fails
  if no file matches `docs/superpowers/specs/*-<name>-design.md`. An open PR is **not**
  required: its absence selects pre-push mode rather than failing the command.

## Steps

1. **Locate the spec PR** for `<name>` (the open PR that adds/modifies a file matching
   `docs/superpowers/specs/*-<name>-design.md`). None found → pre-push mode; carry on without
   one.
2. **Invoke the `spec-review` subagent** (`.claude/agents/spec-review`) against the resolved
   `docs/superpowers/specs/*-<name>-design.md` file, `docs/process/AGENTS.md`, and the full
   `docs/conventions/` (naming, architecture, api, persistence, events). The agent produces one
   verdict with four sub-verdicts.
3. **Route the verdict** per `docs/process/AGENTS.md` — a `FAIL` verdict is never posted:
   - **`FAIL`** → report the verdict and its full findings to the author in this session, verbatim.
     Post no PR comment and send no comms notification. In pre-push mode this also blocks the push
     (`/new-spec` step 11). Do not edit the spec to satisfy the findings — the author decides.
   - **`PASS` / `PASS_WITH_ISSUES`** → post the verdict plus sub-verdict lines and findings as a
     comment on the spec PR from step 1, and report it in-session as well. In pre-push mode there
     is no PR yet, so the caller MUST post the verdict verbatim as the new PR's first comment as
     soon as it opens one (`/new-spec` step 11) — a passing pre-push verdict is still a posted
     verdict, just posted by the caller a moment later.

   Do not commit anything to the repo — a PR comment is the only artifact this command ever leaves
   behind.
4. If the subagent raises the change level (e.g. C1 → C2), surface its `**Change level:**` line
   prominently wherever step 3 routed the verdict (a reviewer may raise, never lower, per the hard
   rules); the spec author then updates the spec's Change level section. The raise rides along
   with whatever verdict the agent emitted — it does not by itself make the verdict `FAIL` — so a
   `PASS_WITH_ISSUES` that recommends C2 is still commented on the PR.
5. **Recommend `/arch-review` on PASS or PASS_WITH_ISSUES.** After routing, print to the invoking
   session:
   `Recommended next step: after this spec merges, run /arch-review <name> to sync docs/architecture/current/ with the approved spec. It is optional and exits IN SYNC if there is no drift.`
   Do not run `/arch-review` automatically — sync is a manual, post-approval step (see
   `.claude/skills/arch-review/SKILL.md`).
6. If a comms channel is configured **and** the verdict was `PASS` or `PASS_WITH_ISSUES`, post a
   short notification with the verdict. A `FAIL` notifies nobody.

## Config

Reads `.claude/tracker.json` → `comms` (optional notification only). Does not read `tracker` or
`gitHost` — posting to this repo's own PR uses its own git remote, not a service repo or the tracker.
