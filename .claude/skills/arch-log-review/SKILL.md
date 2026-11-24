---
name: arch-log-review
description: Review a change to docs/architecture/current/ for the paired arch-log entry — invokes the architecture-log-review agent and routes the verdict. Runs with or without an open PR — no PR selects pre-push mode (how /arch-change gates its push). PASS/PASS_WITH_ISSUES is posted as a PR comment (on-PR) or handed back to the caller to post as the new PR's first comment (pre-push); FAIL goes back to the author in-session and is never posted. Independent of C1/C2.
disable-model-invocation: true
---

# /arch-log-review `[<pr-url>]`

## Purpose

Runs `architecture-log-review` on a change that touches `docs/architecture/current/` and routes
the verdict. A `FAIL` is reported to the author in the invoking session and posted nowhere.
Independent of the feature-level `spec-review` gate — a pure refactor of `current/` with no feature
spec still needs this check.

Runs in either of two modes, decided by whether a PR exists:

- **pre-push mode** (no PR for the current branch) — how `/arch-change` step 12 invokes it. Reviews
  the local branch diff and returns the verdict into the calling session; it gates the push and
  there is nothing to comment on.
- **on-PR mode** (PR found) — invoked by hand on an already-open PR, e.g. one opened after a
  standalone `/modularize` run or a manual edit, or after the author pushed a fix. A passing verdict
  is commented; a `FAIL` still is not.

## Inputs

- `<pr-url>` (optional) — full PR/MR URL. If omitted, look up the current branch's open PR/MR. An
  open PR is **not** required: its absence selects pre-push mode rather than failing the command.

## Steps

1. **Resolve mode** — obtain the `gitHost` client per
   [Client resolution](../../../docs/process/README.md#client-resolution), then take the PR from
   `<pr-url>` or look up the current branch's open PR. Found → on-PR mode. None → pre-push mode.
2. **Get the diff.**
   - **on-PR** → fetch the PR diff.
   - **pre-push** → `git fetch origin <default>` then use `git diff origin/<default>...HEAD`
     (committed changes on the branch; uncommitted changes are not reviewed).
3. **Guard: does the diff touch `docs/architecture/current/`?** If no, print "no architecture
   changes; skipping" and exit with no verdict.
4. **Invoke** the `architecture-log-review` agent with the diff as input.
5. **Route the verdict** per `docs/process/AGENTS.md` — a `FAIL` verdict is never posted:
   - **`FAIL`** → report the verdict and its full findings to the author in this session, verbatim;
     no PR comment. In pre-push mode this also blocks the push (`/arch-change` step 12). Do not
     edit the log entry or `current/` to satisfy the findings — the author decides.
   - **`PASS` / `PASS_WITH_ISSUES`** → on-PR: post the verdict plus findings as a comment on the PR
     and report it in-session. Pre-push: return it into the session; the caller MUST post it
     verbatim as the new PR's first comment as soon as it opens one (`/arch-change` step 12).

   Do not commit anything to the repo — a PR comment is the only artifact this command ever leaves
   behind.

## Config

Reads `.claude/tracker.json` → `gitHost` (platform, instanceUrl, org); the `gitHost` client is
obtained per [Client resolution](../../../docs/process/README.md#client-resolution). Read-only
against `docs/`; writes only a PR comment on passing on-PR verdicts.
