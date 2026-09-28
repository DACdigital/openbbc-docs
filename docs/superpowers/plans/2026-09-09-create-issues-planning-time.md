# /create-issues as planning-time issue creator — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Reshape `/create-issues` from a spec-derived parent+children decomposer into a single-purpose planning-time command; wire `/plan-sprint` to invoke it via a bulk-create prompt; document that L3 tree paths are usable scope-grounding for `/split-issue`; move `/create-issues` into the planning section of the per-feature flow diagram.

**Architecture:** Docs-only. Four files touched — `.claude/skills/create-issues/SKILL.md`, `.claude/skills/plan-sprint/SKILL.md`, `.claude/skills/split-issue/SKILL.md`, `docs/process/README.md`. No code, no tests. Verification is reading + grepping. Spec: `docs/superpowers/specs/2026-09-09-create-issues-planning-time-design.md`.

**Tech Stack:** Markdown, YAML front-matter.

**Commit policy:** Per standing user instruction, do not commit the spec or plan into the project docs repo. In the template repo this plan lives at `docs/superpowers/plans/2026-09-09-create-issues-planning-time.md` and IS committed (matches the template's own pattern of shipping shaping-doc history). Implementation edits may be committed as normal work.

---

## File structure

- `.claude/skills/create-issues/SKILL.md` — full rewrite of Purpose + Steps; new frontmatter description. Shrinks from ~65 lines to ~40.
- `.claude/skills/plan-sprint/SKILL.md` — insert one new step between current step 7 and step 8; update step 8 wording; extend frontmatter description.
- `.claude/skills/split-issue/SKILL.md` — insert one paragraph after the existing "For proactive planning-time decomposition..." block. Zero behavior/flag changes.
- `docs/process/README.md` — reorder the per-feature flow diagram; update the ROADMAP-column-owner sentence in the Sprint-driven ROADMAP growth section.

Each task below is one edit + one verification. Steps 5-second verification is a `grep`/`sed`/`cat` on the exact fragment.

---

## Task 1: Rewrite `create-issues/SKILL.md`

**Files:**
- Modify: `.claude/skills/create-issues/SKILL.md` (full body replacement of Purpose + Inputs + Steps + Config)

- [ ] **Step 1: Read the current file to confirm baseline.**

  Run: `cat .claude/skills/create-issues/SKILL.md`

  Expected: file starts with `name: create-issues`, contains the current 6-step flow with the spec-merged gate (step 1), spec-reading (step 2), and L3 sub-issue prompt (step 5).

- [ ] **Step 2: Overwrite the file with the new body.**

  Replace the entire file contents with:

  ````markdown
  ---
  name: create-issues
  description: Create a tracker issue for a feature discussed during planning. One call per feature, one tracker issue created per call. No spec required. Sub-issues, if ever needed, are created explicitly via /split-issue — never here.
  disable-model-invocation: true
  ---

  # /create-issues `<name>`

  ## Purpose

  Create the tracker issue for a feature discussed during planning. Called during (or shortly
  after) `/plan-sprint` — one call per new `ROADMAP.md` row. The issue stands alone: no children,
  no assumptions about hierarchy. If the scope later needs breaking down, that's `/split-issue`'s
  job, invoked explicitly.

  This command does **not** read specs, plans, or the modularity tree. Everything it needs is on
  the ROADMAP row. The old post-spec-merge decomposition flow is gone — see
  `docs/superpowers/specs/2026-09-09-create-issues-planning-time-design.md` for the reshape.

  ## Inputs

  - `<name>` (required) — the feature name; must match a `Feature` value in `ROADMAP.md`. Fails
    if no such row exists.

  ## Steps

  1. **Resolve the ROADMAP row** by `<name>`. If no row matches, exit with
     `no ROADMAP row for <name> — run /plan-sprint first to append it`. Read the row's
     `Feature`, `Priority`, `Level`, `Module`, `Milestone` columns.
  2. **Determine target location** for the new issue:
     - `.claude/tracker.json` → `tracker.platform` picks the adapter.
     - For **plane**, the location is `tracker.projectId` — no repo selection needed.
     - For **github** and **gitlab**, the issue goes in a service repo. Read `workspace/INDEX.md`
       for the list of repos. If there is exactly one repo, use it. If there are multiple, prompt
       the user to pick one (default: the repo whose name matches the row's `Module` slug when
       one matches, otherwise no default). Do not attempt to infer from module front-matter —
       the modularity tree does not carry a `home_repo` field today.
  3. **Create one tracker issue** via the adapter:
     - **Plane** — create a work item under `tracker.projectId` with `title = <feature>`, priority
       mapped from the row, and no parent (this is a top-level item). Assign to the milestone/cycle
       whose name matches the row's `Milestone` column, creating it if missing.
     - **GitHub** — create an issue in the resolved service repo with `title = <feature>`. Assign
       to the milestone whose name matches the row's `Milestone`, creating it if missing.
     - **GitLab** — create an issue in the resolved service repo with `title = <feature>`. Assign
       to the milestone whose name matches the row's `Milestone`, creating it if missing.

     The issue description is a two-line stub: `Feature: <name>` on line 1, `Planned in
     <sprint-name>` on line 2. No spec link, no scope block — those come later if a spec is
     written.
  4. **Update `ROADMAP.md`.** Fill the row's `Issues` column with the created issue's link;
     leave `Link` blank (there is no epic/tracking-issue at this stage — `/roadmap-sync`
     manages that column during planning if a team uses epics).
  5. **Print one-line summary** — `created <issue-id> at <url>` — and exit.

  ## What this command does NOT do

  - Read specs, plans, or the modularity tree.
  - Create sub-issues, child work items, or L3-derived shards. That is `/split-issue`'s job,
    invoked explicitly when needed.
  - Suggest `/roadmap-sync` on completion. Per `docs/process/AGENTS.md`, that command is
    planning-only and callers should not chain into it.
  - Run any pre-flight gate other than "the ROADMAP row exists."

  ## Config

  Reads `.claude/tracker.json` → `tracker` (platform, org, projectId) to select the adapter in
  step 3. Also reads `gitHost` when `tracker.platform` is `github` or `gitlab`, to know which org
  the target service repo lives in. Reads `workspace/INDEX.md` to list candidate repos in step 2.
  ````

- [ ] **Step 3: Verify the new file.**

  Run: `grep -c 'spec' .claude/skills/create-issues/SKILL.md`

  Expected: `2` or fewer — only the "This command does **not**... Read specs" line and the
  spec-file reference at the top mentioning the reshape spec. No "spec PR merged" or "read the
  spec's Scope/Contracts" content.

  Run: `grep -c 'L3\|sub-issue' .claude/skills/create-issues/SKILL.md`

  Expected: `0` for `L3` (this command no longer knows about L3) and `1` for `sub-issue` (the
  "no sub-issues, that's /split-issue" callout).

- [ ] **Step 4: Commit.**

  ```bash
  git add .claude/skills/create-issues/SKILL.md
  git commit -m "create-issues: reshape to planning-time issue creator (per feature, no spec, no children)"
  ```

---

## Task 2: Insert bulk-create prompt into `plan-sprint/SKILL.md`

**Files:**
- Modify: `.claude/skills/plan-sprint/SKILL.md` (frontmatter description + insert step 8 + renumber old step 8 to step 9 + update wording of old step 8)

- [ ] **Step 1: Read the current file to confirm baseline.**

  Run: `cat .claude/skills/plan-sprint/SKILL.md`

  Expected: file's normal-mode has 8 steps, last of which is "Print next-steps summary — one line per row that needs a spec: `next: /new-spec <slug>`."

- [ ] **Step 2: Update the frontmatter description.**

  Change the description from:

  > `Interactive sprint planning — reads the modularity tree, shows candidate scope (unexpanded L1/L2 placeholders + in-flight items), asks DM to pick sprint contents (from tree + ad-hoc), creates the tracker milestone, and appends rows to ROADMAP.md. Grows the plan one sprint at a time. Supports --refine on the currently active sprint.`

  To:

  > `Interactive sprint planning — reads the modularity tree, shows candidate scope (unexpanded L1/L2 placeholders + in-flight items), asks DM to pick sprint contents (from tree + ad-hoc), creates the tracker milestone, appends rows to ROADMAP.md, and prompts to bulk-create tracker issues via /create-issues. Grows the plan one sprint at a time. Supports --refine on the currently active sprint.`

- [ ] **Step 3: Insert new step 8 between current steps 7 and 8.**

  In the "Normal mode" section, immediately after step 7 (the append-to-ROADMAP step ending with
  `... Spec, Issues, Link start empty and are filled later by /new-spec, /create-issues, /roadmap-sync.`) and before the current step 8 (the "Print next-steps summary" step), insert this
  new step:

  ```markdown
  8. **Prompt to bulk-create tracker issues.** After the rows are appended, prompt
     `Create tracker issues for these N features now? [Y/n]` (default `Y`). On `Y`, invoke
     `/create-issues <name>` once per newly appended row, in the order they were appended, one
     at a time — tracker item numbering depends on serialized creation, and parallelizing here
     scrambles the board. Print each command's one-line summary as it completes. On `n`, skip
     — the user can run `/create-issues <name>` per row later. If any invocation fails
     mid-loop, stop, print the failure and which rows still lack issues, and exit; the DM
     re-runs `/create-issues` per gap.
  ```

- [ ] **Step 4: Renumber the old step 8 → step 9 and update its wording.**

  Change the old step 8 (now step 9) from:

  ```markdown
  8. **Print next-steps summary** — one line per row that needs a spec:
     `next: /new-spec <slug>`.
  ```

  To:

  ```markdown
  9. **Print next-steps summary.** For each row: `next: /new-spec <slug>` if a spec is needed
     (level C1 or C2), or `next: implement <slug>` if the row is C0 and already ticketed. If
     step 8 was skipped, prepend `next: /create-issues <slug>` before the /new-spec line.
  ```

- [ ] **Step 5: Verify.**

  Run: `grep -c '^[0-9]\+\.' .claude/skills/plan-sprint/SKILL.md`

  Expected: normal-mode section has 9 numbered steps (was 8). `--refine` mode still has its 3.

  Run: `grep -n 'Prompt to bulk-create tracker issues' .claude/skills/plan-sprint/SKILL.md`

  Expected: one match near step 8 in the normal-mode block.

- [ ] **Step 6: Commit.**

  ```bash
  git add .claude/skills/plan-sprint/SKILL.md
  git commit -m "plan-sprint: bulk-create tracker issues via /create-issues at end of run"
  ```

---

## Task 3: Add L3-context paragraph to `split-issue/SKILL.md`

**Files:**
- Modify: `.claude/skills/split-issue/SKILL.md` (insert one paragraph in Purpose)

- [ ] **Step 1: Read the current file to confirm baseline.**

  Run: `sed -n '9,22p' .claude/skills/split-issue/SKILL.md`

  Expected: shows the Purpose section including "For **proactive** planning-time decomposition..." block ending around line 17.

- [ ] **Step 2: Insert the new paragraph immediately after the "For proactive planning-time decomposition" block.**

  Find the block:

  ```markdown
  For **proactive** planning-time decomposition (create L3 tree nodes for a feature *before*
  starting), use `/modularize <l2-slug>` instead — that is the docs-side tool that writes to the
  tree.
  ```

  Immediately after it (before the "This skill is also **the tracker writer for `/split-pr`**"
  block), insert:

  ```markdown
  **When L3 tree nodes already exist** for the feature's L2 module — because someone ran
  `/modularize <l2-slug>` earlier — the user may hand those paths in as scope-grounding context
  during a planning conversation (e.g. "the tree under `docs/architecture/current/modularity/<...>/<l2-slug>/`
  has these three L3 subdirs; base your proposals on them"). The skill uses that context in
  step 5's auto-propose, exactly the same way it uses `--pr` file lists. No new flag, no
  auto-detection — the user opts in by providing the paths. When `/split-pr` needs a
  machine-consumable handoff, `--groups` remains the mechanism.
  ```

- [ ] **Step 3: Verify no other section changed.**

  Run: `git diff --stat .claude/skills/split-issue/SKILL.md`

  Expected: exactly one file changed, one insertion, zero deletions (or one deletion for the
  blank line that shifted).

  Run: `grep -c 'When L3 tree nodes already exist' .claude/skills/split-issue/SKILL.md`

  Expected: `1`.

  Run: `grep -c 'auto-detection' .claude/skills/split-issue/SKILL.md`

  Expected: `1` — confirms the "no auto-detection" wording made it in.

- [ ] **Step 4: Commit.**

  ```bash
  git add .claude/skills/split-issue/SKILL.md
  git commit -m "split-issue: document L3 tree paths as valid scope-grounding context (user-provided, not auto-detected)"
  ```

---

## Task 4: Reorder per-feature flow diagram in `docs/process/README.md`

**Files:**
- Modify: `docs/process/README.md` (per-feature flow block around lines 140-157; ROADMAP-column-owner sentence in Sprint-driven ROADMAP growth section)

- [ ] **Step 1: Read the current per-feature flow block.**

  Run: `sed -n '140,160p' docs/process/README.md`

  Expected: shows the ASCII flow with `/new-feature` → `/new-spec` → `/create-issues` →
  `/roadmap-sync` → devs implement chain, with `/split-issue` mentioned as an inline hint under
  the devs step.

- [ ] **Step 2: Replace the per-feature flow block.**

  Find the block starting with `select feature (from ROADMAP or ad-hoc)` and ending with
  `all child issues close → feature done → ROADMAP updated`. Replace with:

  ```
  select feature (from ROADMAP or ad-hoc during /plan-sprint)
    │
    ├─ /create-issues <name>    → one tracker issue per feature (planning-time; called by
    │                            /plan-sprint's Y/n prompt, or standalone per row)
    │
    ├─ if capability not yet in solution architecture:
    │    /new-feature   → docs/architecture/current/** + docs/architecture/logs/…
    │                   → arch PR → /arch-log-review → human merge  ← GATE
    │
    ├─ /new-spec        → docs/superpowers/specs/YYYY-MM-DD-<name>-design.md
    │                   → spec PR → /spec-review (unified: structure + naming + arch + contracts)
    │                   → human merge  ← GATE
    │                   → /arch-review <feature>  (optional, post-approval — sync current/)
    │
    ├─ Devs implement in workspace/<repo>, code PRs reviewed & merged
    │  ↳ /split-issue <id> if a dev/reviewer wants smaller PRs mid-flight (tracker-only)
    │
    └─ all child issues close → feature done → ROADMAP updated
  ```

  Note the changes: `/create-issues` moves to the top (planning-time); the old `/roadmap-sync`
  bullet is dropped from this flow (it's planning-only, not per-feature — already covered by
  the AGENTS.md rule).

- [ ] **Step 3: Update the ROADMAP-column-owner sentence in the Sprint-driven ROADMAP growth section.**

  Find the sentence in `## Sprint-driven ROADMAP growth` (near line 128-138) that describes what
  fills ROADMAP columns. Currently reads something like `... start empty and are filled later by
  /new-spec, /create-issues, /roadmap-sync`. Change to:

  > `... start empty. /create-issues fills the Issues column at planning time; /new-spec sets Spec at spec-write time; /roadmap-sync reconciles Link + Milestone during planning refresh.`

  If the sentence is worded differently in the current file (edit-tolerant): find the mention of
  "filled later by /new-spec, /create-issues, /roadmap-sync" (or similar) and replace with the
  above.

- [ ] **Step 4: Verify.**

  Run: `grep -n '/create-issues' docs/process/README.md`

  Expected: appears in the flow diagram at the planning position (top of flow), and in the
  ROADMAP-columns sentence. No occurrence after `/new-spec` in the flow diagram.

  Run: `grep -n '/roadmap-sync' docs/process/README.md`

  Expected: no longer appears in the per-feature flow diagram. Still appears in the
  Tracker-adapters paragraph (line ~191) and in the ROADMAP-columns sentence — those are the
  reference mentions.

- [ ] **Step 5: Commit.**

  ```bash
  git add docs/process/README.md
  git commit -m "process: reorder per-feature flow — /create-issues moves to planning; /roadmap-sync out of the diagram"
  ```

---

## Task 5: Cross-file verification pass

**Files:** none modified in this task — read-only checks.

- [ ] **Step 1: Search for stale post-spec `/create-issues` references.**

  Run:

  ```bash
  grep -rn '/create-issues' .claude/ docs/process/ CLAUDE.md README.md AGENTS.md
  ```

  Expected: every match falls into one of these categories:
  - `create-issues/SKILL.md` itself (the skill definition)
  - `plan-sprint/SKILL.md` (bulk-create prompt + description)
  - `docs/process/README.md` per-feature flow (at planning position)
  - `docs/process/README.md` ROADMAP-columns sentence
  - `docs/process/AGENTS.md` "docs → tracker" rule (unchanged — still lists /create-issues as a docs-repo skill; correct)
  - The reshape spec + this plan (`docs/superpowers/specs/…` and `docs/superpowers/plans/…`)

  Any match that describes `/create-issues` running "after the spec PR merges" or "on approved
  spec" is a stale reference and must be fixed.

- [ ] **Step 2: Search for stale L3-sub-issue-materialization references.**

  Run:

  ```bash
  grep -rn 'L3.*sub-issue\|sub-issue.*L3\|L3-derived' .claude/ docs/process/
  ```

  Expected: zero matches. The old opt-in-L3-sub-issues flow is gone from create-issues; no
  other skill or doc should mention it.

- [ ] **Step 3: Confirm `/split-issue` behavior is unchanged.**

  Run:

  ```bash
  git diff master -- .claude/skills/split-issue/SKILL.md
  ```

  Expected: only the one paragraph inserted in Task 3. No changes to the `--groups`, `--refine`,
  `--dry-run`, or auto-propose sections.

- [ ] **Step 4: No commit** — this task produces no writes if the previous tasks were clean. If
  any grep in step 1 or 2 flags a stale reference, fix it and add an amend-or-followup commit
  scoped to that file.

---

## Task 6: Open the MR

**Files:** none modified in this task.

- [ ] **Step 1: Push the branch.**

  Run:

  ```bash
  git push -u origin spec/create-issues-planning-time
  ```

  Expected: push succeeds; GitLab prints the "create MR" URL.

- [ ] **Step 2: Open the MR via GitLab API.**

  Use `$GITLAB_TOKEN` and the pattern from prior MRs in this repo (see `!13`, `!15`, `!16`,
  `!17` in `dac-docs-template` — same repo, same `PROJECT_PATH`):

  - **Title:** `feat: reshape /create-issues to planning-time-only + /plan-sprint bulk-create + /split-issue L3 context`
  - **Target:** `master`
  - **Description:** Reference the spec (`docs/superpowers/specs/2026-09-09-create-issues-planning-time-design.md`), summarize the four file changes, and include a test-plan checklist covering the acceptance criteria from the spec (six items).
  - **Options:** `remove_source_branch: true`.

- [ ] **Step 3: Print the MR URL for the user.**

  Expected: MR opened on `dac-docs-template` in the `dacdigital/tech-pulse/spec-driven-development`
  group, ready for human review.

---

## Self-review notes (for the implementing agent)

- **Spec coverage.** Every acceptance criterion in the spec maps to a task: AC1-2 → Task 1
  (rewritten create-issues); AC3 → Task 2 (plan-sprint bulk-create); AC4 → Task 3
  (split-issue docs-only paragraph); AC5 → Task 4 (flow diagram reorder); AC6-7 → Task 5
  (verification greps).

- **Placeholder scan.** No TBD/TODO in the plan. Every step names its file, its exact edit,
  and its verification command with expected output.

- **Type/naming consistency.** `/create-issues` → `/create-issues` everywhere (no rename).
  ROADMAP column names (`Feature`, `Priority`, `Level`, `Module`, `Spec`, `Issues`,
  `Milestone`, `Link`) match the schema in `docs/superpowers/specs/2026-08-12-sprint-driven-roadmap-design.md` Part 3, verified in Task 4 step 3.

- **Frequent commits.** Six commits across Tasks 1-6 (one per file changed, plus one for the
  push). Each commit is a self-contained, revertible unit.
