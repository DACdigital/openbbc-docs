---
name: sdd-rebase
description: Sync template-generic updates from dac-docs-template into this project docs repo. Fetches the upstream template, 3-way merges what the template changed since the last recorded template base into the project (skills, agents, process docs, conventions), never touches project-only paths, skips project-specific paths (tracker.json, .mcp.json, docs/architecture/, per-feature specs, ROADMAP.md, filled-in CLAUDE.md/README.md), removes leftover copies of the template's own design specs and plans, and opens a PR with the delta. Never runs in the template repo itself.
disable-model-invocation: true
---

# /sdd-rebase

## Purpose

Automate what today happens as a hand-copied `chore: sync template updates from dac-docs-template!X`
commit: pull template-generic improvements (new skills, updated process rules, fixed conventions)
from the upstream `dac-docs-template` and open a PR against the current project docs repo. The PR
is the safety net — a human reviews the diff before merging.

The skill uses an **exclude list** rather than an include list: it merges almost everything the
template changed and skips a short, stable list of paths that belong to the project. New template
files propagate automatically. Project-owned paths, project-only files, and project edits to
template files are never clobbered (see [Model](#model)).

## Inputs

- None required. Optional `--dry-run` prints the sync plan and exits without branching, staging,
  or pushing.

## Model

A sync is a 3-way merge, not a copy. Three versions of each path take part:

- **project**: the file at `HEAD` of this repo.
- **base**: the file at the template commit the last sync merged (the *template base*).
- **template**: the file at `template/master`.

The delta is what the template changed between base and `template/master`. How `HEAD` differs
from `template/master` does not matter: that difference includes every project addition, and
copying over it would wipe project work.

Rules that follow from this:

- A path is updated only if the template changed it since base.
- A path is deleted only if the template deleted it since base.
- A path that was never in the template (not in base, not in `template/master`) is project-only.
  The sync never touches it. Project-added skills, docs and scripts are expected and safe.
- A project change is never overwritten silently. Both sides changed means a 3-way merge, and a
  merge that cannot keep both intents is left with conflict markers for a human.

## Steps

1. **Refuse if run in the template repo itself.** Read `origin`'s URL. If it ends with
   `/dac-docs-template.git` (or matches the canonical template URL below), print
   `refusing to sync-from-template in the template repo itself` and exit. This is a
   downstream-only command.

2. **Ensure the `template` git remote exists.** Run `git remote get-url template`. If it fails,
   add it from the canonical URL. Prompt the user first:

   ```
   No `template` remote configured. Add it pointing at
   git@gitlab.dac.digital:dacdigital/tech-pulse/spec-driven-development/dac-docs-template.git?
   [Y/n] (or paste a different URL)
   ```

   Default Y. On Y, `git remote add template <url>`. On a custom URL, use that instead.

3. **Fetch the template.** `git fetch template` pulls `template/master` and every template
   branch. Feature branches matter: a past sync may have taken files from one.

4. **Use the latest copy of this skill.** Compare the local `.claude/skills/sdd-rebase/SKILL.md`
   with `git show template/master:.claude/skills/sdd-rebase/SKILL.md`. If they differ, follow the
   template's copy for this run. An old local copy may still overwrite instead of merge.

5. **Find the template base.**
   - **Recorded.** Read it from the newest sync commit that has one:

     ```
     git log -1 --grep='^template-base: ' --format=%B \
       | sed -n 's/^template-base: \([0-9a-f]\{40\}\)$/\1/p' | tail -1
     ```

     Check that the sha exists locally (`git cat-file -e <sha>^{commit}`). It may sit on a
     template feature branch and not on `template/master`. That is fine, use it as is.
   - **Reconstructed.** If no base is recorded, or the recorded sha is missing, rebuild it by
     content. Candidates are the commits on every template branch
     (`git rev-list --remotes=template`), newest first. For each candidate, count the paths
     outside the exclude and SKIP lists where the blob at the candidate equals the blob at `HEAD`
     (`git ls-tree -r` on both, compare object ids). The candidate with the highest count wins.
     Ties go to the newest. Show the top three with their counts and ask the user to confirm
     before going on. Mark the base as *reconstructed* for the PR body (step 13).
   - **None.** If no candidate shares a single file with `HEAD`, stop and ask the user for a base.
     Do not fall back to a plain copy.

6. **Compute the delta.** Run

   ```
   git diff --name-status --no-renames <base> template/master -- . <exclude-args>
   ```

   where `<exclude-args>` are `:(exclude)` pathspecs for the exclude list and the SKIP list. A
   rename shows as a delete plus an add. If the delta is empty and step 11 finds no template
   docs to remove, print `already up-to-date with template/master` and exit.

   Classify each delta path against `HEAD`:

   | Template change | State at `HEAD` | Class | Action |
   |---|---|---|---|
   | modified | equals base | clean update | take the template version |
   | modified | equals template | already in | nothing |
   | modified | differs from both | both changed | 3-way merge (step 9) |
   | modified | missing | project deleted | keep it deleted, list under Conflicts |
   | added | missing | template addition | add the template version |
   | added | equals template | already in | nothing |
   | added | differs | both added | 3-way merge with an empty base, list under Conflicts |
   | deleted | equals base | template deletion | `git rm` |
   | deleted | missing | already gone | nothing |
   | deleted | differs from base | project changed, template deleted | keep the file, list under Conflicts |

   Also list the **project-only paths**: tracked at `HEAD`, absent from both base and
   `template/master`, and not on the exclude list. These are reported only.

7. **On `--dry-run`, print the plan and exit.** One section each, with path counts:
   - merged cleanly (clean updates, plus both-changed paths whose trial merge has no conflict),
   - conflicts (trial merge with conflicts, and every row above marked "list under Conflicts"),
   - template additions,
   - template deletions,
   - template docs to remove (step 11), and template docs kept because the project changed them,
   - project-only paths left alone,
   - drift on the SKIP list (`CLAUDE.md`, `README.md`, `.gitignore`), as a diff against
     `template/master`, so the human knows what to reconcile by hand.

   Also print the base sha and whether it was recorded or reconstructed. For the trial merge,
   run `git merge-file -p` into a temp file so the working tree stays untouched.

8. **Otherwise, create the sync branch.** Name it `sync/from-template-YYYY-MM-DD` (append `-<n>`
   if a same-day branch already exists). Branch off the current default branch.

9. **Apply the delta.** Work only on the paths from step 6. Never touch the exclude list, the
   SKIP list, or project-only paths.
   - Clean updates and template additions: `git show template/master:<path> > <path>`.
   - Template deletions: `git rm <path>`.
   - Both changed: write base and template to temp files, then merge into the project copy:

     ```
     tmp=$(mktemp -d)
     git show <base>:<path> > "$tmp/base"            # empty file for "both added"
     git show template/master:<path> > "$tmp/template"
     git merge-file --diff3 -L project -L template-base -L template \
       <path> "$tmp/base" "$tmp/template"
     ```

     Exit code 0 means a clean merge. A positive code is the number of conflict hunks.
   - Conflict hunks: read each one and resolve it by keeping both intents where they do not
     contradict. A typical case: the template added one rule and the project added another, so
     keep both. If the intents contradict, leave the markers in the file. Either way, list the
     path under Conflicts with its outcome (`resolved, both kept` or `markers left`) and one line
     on what each side changed.
   - Binary files that both sides changed: keep the project version and list it under Conflicts.

10. **Merge `.gitignore` as a union.** Do not overwrite. Add lines present in
    `template/master:.gitignore` that are missing locally. Keep local-only lines.

11. **Remove template docs.** Projects set up before `/setup-project` dropped the template's
    design specs and plans still carry them in `docs/superpowers/`, frozen there because that path
    is excluded. Find them by content, not by name, so the project's own specs are never touched:
    a tracked file under `docs/superpowers/` (except `.gitkeep`) is a template doc when its blob
    equals the blob at the same path in `<base>` or in `template/master`:

    ```
    git ls-tree -r HEAD -- docs/superpowers/   # compare object ids per path with
    git ls-tree -r <base> -- docs/superpowers/ # these two; a match on either is a template doc
    git ls-tree -r template/master -- docs/superpowers/
    ```

    `git rm` every match. A path that exists in the template but differs at `HEAD` was edited by
    the project: keep it and list it under Conflicts as `template doc, changed in project`. Leave
    arch-log entries that link a removed file as they are — logs are append-only and keep the link
    as history. Nothing is ever added to `docs/superpowers/`. The sync PR is the review step.

12. **Run the post-sync report.** See [Post-sync report](#post-sync-report-client-resolution).

13. **Commit.** One commit. Title: `sync: template updates from dac-docs-template (<N> files)`.
    The body has these parts:
    - `template-base: <full sha>` as a trailer at the end of the body. The sha is the
      `template/master` commit this sync merged. The next sync uses it as its base.
    - `## Template base`: the previous base sha, and how it was found (recorded, or
      reconstructed with the match count). A reconstructed base needs a reviewer's check.
    - The source range `<previous-base-short>..template/master@<short-sha>`.
    - Lists of paths merged cleanly, added, and deleted.
    - `## Template docs removed`: the paths step 11 removed. Omit when none.
    - `## Conflicts`: every path from step 9 with its outcome. Paths with `markers left` come
      first. Omit the section when there are none.
    - `## Drift on SKIP list (review manually)`: a diff of `CLAUDE.md`, `README.md`, and any
      other SKIP-list file that diverged from `template/master`.
    - The `## Client resolution` section from step 12 when it has findings.

    Project-only paths are not listed in the commit. Their count is enough.

14. **Push and open a PR.** Push the branch. Open a PR with the `gitHost` client per
    [Client resolution](../../../docs/process/README.md#client-resolution) (GitLab MR, GitHub
    PR). Title: same as the commit. Body: same as the commit body, including the
    `template-base:` line, plus a `## Test plan` checklist:
    - "resolve every path under Conflicts marked `markers left`, then check the ones marked
      `resolved, both kept`",
    - "walk each updated skill's `Steps` and confirm no project rule was lost",
    - "if the template base was reconstructed, confirm it",
    - "check that every path under Template docs removed is a template design doc, not a project
      spec",
    - "if any SKIP-list drift is called out above, decide whether to open a follow-up manual
      merge".

## Exclude list (project-owned; never overwrite)

Paths on this list are skipped entirely. The skill treats them as project-owned; the template's
version is never copied down.

- `.claude/tracker.json` — project's tracker/gitHost/comms config.
- `.claude/settings.local.json` — gitignored per-machine settings.
- `.mcp.json` — project's MCP server config (committed; secrets only as `${VAR}`). Reported on by the
  post-sync report below, never edited.
- `.env` — gitignored, no longer read by anything; left alone and reported on. `.env.example` is
  **not** excluded: it is template-owned, so its deletion syncs down.
- `docs/architecture/**` — project's target solution architecture.
- `docs/repos/**` — project's per-repo navigation indexes.
- `docs/superpowers/**` — feature specs and plans. In the template these are the template's own
  design history, which `/setup-project` removes from new projects and step 11 removes from older
  ones; in a project they are its feature specs. Nothing is ever copied down, so a removed template
  spec never comes back.
- `docs/superpowers/plans/**`: implementation plans are never committed in a project (the
  template's few historical plans are removed by `/setup-project` or step 11). Covered by the line above,
  listed so it stays excluded if that line ever narrows.
- `docs/<feature>/**` — legacy per-feature spec dirs (pre-MR !30); still excluded for projects
  that haven't migrated.
- `ROADMAP.md` — project's living sprint plan.
- `workspace/**` — gitignored service-repo checkouts.

## Post-sync report (client resolution)

Runs on every sync after step 11, and is idempotent: it only inspects state and writes nothing but a
`## Client resolution` section in the commit and PR body. Omit the section when every item is clean.
See [Client resolution](../../../docs/process/README.md#client-resolution) for the model these checks
enforce.

- **`.mcp.json` contents** (project-owned — report, never edit). Flag:
  - git-host blocks (GitHub / GitLab) — now redundant: the git host resolves through `gh` / `glab` or
    the user's own MCP server;
  - non-secret `${VAR}` values (`PLANE_BASE_URL`, `PLANE_WORKSPACE_SLUG`) that `tracker.json` can
    supply as literals;
  - any literal-looking secret — it is committed and must be rotated.
- **Token history.** If `git log -p -- .mcp.json` matches a token pattern (`plane_`, `glpat-`, `ghp_`,
  `github_pat_`), report **only the commit SHA and the token type — never the matched text**, and say
  the token must be rotated. Rotation is the remedy; scrubbing history is the maintainer's call.
- **`.env`.** If it exists locally: "`.env` is no longer read; move its values into your shell
  environment (or a tool such as direnv)."
- **`tracker.json`** without `instanceUrl` for `tracker` / `gitHost`: "re-run `/setup-project`
  (Phase 0) to add instance URLs".

A project's first sync after this change runs the project's **old** copy of this skill, which still
excludes `.env.example` and has no post-sync report. The migration therefore completes on the
**next** sync, run by the new copy. Until then `.env.example` lingers; that is harmless, since
nothing reads it.

## SKIP list (drift reported, not copied)

Paths on this list have a mix of template-owned and project-owned content. The skill does not
overwrite them, but it does surface their drift in the PR body so the human can reconcile by
hand.

- `CLAUDE.md` — template owns the process rules; project owns the "Project context" fill-in
  and any project-added rules.
- `README.md` — template owns the layout/process narrative; project owns the title line.
- `.gitignore` — handled specially in step 10 (union merge, not overwrite).

## Config

Reads `.claude/tracker.json` → `gitHost` in step 14, via
[Client resolution](../../../docs/process/README.md#client-resolution), to open the PR; reads
`tracker` / `gitHost` `instanceUrl` presence for the post-sync report. Does not read `comms`. No
writes outside the sync branch it opens (plus its own `clients.local.json` entry).
