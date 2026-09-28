---
name: sdd-rebase
description: Sync template-generic updates from dac-docs-template into this project docs repo. Fetches the upstream template, copies files that belong to the template (skills, agents, process docs, conventions), skips project-specific paths (tracker.json, .mcp.json, docs/architecture/, per-feature specs, ROADMAP.md, filled-in CLAUDE.md/README.md), and opens a PR with the delta. Never runs in the template repo itself.
disable-model-invocation: true
---

# /sdd-rebase

## Purpose

Automate what today happens as a hand-copied `chore: sync template updates from dac-docs-template!X`
commit: pull template-generic improvements (new skills, updated process rules, fixed conventions)
from the upstream `dac-docs-template` and open a PR against the current project docs repo. The PR
is the safety net — a human reviews the diff before merging.

The skill uses an **exclude list** rather than an include list: it copies almost everything from
the template and skips a short, stable list of paths that belong to the project. New template
files propagate automatically; project-owned paths are never clobbered.

## Inputs

- None required. Optional `--dry-run` prints the diff summary and exits without branching, staging,
  or pushing.

## Steps

1. **Refuse if run in the template repo itself.** Read `origin`'s URL; if it ends with
   `/dac-docs-template.git` (or matches the canonical template URL below), print
   `refusing to sync-from-template in the template repo itself` and exit. This is a downstream-only
   command.

2. **Ensure the `template` git remote exists.** Run `git remote get-url template`. If it fails, add
   it from the canonical URL — prompt the user first:

   ```
   No `template` remote configured. Add it pointing at
   git@gitlab.dac.digital:dacdigital/tech-pulse/spec-driven-development/dac-docs-template.git?
   [Y/n] (or paste a different URL)
   ```

   Default Y. On Y, `git remote add template <url>`. On a custom URL, use that instead.

3. **Fetch the template.** `git fetch template` — pulls the latest `template/master` locally.

4. **Compute the template-generic delta.** Run
   `git diff --stat template/master..HEAD -- <exclude-args>` where `<exclude-args>` is every path
   in `HEAD` and `template/master` except the exclude list below (use `:(exclude)` pathspecs).
   Also produce a plain list of paths that differ. If the list is empty, print
   `already up-to-date with template/master` and exit.

5. **On `--dry-run`, print the summary and exit.** Show which files would be updated, which would
   be created, which would be deleted, and — separately — the drift on the SKIP list files
   (`CLAUDE.md`, `README.md`, `.gitignore`) so the human knows what to reconcile by hand.

6. **Otherwise, create the sync branch.** Name it `sync/from-template-YYYY-MM-DD` (append
   `-<n>` if a same-day branch already exists). Branch off the current default branch.

7. **Copy template-generic files.** For every path in the delta that is not on the exclude or
   SKIP list, overwrite the local copy with the template's version. For deletions in
   `template/master`, run `git rm` locally. Do **not** touch anything on the exclude or SKIP list.

8. **Merge `.gitignore` as a union.** Do not overwrite. Read both versions, add lines present in
   `template/master:.gitignore` that are missing locally, preserve local-only lines.

9. **Commit.** One commit, title `sync: template updates from dac-docs-template (<N> files)`,
   body listing the source range (`template/master@<short-sha>..HEAD`), the files updated,
   deleted, and created, and — under a `## Drift on SKIP list (review manually)` heading — a diff
   of `CLAUDE.md`, `README.md`, and any other SKIP-list files that diverged. This tells the human
   reviewer what a follow-up manual merge would need to touch.

10. **Push and open a PR.** Push the branch; open a PR (adapter branches on `gitHost.platform`
    per `.claude/tracker.json` — GitLab MR, GitHub PR). Title: same as commit. Body: same as
    commit body, plus a `## Test plan` checklist with two boxes: "walk each updated skill's
    `Steps` and confirm nothing project-specific was silently overwritten", "if any SKIP-list
    drift is called out above, decide whether to open a follow-up manual merge".

## Exclude list (project-owned; never overwrite)

Paths on this list are skipped entirely. The skill treats them as project-owned; the template's
version is never copied down.

- `.claude/tracker.json` — project's tracker/gitHost/comms config.
- `.claude/settings.local.json` — gitignored per-machine settings.
- `.mcp.json` — project's MCP server config (URLs, tokens).
- `.env`, `.env.example` — project's env config.
- `docs/architecture/**` — project's target solution architecture.
- `docs/repos/**` — project's per-repo navigation indexes.
- `docs/superpowers/**` — feature specs and plans; both template and project write here, safer
  to leave the project's copies untouched.
- `docs/<feature>/**` — legacy per-feature spec dirs (pre-MR !30); still excluded for projects
  that haven't migrated.
- `ROADMAP.md` — project's living sprint plan.
- `workspace/**` — gitignored service-repo checkouts.

## SKIP list (drift reported, not copied)

Paths on this list have a mix of template-owned and project-owned content. The skill does not
overwrite them, but it does surface their drift in the PR body so the human can reconcile by
hand.

- `CLAUDE.md` — template owns the process rules; project owns the "Project context" fill-in
  and any project-added rules.
- `README.md` — template owns the layout/process narrative; project owns the title line.
- `.gitignore` — handled specially in step 8 (union merge, not overwrite).

## Config

Reads `.claude/tracker.json` → `gitHost` in step 10 to pick the PR adapter (GitLab MR / GitHub
PR). Does not read `tracker` or `comms`. No writes outside the sync branch it opens.
