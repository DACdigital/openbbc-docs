---
name: propagate-rules
description: Generate repo/language-specific agent rules from docs/conventions/ into each service repo via a code PR
disable-model-invocation: true
---

# /propagate-rules

## Purpose

Onboarding Phase 6. Keeps the **project-wide** conventions in this docs repo as the single source, and
delivers the **repo/language-specific** slice into each service repo. Rules land via a **code PR** in
the service repo (respecting the code gate), merging with — never clobbering — any existing agent rules.

## Inputs

None required — interactive. Reads `docs/conventions/`, the `workspace/<repo>/` checkouts, their
`docs/repos/<repo>.md` indexes, and `.claude/tracker.json` → `gitHost`. Run `/setup-workspace` first so
the repos exist locally.

## Steps

1. **Read the master conventions** in `docs/conventions/` (architecture, naming, api, persistence,
   events, testing). These stay here — they are the project-wide source of truth and are **not** copied
   into repos.
2. **For each `workspace/<repo>/`**, derive only the **repo/language-specific** rules — the concrete,
   stack-bound guidance the cross-repo conventions imply for this repo (e.g. pytest + Playwright E2E for
   a TypeScript frontend; Maven/Spring layering + PMD rules for a Java service). Use the repo's
   `docs/repos/<repo>.md` index for its stack.
3. **Compose the repo's agent rules** — an `AGENTS.md` (and/or `.claude/` rules) containing:
   - a short **pointer** back to this docs repo's `docs/conventions/` for the project-wide standards;
   - the repo/language-specific rules from step 2.
4. **Merge, don't clobber.** If the repo already has `AGENTS.md` / `CLAUDE.md` / `.claude` content,
   append and reconcile rather than overwrite; preserve the team's existing rules.
5. **Deliver via PR.** Write the changes on a branch in `workspace/<repo>/` and open a code PR on the
   git host for human review/merge. A freshly-created, empty greenfield repo (no history to review
   against) may commit directly to its default branch instead.
6. **Report** one line per repo: the PR URL (or a direct-commit note) and whether rules were merged into
   existing ones or newly created.

## Config

Reads `.claude/tracker.json` → `gitHost`. Does not touch `tracker` or `comms`.
