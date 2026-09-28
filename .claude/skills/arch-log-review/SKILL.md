---
name: arch-log-review
description: On-demand review of a PR that touches docs/architecture/current/ — invokes the architecture-log-review agent and routes the verdict. PASS/PASS_WITH_ISSUES is posted as a PR comment; FAIL goes back to the author in-session and is never posted. Independent of C1/C2 — any PR touching current/ needs the paired log entry.
disable-model-invocation: true
---

# /arch-log-review `[<pr-url>]`

## Purpose

Runs `architecture-log-review` on a PR that touches `docs/architecture/current/` and routes the
verdict: `PASS` / `PASS_WITH_ISSUES` is posted as a PR comment, `FAIL` is reported to the author
in the invoking session and posted nowhere. Independent of the feature-level `spec-review` gate —
a pure refactor of `current/` with no feature spec still needs this check.

## Inputs

- `<pr-url>` (optional) — full PR/MR URL. If omitted, use the current git branch's open PR/MR
  from the configured git-host.

## Steps

1. **Resolve target PR** — either from `<pr-url>` or the current branch's open PR (via git-host
   MCP).
2. **Fetch PR diff** — via git-host MCP; extract the file list.
3. **Guard: does PR touch `docs/architecture/current/`?** If no, print "no architecture changes;
   skipping" and exit with no verdict.
4. **Invoke** the `architecture-log-review` agent with the PR diff as input.
5. **Route the verdict** per `docs/process/AGENTS.md` — a `FAIL` verdict is never posted:
   - **`FAIL`** → report the verdict and its full findings to the author in this session,
     verbatim; no PR comment. Do not edit the log entry to satisfy the findings — the author
     decides.
   - **`PASS` / `PASS_WITH_ISSUES`** → post the verdict plus findings as a comment on the PR via
     git-host MCP.

   Do not commit anything to the repo — a PR comment is the only artifact this command ever
   leaves behind.

## Config

Reads `.claude/tracker.json` → `gitHost` (platform + org). Read-only against `docs/`; writes only
a PR comment on passing verdicts.
