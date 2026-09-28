---
name: split-issue
description: Reactive tracker-only issue splitting — decomposes a tracker issue into sub-issues on demand (dev sees their PR is too big; reviewer wants smaller diffs). Reads issue description + optional PR link for scope grounding, auto-proposes sub-issues, refines via Q&A, creates them in the tracker under the parent. Also the tracker writer for /split-pr, which hands it an already-approved partition via --groups. Use only when explicitly invoked as /split-issue or when /split-pr delegates to it — never start it on your own initiative. Never writes to docs/.
---

# /split-issue `<issue-id> [--pr <pr-url>] [--refine] [--groups <path>] [--dry-run]`

## Purpose

Split a tracker issue into sub-issues when its scope is too large for a reviewable PR. Triggered
by anyone (dev mid-implementation, reviewer during review, PL). **Tracker-only**: never writes to
`docs/`, never touches the modularity tree, never creates arch-log entries. This respects the
`docs → tracker` one-way direction rule in `docs/process/AGENTS.md`.

For **proactive** planning-time decomposition (create L3 tree nodes for a feature *before*
starting), use `/modularize <l2-slug>` instead — that is the docs-side tool that writes to the
tree.

**When L3 tree nodes already exist** for the feature's L2 module — because someone ran
`/modularize <l2-slug>` earlier — the user may hand those paths in as scope-grounding context
during a planning conversation (e.g. "the tree under `docs/architecture/current/modularity/<...>/<l2-slug>/`
has these three L3 subdirs; base your proposals on them"). The skill uses that context in
step 5's auto-propose, exactly the same way it uses `--pr` file lists. No new flag, no
auto-detection — the user opts in by providing the paths. When `/split-pr` needs a
machine-consumable handoff, `--groups` remains the mechanism.

This skill is also **the tracker writer for `/split-pr`**. `/split-pr` partitions a diff, gets the
division approved as a table, and then calls `/split-issue <parent> --groups <file>` to materialize
one sub-issue per PR in the stack. Sub-issue creation lives here and only here, so the tracker has a
single writer.

## Inputs

- `<issue-id>` (required) — tracker-native id. Plane work-item id / GitHub `#NN` (or full URL) /
  GitLab `#NN` (or full URL).
- `--pr <pr-url>` (optional) — link to a draft/open PR. If provided, the skill reads the PR's
  file-list and summary via the git-host client as extra scope grounding.
- `--refine` (optional flag) — operate on the issue's existing sub-issues (add / rename / drop)
  instead of creating fresh ones.
- `--groups <path>` (optional) — path to a JSON array of already-approved groups, normally written
  by `/split-pr`. Creates exactly those sub-issues, in that order, with no proposal and no Q&A —
  see **`--groups` mode**. Mutually exclusive with `--refine`; refuse with
  `--groups and --refine cannot be combined` when both are passed.
- `--dry-run` (optional flag) — print the sub-issues that would be created, one line each, and
  exit. **Makes no tracker call of any kind.** Valid in every mode; in `--refine` it prints the
  operation queue instead of applying it.

## Steps

### Normal mode (no `--refine`)

1. **Resolve tracker platform** from `.claude/tracker.json` → `tracker.platform`.
2. **Fetch the issue** via the tracker adapter — title, description, existing sub-issues (if any).
3. **Refuse-with-hint on existing sub-issues.** If the issue already has sub-issues, print
   `use --refine to modify existing sub-issues` and exit.
4. **Fetch PR context** — if `--pr` was passed, fetch the PR's file list and summary through
   whatever client this environment provides for `gitHost.platform`: an MCP server, a CLI (`glab`,
   `gh`, …), or the REST API. Check it targets `gitHost.apiUrl` — a client bound to the platform's
   public instance cannot see a self-hosted project, and fails as though the project were missing.
   On failure, warn and continue without PR grounding.
5. **Auto-propose N sub-issues.** Based on issue description + optional PR context, propose N
   sub-issues with `{title, summary}`. Each should be sized for a small-diff PR.
6. **Q&A refine, one at a time.** For each proposal, prompt keep / edit / drop. Edits prompt only
   for `title`, `summary`.
7. **Add-more loop.** After the last proposal, prompt "any missing?". If yes, mini Q&A per new
   sub-issue.
8. **Confirm parent.** Default parent is `<issue-id>`. User can override to a different tracker
   issue as parent.
9. **Materialize sub-issues.** Under `--dry-run`, print one line per sub-issue —
   `would create under <parent>: <title>` — and exit here, having called nothing. Otherwise one
   tracker call per sub-issue via the adapter, **issued one at a time, in list order**. Trackers
   number items in the order they arrive, so creating them concurrently scrambles the board: the
   sub-issues read top-to-bottom in an order that has nothing to do with the order they are meant to
   be worked in. Sequential creation costs a few seconds and makes the numbering mean something.
   - **Plane** — create a work item with `parent = <resolved-parent>`, using the parent's
     work-item type.
   - **GitHub** — create an issue in the same repo, attach as sub-issue via the `/sub_issues` API,
     inherit milestone from parent when set.
   - **GitLab** — create an issue in the same project, link to parent via the linked-issue
     relation (`relates_to`), inherit milestone from parent when set.

   **Inherit priority from the parent** wherever the tracker has one; a child created at the
   tracker's default drops the urgency the parent carried, and nobody goes back to set it on seven
   items by hand. **Do not inherit state.** A fresh sub-issue belongs in the project's default entry
   state even when the parent is already in progress — the parent's state describes the parent's
   work, not the child's.

   Each sub-issue's description contains a link back to the parent issue and the Q&A summary.
10. **Print summary** — list of created sub-issue ids + URLs.

### `--groups` mode

The caller has already produced the partition and already had a human approve it. Do not re-propose,
do not re-ask, do not reorder.

1. **Read the handoff file** — a JSON array in the intended order, each element
   `{title, summary, paths}`. `paths` may be empty; `title` and `summary` may not. Refuse with
   `--groups file is not a non-empty array of {title, summary}` on anything else, and read it once:
   it is an input, not a place to write results back to.
2. **Resolve platform and fetch the parent** as in normal-mode steps 1–2.
3. **Do not refuse on existing sub-issues.** Normal-mode step 3's guard does not apply here — the
   caller has already shown them to the user and been told to create a new set anyway. Note in the
   summary how many already existed and that they were left untouched.
4. **Materialize** as normal-mode step 9, honouring `--dry-run` the same way: one line per
   sub-issue, no tracker call. Each description contains a link back to the parent, the group's
   `summary`, and its `paths` as a list — the path list is what makes the sub-issue legible next to
   the MR that implements it.
5. **Print summary** — created sub-issue ids and URLs **in the input array's order**, so the caller
   can map them back onto its groups by position. If any creation failed, say which index failed and
   do not renumber the rest; the caller checks the count before it opens anything.

### `--refine` mode

1. **Resolve platform + fetch the parent issue and its existing sub-issues** as in normal mode
   steps 1–2.
2. **List existing sub-issues** as a numbered menu.
3. **Prompt for an operation** — `add | rename <n> | drop <n> | done`. Loop until `done`.
   - **add** — same mini-Q&A as normal mode step 7; create in tracker under the parent.
   - **rename <n>** — prompt for new title/summary; update via tracker adapter. Never touches
     `docs/`.
   - **drop <n>** — close the sub-issue in the tracker with reason `dropped during /split-issue
     --refine`. Do not delete the tracker item — closure preserves audit history.

   Under `--dry-run`, collect the operations and print the queue at `done` instead of applying any
   of them.

## Config

Reads `.claude/tracker.json` → `tracker.platform` (for the tracker adapter) and `gitHost.platform`
and `gitHost.apiUrl` (which pick and validate the host client used when `--pr` is passed — see
normal-mode step 4). **Writes only to the tracker and to nothing under `docs/`.** Enforcing this is
the design principle from
`docs/superpowers/specs/2026-08-12-reactive-issue-splitting-design.md` — do not reintroduce any
promote-to-tree flow.

**`--dry-run` is a hard guarantee, not a courtesy.** It exists because `/split-pr` chains into this
skill: someone dry-running a split of an open PR must not find sub-issues waiting for them
afterwards. No mode may reach a tracker write with the flag set.
