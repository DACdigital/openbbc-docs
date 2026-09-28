# Split `/new-feature` into `/new-feature` (arch) + `/new-spec` (spec) — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Split the current fused `/new-feature` skill into two commands with cleanly separated
concerns — `/new-feature` for evolving `docs/architecture/current/`, `/new-spec` for initializing a
feature spec against an existing arch node — and rename the change-classification axis from
`L0/L1/L2` to `C0/C1/C2` so it stops colliding with the modularity tree's `L1/L2/L3` axis.

**Architecture:** Docs-only. Sixteen files touched, one new skill file. Two axes maintained side
by side: **L1/L2/L3** for arch tree depth (unchanged), **C0/C1/C2** for change class (renamed).
Rename discipline: touch `L0/L1/L2` only where it means "change class"; leave every reference to
modularity-tree depth alone.

**Tech Stack:** Markdown.

**Commit policy:** Do not commit the spec or plan. Implementation edits may be committed.
Recommended branch: `feat/new-feature-new-spec-split` (branched from `master`).

**Spec:** `docs/superpowers/specs/2026-09-01-new-feature-new-spec-split-design.md`

---

## File Structure

**Modify (existing):**
- `docs/process/README.md` — rename change-class L→C in change-classification table + prose;
  update per-feature flow diagram; add "Two entry points" subsection.
- `docs/process/AGENTS.md` — rename change-class L→C; rename required-section 2 heading to
  "Change level (C0/C1/C2)"; restate hard rules with C-prefix.
- `docs/conventions/api.md` — rename two change-class references.
- `docs/conventions/architecture.md` — rename one change-class reference.
- `.claude/agents/spec-review.md` — rename change-class L→C.
- `.claude/agents/architecture-review.md` — rename change-class L→C.
- `.claude/agents/contract-data-event-review.md` — rename change-class L→C.
- `.claude/skills/spec-review/SKILL.md` — rename change-class L→C.
- `.claude/skills/arch-review/SKILL.md` — rename change-class L→C.
- `.claude/skills/contract-data-event-review/SKILL.md` — rename change-class L→C.
- `.claude/skills/arch-log-review/SKILL.md` — surgical rename (one line, one ref).
- `.claude/skills/create-issues/SKILL.md` — surgical rename (change-class refs; leave modularity
  L2 reference alone).
- `.claude/skills/plan-sprint/SKILL.md` — surgical rename (one line, one ref; leave all modularity
  L1/L2 references alone).
- `.claude/skills/new-feature/SKILL.md` — full rewrite for arch-evolution role.
- `README.md` — rename change-class L→C in repo-layout comment; add Levels section explaining both
  axes side by side.
- `CLAUDE.md` — add Levels section (may link back to `README.md`).

**Create (new):**
- `.claude/skills/new-spec/SKILL.md` — new command initializing a feature spec against an existing
  arch node, with novelty guard.

**Delete:** none.

**Files NOT to touch** (they contain L-refs that are all modularity, not change class):
- `.claude/skills/modularize/SKILL.md`.

---

## Task 1 — Baseline verification and branch

**Files:** none modified.

- [ ] **Step 1: Confirm clean starting state**

Run: `git status && git branch --show-current`
Expected: on `master`, up to date with `origin/master`. Untracked spec/plan files under
`docs/superpowers/{plans,specs}/` are pre-existing and OK to leave.

- [ ] **Step 2: Baseline the change-class L-refs**

Run:

    git grep -nE '\bL[012]\b' -- '.claude/' 'docs/process/' 'docs/conventions/' 'README.md' 'CLAUDE.md' | wc -l

Expected: 60-ish matches (mixed change-class and modularity). Do not act on this number as a
verification target — later tasks target specific strings. The purpose is to notice the baseline
so you spot obvious over-reach after each task.

- [ ] **Step 3: Create the working branch**

Run: `git checkout -b feat/new-feature-new-spec-split`

If a branch by this name exists locally, ask the user before proceeding (do not force).

---

## Task 2 — Rename change-class L→C in `docs/process/README.md`

**Files:**
- Modify: `docs/process/README.md`

**Scope for this task:** rename change-class references only. The lines describing modularity
(Architecture (L1) → Service (L2) → Implementation (L3); tree L2 nodes; L2 module into L3 shards;
L1 parent) are OUT OF SCOPE — do not touch them.

- [ ] **Step 1: Replace the change-classification prose sentence (line 16)**

Old:

    Every change is classified L0/L1/L2 before work starts. The level decides how much process applies.

New:

    Every change is classified C0/C1/C2 before work starts. The change level decides how much process applies.

- [ ] **Step 2: Replace the change-classification table rows (lines 18-22)**

Old:

    | Level | When | Process |
    |-------|------|---------|
    | **L0 — trivial** | typo, comment, one-liner, rename with no contract impact, safe patch bump | fast lane — no spec, done directly in the service repo |
    | **L1 — normal** | typical functional change; no impact on architecture/DB/API/events | spec → spec-review → (optional plan) → issues → implement → code PR review |
    | **L2 — architectural/data/contract** | new bounded context; DB/API/event/contract change; tenant isolation; framework upgrade | L1 + architecture-review + contract/data/event-review before issues |

New:

    | Change level | When | Process |
    |--------------|------|---------|
    | **C0 — trivial** | typo, comment, one-liner, rename with no contract impact, safe patch bump | fast lane — no spec, done directly in the service repo |
    | **C1 — normal** | typical functional change; no impact on architecture/DB/API/events | spec → spec-review → (optional plan) → issues → implement → code PR review |
    | **C2 — architectural/data/contract** | new bounded context; DB/API/event/contract change; tenant isolation; framework upgrade | C1 + architecture-review + contract/data/event-review before issues |

- [ ] **Step 3: Replace the fast-lane rule (line 27)**

Old:

    - The fast lane is **L0-only** — never for changes touching DB, migrations, API, events, security,

New:

    - The fast lane is **C0-only** — never for changes touching DB, migrations, API, events, security,

- [ ] **Step 4: Replace gate references (lines 34, 37, 42)**

Line 34 — old:

    - **Spec gate (L1/L2)** — happens **here**, on the spec PR that adds `docs/<feature>/spec.md`.

Line 34 — new:

    - **Spec gate (C1/C2)** — happens **here**, on the spec PR that adds `docs/<feature>/spec.md`.

Line 37 — old:

    - **Architecture + contract/data/event gates (L2 only)** — happen on the **same spec PR**, via

Line 37 — new:

    - **Architecture + contract/data/event gates (C2 only)** — happen on the **same spec PR**, via

Line 42 — old:

      of L1/L2 — a pure refactor still triggers it.

Line 42 — new:

      of C1/C2 — a pure refactor still triggers it.

- [ ] **Step 5: Replace the per-feature-flow gate token (line 101)**

Old:

      → [L2] /arch-review + /contract-data-event-review → verdicts on same PR

New:

      → [C2] /arch-review + /contract-data-event-review → verdicts on same PR

Note: this diagram is fully replaced in Task 14. This step is a stopgap that keeps the file
consistent between now and then.

- [ ] **Step 6: Replace the "usually L0" line (line 112)**

Old:

    usually L0 or fold into the feature's existing L1/L2 spec.

New:

    usually C0 or fold into the feature's existing C1/C2 spec.

- [ ] **Step 7: Verify no accidental modularity rewrites**

Run: `git diff docs/process/README.md`

Confirm the diff does NOT touch any of these lines:
- `Between spec-approval and issue-creation, the team can decompose a feature's L2 module into L3`
- `Three levels only: Architecture (L1) → Service (L2) → Implementation (L3).`
- `the sprint's contents (from tree L2 nodes and/or ad-hoc items)`

If any of those changed, revert them.

- [ ] **Step 8: Commit**

    git add docs/process/README.md
    git commit -m "Rename change classification L→C in docs/process/README.md"

---

## Task 3 — Rename change-class L→C in `docs/process/AGENTS.md`

**Files:**
- Modify: `docs/process/AGENTS.md`

All L-references in this file are change-class. No modularity refs to preserve.

- [ ] **Step 1: Replace required-section item 2 heading (line 16)**

Old:

    2. **Level (L0/L1/L2)** — the proposed classification per `docs/process/README.md`, with a one-line

New:

    2. **Change level (C0/C1/C2)** — the proposed classification per `docs/process/README.md`, with a one-line

- [ ] **Step 2: Replace Contracts requirement (line 19)**

Old:

    4. **Contracts** — API / data / events touched or introduced. **Required for L2**; may be stated as

New:

    4. **Contracts** — API / data / events touched or introduced. **Required for C2**; may be stated as

- [ ] **Step 3: Replace L1/L0-adjacent wording (line 20)**

Old:

       "none" for L1/L0-adjacent specs, but the section itself must still be present.

New:

       "none" for C1/C0-adjacent specs, but the section itself must still be present.

- [ ] **Step 4: Replace ambiguous-level hard rule (line 28)**

Old:

    - **Ambiguous level → pick the higher one.** If it is unclear whether a change is L0/L1 or L1/L2,

New:

    - **Ambiguous change level → pick the higher one.** If it is unclear whether a change is C0/C1 or C1/C2,

- [ ] **Step 5: Replace escalation rule (line 31)**

Old:

      `/contract-data-event-review` may escalate an L1 spec to L2 if they find architecture/data/contract

New:

      `/contract-data-event-review` may escalate a C1 spec to C2 if they find architecture/data/contract

- [ ] **Step 6: Replace fast-lane hard rule (lines 33 and 35)**

Line 33 — old:

    - **The fast lane is L0-only.** Never route a change through the fast lane — no spec, done directly in

Line 33 — new:

    - **The fast lane is C0-only.** Never route a change through the fast lane — no spec, done directly in

Line 35 — old:

      isolation, or a framework/dependency upgrade. Any of those makes the change at least L1.

Line 35 — new:

      isolation, or a framework/dependency upgrade. Any of those makes the change at least C1.

- [ ] **Step 7: Verify**

Run: `git grep -nE '\bL[012]\b' -- docs/process/AGENTS.md`
Expected: no matches.

- [ ] **Step 8: Commit**

    git add docs/process/AGENTS.md
    git commit -m "Rename change classification L→C in docs/process/AGENTS.md"

---

## Task 4 — Rename change-class L→C in `docs/conventions/`

**Files:**
- Modify: `docs/conventions/api.md`
- Modify: `docs/conventions/architecture.md`

All L-references in these files are change-class.

- [ ] **Step 1: Update `docs/conventions/api.md` line 3**

Old:

    APIs are contracts between services and their consumers. Changing one is an L2 change (see

New:

    APIs are contracts between services and their consumers. Changing one is a C2 change (see

- [ ] **Step 2: Update `docs/conventions/api.md` line 41**

Old:

      review before implementation (the L2 gate).

New:

      review before implementation (the C2 gate).

- [ ] **Step 3: Update `docs/conventions/architecture.md` line 41**

Old:

    - Do not split for org-chart reasons alone or "just in case" — a new context is an L2 change (see

New:

    - Do not split for org-chart reasons alone or "just in case" — a new context is a C2 change (see

- [ ] **Step 4: Verify**

Run: `git grep -nE '\bL[012]\b' -- docs/conventions/`
Expected: no matches.

- [ ] **Step 5: Commit**

    git add docs/conventions/api.md docs/conventions/architecture.md
    git commit -m "Rename change classification L→C in docs/conventions/"

---

## Task 5 — Rename change-class L→C in `.claude/agents/spec-review.md`

**Files:**
- Modify: `.claude/agents/spec-review.md`

All L-references in this file are change-class.

- [ ] **Step 1: Update description (line 3)**

Old:

    description: Reviews a feature spec (docs/<feature>/spec.md) for completeness against the required sections, consistency, clarity, correct L0/L1/L2 classification, and YAGNI. Emits a PASS/PASS_WITH_ISSUES/FAIL verdict for the invoking command to post as a comment on the spec PR. Use when /spec-review is invoked on an open spec PR, before architecture/contract review or human approval.

New:

    description: Reviews a feature spec (docs/<feature>/spec.md) for completeness against the required sections, consistency, clarity, correct C0/C1/C2 change-level classification, and YAGNI. Emits a PASS/PASS_WITH_ISSUES/FAIL verdict for the invoking command to post as a comment on the spec PR. Use when /spec-review is invoked on an open spec PR, before architecture/contract review or human approval.

- [ ] **Step 2: Update Inputs step 1 (line 18)**

Old:

    1. `docs/process/README.md` — the process, the L0/L1/L2 classification table, and the gates.

New:

    1. `docs/process/README.md` — the process, the C0/C1/C2 change-classification table, and the gates.

- [ ] **Step 3: Update check 1 heading + text (lines 32-34)**

Old:

       order: Business value / Why, Level (L0/L1/L2), Scope (in / out), Contracts, Acceptance criteria,
       Risks & assumptions. A missing section is a blocking finding. Contracts must be present even for L1
       (it may say "none"); for L2 it must have real content.

New:

       order: Business value / Why, Change level (C0/C1/C2), Scope (in / out), Contracts, Acceptance criteria,
       Risks & assumptions. A missing section is a blocking finding. Contracts must be present even for C1
       (it may say "none"); for C2 it must have real content.

- [ ] **Step 4: Update check 3 (lines 38-40)**

Old:

    3. **Level classification.** Check the declared L0/L1/L2 against the table in `docs/process/README.md`.
       If the spec's own Scope or Contracts describe DB/migration, API, event, security, tenant-isolation,
       or framework-upgrade impact but the spec claims L1 or lower, flag it and state the correct level.

New:

    3. **Change-level classification.** Check the declared C0/C1/C2 against the table in `docs/process/README.md`.
       If the spec's own Scope or Contracts describe DB/migration, API, event, security, tenant-isolation,
       or framework-upgrade impact but the spec claims C1 or lower, flag it and state the correct level.

- [ ] **Step 5: Update FAIL line (line 55)**

Old:

      testable; Contracts section missing or empty for a change that is clearly L2; internal contradictions

New:

      testable; Contracts section missing or empty for a change that is clearly C2; internal contradictions

- [ ] **Step 6: Update level-flag output-format line (near line 85)**

Locate:

    If the change looks under-classified, add one line directly after the verdict:
    `**Level:** spec declares L<n>; recommend L<n+1> — <one-line reason>.`

Replace with:

    If the change looks under-classified, add one line directly after the verdict:
    `**Change level:** spec declares C<n>; recommend C<n+1> — <one-line reason>.`

- [ ] **Step 7: Verify**

Run: `git grep -nE '\bL[012]\b' -- .claude/agents/spec-review.md`
Expected: no matches.

- [ ] **Step 8: Commit**

    git add .claude/agents/spec-review.md
    git commit -m "Rename change classification L→C in agents/spec-review"

---

## Task 6 — Rename change-class L→C in `.claude/agents/architecture-review.md`

**Files:**
- Modify: `.claude/agents/architecture-review.md`

All L-references in this file are change-class.

- [ ] **Step 1: Update description (line 3)**

Old:

    description: L2-only. Reviews a feature spec's architectural impact against docs/conventions/architecture.md — bounded contexts, service boundaries, dependency direction, data ownership, tenant isolation. Emits a PASS/PASS_WITH_ISSUES/FAIL verdict for the invoking command to post as a comment on the spec PR. Use when /arch-review is invoked on an L2 spec, after spec-review has returned PASS or PASS_WITH_ISSUES.

New:

    description: C2-only. Reviews a feature spec's architectural impact against docs/conventions/architecture.md — bounded contexts, service boundaries, dependency direction, data ownership, tenant isolation. Emits a PASS/PASS_WITH_ISSUES/FAIL verdict for the invoking command to post as a comment on the spec PR. Use when /arch-review is invoked on a C2 spec, after spec-review has returned PASS or PASS_WITH_ISSUES.

- [ ] **Step 2: Update body opening (line 10)**

Old:

    **L2** specs (per `docs/process/README.md`), on the same spec PR as spec-review. You assess

New:

    **C2** specs (per `docs/process/README.md`), on the same spec PR as spec-review. You assess

- [ ] **Step 3: Update inputs line 20**

Old:

    1. `docs/process/README.md` and `docs/process/AGENTS.md` — confirm the spec is genuinely L2, and recall

New:

    1. `docs/process/README.md` and `docs/process/AGENTS.md` — confirm the spec is genuinely C2, and recall

- [ ] **Step 4: Verify**

Run: `git grep -nE '\bL[012]\b' -- .claude/agents/architecture-review.md`
Expected: no matches.

- [ ] **Step 5: Commit**

    git add .claude/agents/architecture-review.md
    git commit -m "Rename change classification L→C in agents/architecture-review"

---

## Task 7 — Rename change-class L→C in `.claude/agents/contract-data-event-review.md`

**Files:**
- Modify: `.claude/agents/contract-data-event-review.md`

All L-references in this file are change-class.

- [ ] **Step 1: Update description (line 3)**

Old:

    description: L2-only. Reviews a feature spec's API/data/event contract changes against docs/conventions/api.md, persistence.md, and events.md — versioning, backward compatibility, schema evolution, migration safety. Emits a PASS/PASS_WITH_ISSUES/FAIL verdict for the invoking command to post as a comment on the spec PR. Use when /contract-data-event-review is invoked on an L2 spec, alongside architecture-review.

New:

    description: C2-only. Reviews a feature spec's API/data/event contract changes against docs/conventions/api.md, persistence.md, and events.md — versioning, backward compatibility, schema evolution, migration safety. Emits a PASS/PASS_WITH_ISSUES/FAIL verdict for the invoking command to post as a comment on the spec PR. Use when /contract-data-event-review is invoked on a C2 spec, alongside architecture-review.

- [ ] **Step 2: Update body opening (line 10)**

Old:

    only for **L2** specs, on the same spec PR as spec-review and architecture-review. You assess the

New:

    only for **C2** specs, on the same spec PR as spec-review and architecture-review. You assess the

- [ ] **Step 3: Update inputs lines 20-21**

Old:

    1. `docs/process/README.md` and `docs/process/AGENTS.md` — confirm L2 classification and that a
       Contracts section is present (required for L2); recall the escalation rule: raise the level, never

New:

    1. `docs/process/README.md` and `docs/process/AGENTS.md` — confirm C2 classification and that a
       Contracts section is present (required for C2); recall the escalation rule: raise the change level, never

- [ ] **Step 4: Verify**

Run: `git grep -nE '\bL[012]\b' -- .claude/agents/contract-data-event-review.md`
Expected: no matches.

- [ ] **Step 5: Commit**

    git add .claude/agents/contract-data-event-review.md
    git commit -m "Rename change classification L→C in agents/contract-data-event-review"

---

## Task 8 — Rename change-class L→C in `.claude/skills/spec-review/SKILL.md`

**Files:**
- Modify: `.claude/skills/spec-review/SKILL.md`

All L-references in this file are change-class.

- [ ] **Step 1: Update Purpose paragraph (line 12)**

Old:

    open spec PR. This is the gate check for L1 (and the first of three, for L2) — the human PL still

New:

    open spec PR. This is the gate check for C1 (and the first of three, for C2) — the human PL still

- [ ] **Step 2: Update step 4 (line 30)**

Old:

    4. If the subagent raises the level (e.g. L1 → L2), report it as a blocking finding in the PR comment

New:

    4. If the subagent raises the change level (e.g. C1 → C2), report it as a blocking finding in the PR comment

- [ ] **Step 3: Update step 4 continuation (line 32)**

Old:

       Level section and runs `/arch-review` + `/contract-data-event-review`, which are now required

New:

       Change level section and runs `/arch-review` + `/contract-data-event-review`, which are now required

- [ ] **Step 4: Verify**

Run: `git grep -nE '\bL[012]\b' -- .claude/skills/spec-review/SKILL.md`
Expected: no matches.

- [ ] **Step 5: Commit**

    git add .claude/skills/spec-review/SKILL.md
    git commit -m "Rename change classification L→C in skills/spec-review"

---

## Task 9 — Rename change-class L→C in `.claude/skills/arch-review/SKILL.md`

**Files:**
- Modify: `.claude/skills/arch-review/SKILL.md`

All L-references in this file are change-class.

- [ ] **Step 1: Update description (line 3)**

Old:

    description: "L2 only — run the architecture-review agent and post its verdict on the spec PR"

New:

    description: "C2 only — run the architecture-review agent and post its verdict on the spec PR"

- [ ] **Step 2: Update Purpose paragraph (line 11)**

Old:

    For L2 (architectural/data/contract) changes only: run the `architecture-review` subagent against

New:

    For C2 (architectural/data/contract) changes only: run the `architecture-review` subagent against

- [ ] **Step 3: Update Purpose closing (line 14)**

Old:

    pass before a human approves and merges an L2 spec PR.

New:

    pass before a human approves and merges a C2 spec PR.

- [ ] **Step 4: Update step 1 (lines 22-24)**

Old:

    1. **Check the spec's declared Level.** If it is L0 or L1, warn the user that architecture review is
       only required for L2 and ask for confirmation before proceeding (this command may still raise the
       level to L2 if it finds architecture impact — a reviewer can raise, never lower).

New:

    1. **Check the spec's declared Change level.** If it is C0 or C1, warn the user that architecture review is
       only required for C2 and ask for confirmation before proceeding (this command may still raise the
       change level to C2 if it finds architecture impact — a reviewer can raise, never lower).

- [ ] **Step 5: Verify**

Run: `git grep -nE '\bL[012]\b' -- .claude/skills/arch-review/SKILL.md`
Expected: no matches.

- [ ] **Step 6: Commit**

    git add .claude/skills/arch-review/SKILL.md
    git commit -m "Rename change classification L→C in skills/arch-review"

---

## Task 10 — Rename change-class L→C in `.claude/skills/contract-data-event-review/SKILL.md`

**Files:**
- Modify: `.claude/skills/contract-data-event-review/SKILL.md`

All L-references in this file are change-class.

- [ ] **Step 1: Update description (line 3)**

Old:

    description: "L2 only — run the contract-data-event-review agent and post its verdict on the spec PR"

New:

    description: "C2 only — run the contract-data-event-review agent and post its verdict on the spec PR"

- [ ] **Step 2: Update Purpose paragraph (line 11)**

Old:

    For L2 (architectural/data/contract) changes only: run the `contract-data-event-review` subagent

New:

    For C2 (architectural/data/contract) changes only: run the `contract-data-event-review` subagent

- [ ] **Step 3: Update Purpose closing (line 14)**

Old:

    before a human approves and merges an L2 spec PR.

New:

    before a human approves and merges a C2 spec PR.

- [ ] **Step 4: Update step 1 (lines 22-24)**

Old:

    1. **Check the spec's declared Level.** If it is L0 or L1, warn the user that this review is only
       required for L2 and ask for confirmation before proceeding (this command may still raise the level
       to L2 if it finds contract/data/event impact the spec didn't declare — a reviewer can raise, never
       lower).

New:

    1. **Check the spec's declared Change level.** If it is C0 or C1, warn the user that this review is only
       required for C2 and ask for confirmation before proceeding (this command may still raise the change
       level to C2 if it finds contract/data/event impact the spec didn't declare — a reviewer can raise, never
       lower).

- [ ] **Step 5: Update step 3 tail (line 30)**

Old:

       event schema/versioning for what the spec declares. A spec with an L2-required Contracts section

New:

       event schema/versioning for what the spec declares. A spec with a C2-required Contracts section

- [ ] **Step 6: Verify**

Run: `git grep -nE '\bL[012]\b' -- .claude/skills/contract-data-event-review/SKILL.md`
Expected: no matches.

- [ ] **Step 7: Commit**

    git add .claude/skills/contract-data-event-review/SKILL.md
    git commit -m "Rename change classification L→C in skills/contract-data-event-review"

---

## Task 11 — Surgical renames in `arch-log-review`, `create-issues`, `plan-sprint` skills

These three files each mix change-class refs with modularity refs. Edit only the change-class
lines listed below; leave everything else untouched.

**Files:**
- Modify: `.claude/skills/arch-log-review/SKILL.md`
- Modify: `.claude/skills/create-issues/SKILL.md`
- Modify: `.claude/skills/plan-sprint/SKILL.md`

- [ ] **Step 1: `arch-log-review` description (line 3) — change-class**

Old:

    description: On-demand review of a PR that touches docs/architecture/current/ — invokes the architecture-log-review agent, posts the verdict as a PR comment. Independent of L1/L2 — any PR touching current/ needs the paired log entry.

New:

    description: On-demand review of a PR that touches docs/architecture/current/ — invokes the architecture-log-review agent, posts the verdict as a PR comment. Independent of C1/C2 — any PR touching current/ needs the paired log entry.

- [ ] **Step 2: `create-issues` line 18 — change-class**

Old:

      in `docs/process/README.md`); for L2 features, also fails if `/arch-review` and

New:

      in `docs/process/README.md`); for C2 features, also fails if `/arch-review` and

- [ ] **Step 3: `create-issues` line 23 — change-class**

Old:

    1. **Verify the gate**: the spec PR for `<name>` is merged. If L2, both review verdicts are present

New:

    1. **Verify the gate**: the spec PR for `<name>` is merged. If C2, both review verdicts are present

- [ ] **Step 4: `create-issues` line 42 — leave untouched (modularity)**

The line `the L2 node whose slug matches` refers to modularity depth. Do not change it.

- [ ] **Step 5: `plan-sprint` line 47 — change-class**

Old:

         (P0/P1/P2), `level` (L0/L1/L2). `Module` column will be left blank for these.

New:

         (P0/P1/P2), `change_level` (C0/C1/C2). `Module` column will be left blank for these.

- [ ] **Step 6: `plan-sprint` other L-refs — leave untouched (modularity)**

Every other L1/L2 reference in `plan-sprint` (lines 3, 12, 36, 42-45) refers to the modularity
tree. Do not change them.

- [ ] **Step 7: Verify**

Run:

    git grep -nE '\bL[012]\b' -- .claude/skills/arch-log-review/SKILL.md .claude/skills/create-issues/SKILL.md

Expected: only line 42 of `create-issues` (the modularity L2 node reference).

Run:

    git grep -nE '\bL[012]\b' -- .claude/skills/plan-sprint/SKILL.md

Expected: modularity refs only (lines 3, 12, 36, 42-45).

- [ ] **Step 8: Commit**

    git add .claude/skills/arch-log-review/SKILL.md .claude/skills/create-issues/SKILL.md .claude/skills/plan-sprint/SKILL.md
    git commit -m "Rename change classification L→C in arch-log-review, create-issues, plan-sprint"

---

## Task 12 — Add Levels section to `README.md` and rename inline change-class ref

**Files:**
- Modify: `README.md`

- [ ] **Step 1: Rename inline change-class ref (line 48)**

Old:

    docs/process/         the process — L0/L1/L2, gates, per-feature flow (tool-neutral source of truth)

New:

    docs/process/         the process — C0/C1/C2 change class, gates, per-feature flow (tool-neutral source of truth)

- [ ] **Step 2: Add the "Levels" section immediately before the "Repo layout" section**

Insert after line 44 (blank line ending Quick start) and before line 45 (`## Repo layout` heading):

    ## Levels

    This project uses two independent "level" axes. Do not confuse them.

    **Architecture tree levels (L1 / L2 / L3)** — depth in the modularity tree under
    `docs/architecture/current/modularity/`. Managed by `/modularize`.

    | Level | Name | What it holds |
    |-------|------|---------------|
    | **L1** | Architecture | services / infra entities (children of the tree root) |
    | **L2** | Service | contract modules within a service |
    | **L3** | Implementation | issue-sized shards to feed `/create-issues` |

    Only L1 and L2 are architecture-of-record; L3 is planning-time decomposition stored in the same
    tree for convenience.

    **Change classification (C0 / C1 / C2)** — how much process a change goes through.

    | Level | Name | When | Process |
    |-------|------|------|---------|
    | **C0** | trivial | typo, comment, one-liner, safe patch bump | fast lane — no spec |
    | **C1** | normal | typical functional change; no arch/DB/API/event impact | spec → `/spec-review` → issues → code PR |
    | **C2** | architectural | new bounded context; DB/API/event/contract change; tenant isolation; framework upgrade | C1 + `/arch-review` + `/contract-data-event-review` |

    **The two axes are independent.** An L2 module (Service level) may host either a C1 or a C2
    change; an L1 addition is usually a C2 change but not by definition.

- [ ] **Step 3: Verify**

Run: `git grep -nE '\bL[012]\b' -- README.md`

Expected: the Levels section's own L1/L2/L3 rows (modularity — those are the new content) but no
change-class L-refs. The line-48 rename should now show C0/C1/C2.

- [ ] **Step 4: Commit**

    git add README.md
    git commit -m "Add Levels section and rename change class L→C in README"

---

## Task 13 — Add Levels section to `CLAUDE.md`

**Files:**
- Modify: `CLAUDE.md`

`CLAUDE.md` currently starts with `@AGENTS.md` and lists project context filled by
`/setup-project`. Add a Levels section that mirrors `README.md`'s.

- [ ] **Step 1: Insert Levels section after `@AGENTS.md` line**

Insert between the current line 1 (`@AGENTS.md`) and line 3 (`## Project context`):

    ## Levels

    Two independent "level" axes — do not confuse them. See `README.md#levels` for full tables.

    - **Architecture tree levels (L1 / L2 / L3)** — depth in the modularity tree under
      `docs/architecture/current/modularity/`. Managed by `/modularize`.
      - **L1** Architecture — services / infra entities.
      - **L2** Service — contract modules within a service.
      - **L3** Implementation — issue-sized shards to feed `/create-issues`.
      Only L1 and L2 are architecture-of-record.
    - **Change classification (C0 / C1 / C2)** — how much process a change goes through.
      - **C0** trivial — fast lane, no spec.
      - **C1** normal — spec → `/spec-review` → issues → code PR.
      - **C2** architectural/data/contract — C1 + `/arch-review` + `/contract-data-event-review`.

    The two axes are independent: an L2 module may host either a C1 or a C2 change.

- [ ] **Step 2: Verify**

Run: `git grep -nE '\bL[012]\b' -- CLAUDE.md`
Expected: L1/L2/L3 in the new Levels section only, no change-class refs.

- [ ] **Step 3: Commit**

    git add CLAUDE.md
    git commit -m "Add Levels section to CLAUDE.md"

---

## Task 14 — Update per-feature flow diagram + add "Two entry points" subsection

**Files:**
- Modify: `docs/process/README.md`

- [ ] **Step 1: Replace the per-feature flow code block (lines 97-108)**

Old:

    ```
    select feature (from ROADMAP)
      → /new-feature  → docs/<feature>/spec.md  → spec PR
      → /spec-review  → verdict on PR
      → [C2] /arch-review + /contract-data-event-review → verdicts on same PR
      → human approves + merges spec PR         ← GATE
      → /create-issues → parent issue in tracker; opt-in sub-issues from L3 tree if present
      → /roadmap-sync  → ROADMAP links spec + issues + tracker; manages tracker milestones/epics
      → Devs implement in workspace/<repo>, code PRs reviewed & merged
         ↳ /split-issue <id> if a dev/reviewer wants smaller PRs mid-flight (tracker-only)
      → all child issues close → feature done → ROADMAP updated
    ```

New:

    ```
    select feature (from ROADMAP or ad-hoc)
      │
      ├─ if capability not yet in solution architecture:
      │    /new-feature   → docs/architecture/current/** + docs/architecture/logs/…
      │                   → arch PR → /arch-log-review → human merge  ← GATE
      │
      ├─ /new-spec        → docs/<feature>/spec.md
      │                   → spec PR → /spec-review [+ /arch-review, /contract-data-event-review for C2]
      │                   → human merge  ← GATE
      │
      ├─ /create-issues   → parent issue in tracker; opt-in sub-issues from L3 tree if present
      ├─ /roadmap-sync    → ROADMAP links spec + issues + tracker; manages tracker milestones/epics
      ├─ Devs implement in workspace/<repo>, code PRs reviewed & merged
      │  ↳ /split-issue <id> if a dev/reviewer wants smaller PRs mid-flight (tracker-only)
      └─ all child issues close → feature done → ROADMAP updated
    ```

- [ ] **Step 2: Add a "Two entry points" subsection immediately below the flow block**

Insert this new subsection between the closing ``` of the flow block and the paragraph that
begins "Task specs / sub-tasks below the feature spec are optional…":

    ### Two entry points

    Feature work has two entry commands. Use whichever matches your intent:

    - **`/new-feature <name>`** — evolve `docs/architecture/current/`: add a new capability, edit
      NFRs/assumptions, or introduce an L1/L2 tree node. Opens an **arch PR** gated by
      `/arch-log-review`. Never writes a `spec.md`.
    - **`/new-spec <name>`** — initialize a feature spec from an existing arch node. Opens a
      **spec PR** gated by `/spec-review` (plus `/arch-review` and `/contract-data-event-review`
      when C2). Never touches `docs/architecture/`.

    When a feature needs both, the arch PR merges first, then `/new-spec` runs and its spec PR
    references the newly-merged arch. `/new-spec` blocks with an instruction to run `/new-feature`
    first when the target capability is a novelty (not in `current/`).

- [ ] **Step 3: Verify no L-refs regressed**

Run: `git grep -nE '\bL[012]\b' -- docs/process/README.md`
Expected: only modularity refs (`Architecture (L1) → Service (L2) → Implementation (L3)`;
`decompose a feature's L2 module into L3`; `tree L2 nodes`; `L3 tree if present` in the flow).

- [ ] **Step 4: Commit**

    git add docs/process/README.md
    git commit -m "Update per-feature flow diagram and add Two entry points subsection"

---

## Task 15 — Rewrite `.claude/skills/new-feature/SKILL.md` for arch-evolution role

**Files:**
- Modify: `.claude/skills/new-feature/SKILL.md` (full rewrite)

- [ ] **Step 1: Replace the entire file content**

Overwrite the file with:

    ---
    name: new-feature
    description: Evolve docs/architecture/current/ — add/change arch content (NFRs, assumptions, README/topology, L1/L2 tree nodes), write the paired arch-log entry, and open the arch PR. Never writes a spec; for that use /new-spec.
    disable-model-invocation: true
    ---

    # /new-feature `<name>`

    ## Purpose

    Evolve the solution architecture. Adds or changes content under `docs/architecture/current/`
    (non-tree files and/or L1/L2 modularity nodes), writes the paired arch-log entry at
    `docs/architecture/logs/YYYY-MM-DD-<name>/README.md`, and opens the arch PR that
    `/arch-log-review` will comment on.

    This command does **not** write a feature spec. To initialize an implementation spec for a
    capability that is already in `current/`, use `/new-spec <name>`.

    ## Inputs

    - `<name>` (required) — kebab-case codename for the arch change. Used as the arch-log entry
      slug (`YYYY-MM-DD-<name>`) and, when adding a tree node, as the node's slug.

    ## Steps

    1. **Print reverse-hint**: "If you meant to write an implementation spec for an existing
       capability, use `/new-spec <name>` instead. Continue with arch evolution? [Y/n]"

    2. **Validate `<name>`** — kebab-case; today's arch-log codename `YYYY-MM-DD-<name>` must not
       already exist under `docs/architecture/logs/`.

    3. **Load context files.** Read (skip any that don't exist):
       - `docs/architecture/current/README.md` — topology narrative.
       - `docs/architecture/current/nfrs.md` — architectural constraints.
       - `docs/architecture/current/assumptions.md` — project-level assumptions.
       - Modularity tree summary — walk `docs/architecture/current/modularity/**/README.md` up to
         depth 2 (L1 and L2); extract each node's `id`, `title`, and Purpose section.

    4. **Duplicate check.** Semantic match of `<name>` and the user's initial description against
       the loaded context. If overlaps are found, present them and offer three options:
       - **(a) same as X** — abort and print `run /modularize --refine edit on <slug>` to modify
         the existing node instead.
       - **(b) related but distinct** — continue with the new content.
       - **(c) unrelated** — continue.

    5. **Q&A**: what changes (which files under `current/`), the driver, the rationale, the
       alternatives considered and rejected.

    6. **Write the diff to `current/`**:
       - For **L1/L2 tree adds** — create the dir + `README.md` under
         `docs/architecture/current/modularity/<...>/<slug>/README.md` with front-matter
         (`id`, `level`, `parent`, `title`) and the standard `## Purpose` / `## Scope (in / out)`
         sections.
       - For **non-tree edits** (NFRs, assumptions, topology narrative) — patch the target files
         directly.

    7. **Write paired arch-log entry** at `docs/architecture/logs/YYYY-MM-DD-<name>/README.md`
       with all required fields per `docs/process/AGENTS.md` and the `architecture-log-review`
       agent:

           # <name> — <one-line title>

           **Date**: YYYY-MM-DD
           **Codename**: <name>

           **Driver**: new feature: <name>

           **Decision**: <one-line summary of what changed in current/>

           **Rationale**: <captured from Q&A — one paragraph>

           **Alternatives rejected**: <list, one per line, with reason>

           **Impact**:
           - <file 1 touched>
           - <file 2 touched>
           - ...

           **Links**:
           - (user may add related spec PRs or tracker items before merge)

    8. **Modularity-impact assessment.** Identify which touched `current/` paths sit inside
       `modularity/<slug>/` where `<slug>` has one or more child subdirectories. Emit exactly one
       of these verdicts:
       - "No refine needed — no existing decomposed nodes were touched."
       - "Suggest running `/modularize --refine <slug>` on: X, Y. Reason: <purpose changed | new
         sibling added | scope narrowed>."

       Never auto-run `/modularize`. The user decides whether to run refine.

    9. **Open the arch PR**: create a branch, commit only `docs/architecture/**` changes (both
       the `current/` diff and the new `logs/YYYY-MM-DD-<name>/` entry), push, and open a PR
       against this repo's default branch titled after `<name>`. `/arch-log-review` will comment
       with its verdict.

    10. If a comms channel is configured, post a short notification that the arch PR for `<name>`
        is open.

    ## Config

    Reads `.claude/tracker.json` → `comms` (optional notification only). Does not read `tracker`
    or `gitHost` — the arch PR opens against this repo's own git remote.

    ## What this command never does

    - Never writes `docs/<name>/spec.md` or `docs/<name>/plan.md`.
    - Never runs `superpowers:brainstorming` or `superpowers:writing-plans`.
    - Never proposes a change level (C0/C1/C2). Change levels are a property of specs, not of
      arch changes. This PR is gated by `/arch-log-review` alone.
    - Never auto-runs `/modularize` — step 8 only emits a verdict + suggestion.

- [ ] **Step 2: Verify frontmatter and section list**

Read `.claude/skills/new-feature/SKILL.md`. Confirm:
- Frontmatter has `name`, `description`, `disable-model-invocation: true`.
- Section headings are: Purpose / Inputs / Steps / Config / What this command never does.
- No mention of `spec.md`, `plan.md`, `brainstorming`, `writing-plans` inside the Steps section.
- The Steps section has exactly 10 numbered steps.

- [ ] **Step 3: Commit**

    git add .claude/skills/new-feature/SKILL.md
    git commit -m "Rewrite /new-feature as arch-evolution command"

---

## Task 16 — Create `.claude/skills/new-spec/SKILL.md`

**Files:**
- Create: `.claude/skills/new-spec/SKILL.md`

- [ ] **Step 1: Confirm the target directory does not yet exist**

Run: `ls .claude/skills/new-spec/ 2>/dev/null || echo NOT_PRESENT`
Expected: `NOT_PRESENT`.

- [ ] **Step 2: Create the file with the following content**

    ---
    name: new-spec
    description: Initialize a feature spec from an existing arch node. Orchestrates superpowers:brainstorming into docs/<name>/spec.md, proposes the C0/C1/C2 change level, optionally chains superpowers:writing-plans, and opens the spec PR. Blocks if the target capability is a novelty not yet in current/.
    disable-model-invocation: true
    ---

    # /new-spec `<name>` `[--issue <tracker-id>]`

    ## Purpose

    Turn an idea into a reviewable feature spec, grounded in an existing node of the solution
    architecture. Orchestrates `superpowers:brainstorming` to produce `docs/<name>/spec.md`,
    proposes the C0/C1/C2 change level per `docs/process/README.md`, and optionally runs
    `superpowers:writing-plans` for a `plan.md`. Ends by opening the spec PR that `/spec-review`
    (plus `/arch-review` and `/contract-data-event-review` for C2) will comment on.

    This command does **not** write to `docs/architecture/current/` or `docs/architecture/logs/`.
    If the target capability is a novelty not yet in the architecture, the command blocks and
    tells the user to run `/new-feature <name>` first.

    ## Inputs

    - `<name>` (required) — kebab-case feature folder name. Fails if `docs/<name>/spec.md` already
      exists (offer to resume that feature instead).
    - `--issue <tracker-id>` (optional) — a tracker ticket ID (e.g. `PROJ-42`) whose title and
      description seed the brainstorm and inform the arch-node mapping.

    ## Steps

    1. **Print reverse-hint**: "If you meant to add a new capability to the solution architecture,
       use `/new-feature <name>` instead. Continue with spec initialization? [Y/n]"

    2. **Resolve mapping** to a `current/` node — try each mode in order and stop at the first
       hit:
       - `<name>` argument matches a modularity slug (walk
         `docs/architecture/current/modularity/**/` and match by `id` front-matter or dir name)
         → mapped node = that slug.
       - `--issue <tracker-id>` present → pull the ticket via the configured tracker adapter
         (`.claude/tracker.json` → `tracker.platform`), extract title and description; fuzzy-match
         the title against modularity node titles. Ticket body seeds the brainstorm regardless.
       - Prior conversation clearly names a module (only when the skill is invoked mid-
         conversation; not applicable in fresh sessions) → propose that mapping.
       - Exactly one plausible modularity node by fuzzy match on `<name>` → propose that mapping.
       - None of the above → treat as ambiguous.

    3. **Confirm or ask**:
       - **Confidently mapped** → report and confirm:
         `"This spec will attach to <slug> (L<level>, <path>). Continue? [Y/n/change]"`.
       - **`change` selected** or **ambiguous** (zero or multiple candidates) → present a
         numbered list of existing modularity nodes plus a `none` option; user picks one.
       - **`none` chosen** or **no candidates at all** → **BLOCK** and print:

             "This looks like a novelty. Run /new-feature <name> first to add it to
             docs/architecture/current/, then re-run /new-spec <name>."

         Exit without writing any files.

    4. **Load context**:
       - Mapped node's `README.md` + parent chain (walk `parent` back to root).
       - `docs/architecture/current/nfrs.md`.
       - `docs/architecture/current/assumptions.md`.
       - Tracker ticket body if `--issue` was passed.
       - If a ROADMAP row matches `<name>`, its `Priority` and `Milestone` columns.

    5. **Run `superpowers:brainstorming`** with the required-sections override from
       `docs/process/AGENTS.md`. The resulting `spec.md` MUST contain, in order:
       Business value / Why, Change level (C0/C1/C2), Scope (in / out), Contracts, Acceptance
       criteria, Risks & assumptions.

    6. **Mid-brainstorm drift safeguard**: if the design surfaces net-new arch not covered by the
       mapped node — e.g., proposes a whole new service, or a new contract on a different service
       — halt with the same block message as step 3 and exit without writing files.

    7. **Write `docs/<name>/spec.md`** with the six required sections.

    8. **Propose the change level (C0/C1/C2)** per `docs/process/README.md`'s change
       classification, with a one-line justification. Write it into the spec's
       "Change level (C0/C1/C2)" section. Ambiguous → pick higher (hard rule). Never route
       through the C0 fast lane if the change touches DB, API, events, security, tenant
       isolation, or an upgrade.

    9. **Optionally run `superpowers:writing-plans`** to produce `docs/<name>/plan.md` from the
       approved spec (implementation detail; not reviewed).

    10. **Open the spec PR**: create a branch, commit only `docs/<name>/{spec.md[,plan.md]}`,
        push, and open a PR against this repo's default branch titled after `<name>`.
        `/spec-review` will comment; if C2 is proposed, `/arch-review` and
        `/contract-data-event-review` also apply on the same PR.

    11. If a comms channel is configured, post a short notification that the spec PR for
        `<name>` is open.

    ## Config

    Reads `.claude/tracker.json` → `tracker.platform` (only when `--issue` is used) and `comms`
    (optional notification). Does not read `gitHost` — the spec PR opens against this repo's own
    git remote.

    ## What this command never does

    - Never writes to `docs/architecture/current/` or `docs/architecture/logs/`.
    - Never runs `/modularize`.
    - Never proceeds past step 3 when the target capability is a novelty; always blocks with the
      instruction to run `/new-feature` first.

- [ ] **Step 3: Verify**

Read the file back and confirm:
- Frontmatter matches the format used by other skills (name, description,
  `disable-model-invocation: true`).
- Section headings: Purpose / Inputs / Steps / Config / What this command never does.
- Steps section has 11 numbered steps.
- No mention of `docs/architecture/current/` writes in the Steps section (only reads).

Run: `git grep -nE '\bL[012]\b' -- .claude/skills/new-spec/SKILL.md`
Expected: `L<level>` on step 3 (arch-tree level, parametric — this is a template placeholder,
not a change-class rename target). No other matches.

- [ ] **Step 4: Commit**

    git add .claude/skills/new-spec/SKILL.md
    git commit -m "Add /new-spec skill for feature spec initialization"

---

## Task 17 — Final verification

**Files:** none modified.

- [ ] **Step 1: Global change-class L-ref sweep**

Run:

    git grep -nE '\bL[012]\b' -- '.claude/' 'docs/process/' 'docs/conventions/' 'README.md' 'CLAUDE.md' 2>/dev/null

Expected: only modularity references. Concretely, these are the only lines that should match:

- `.claude/skills/modularize/SKILL.md` — all L-refs (out of scope).
- `.claude/skills/plan-sprint/SKILL.md` — lines describing the tree (3, 12, 36, 42-45).
- `.claude/skills/create-issues/SKILL.md` — line 42 (`the L2 node whose slug matches`).
- `docs/process/README.md` — `Architecture (L1) → Service (L2) → Implementation (L3)`, `decompose
  a feature's L2 module into L3`, `tree L2 nodes`, `L3 tree if present` (in the flow diagram).
- `README.md` — the new Levels section's L1/L2/L3 rows (modularity).
- `CLAUDE.md` — the new Levels section's L1/L2/L3 items (modularity).
- `.claude/skills/new-spec/SKILL.md` — `L<level>` parametric placeholder in step 3.

If any line outside this whitelist matches, it's an accidental rename miss — fix it.

- [ ] **Step 2: Read-through of the two behavior-changed skills**

Read `.claude/skills/new-feature/SKILL.md` end-to-end and confirm:
- No occurrence of `spec.md`, `plan.md`, `brainstorming`, or `writing-plans` in the Steps
  section.
- Step 8 (Modularity-impact assessment) explicitly says "Never auto-run `/modularize`".
- Step 9 (Open the arch PR) commits only `docs/architecture/**`.

Read `.claude/skills/new-spec/SKILL.md` end-to-end and confirm:
- Step 3 (Confirm or ask) has the exact `BLOCK` message specified in the spec.
- Step 6 (Mid-brainstorm drift) references the same block message.
- Step 10 (Open the spec PR) commits only `docs/<name>/**`.

- [ ] **Step 3: Cross-command reference check**

Run:

    git grep -n '/new-feature' -- '.claude/' 'docs/' 'README.md' 'CLAUDE.md'

Confirm each remaining reference to `/new-feature` describes it as an arch-evolution command (or
is inside a spec/plan under `docs/superpowers/`, which are historical artefacts left alone).

Run:

    git grep -n '/new-spec' -- '.claude/' 'docs/' 'README.md' 'CLAUDE.md'

Confirm each reference correctly describes it as the spec-initialization command.

- [ ] **Step 4: Commit only if fixes were needed**

If the sweep or read-through surfaced any issue, fix it, then:

    git add <fixed files>
    git commit -m "Fix cross-reference / rename miss found in final verification"

If no fixes were needed, no commit for this task.

---

## Task 18 — Open the PR

**Files:** none modified.

- [ ] **Step 1: Push the branch**

Run: `git push -u origin feat/new-feature-new-spec-split`

- [ ] **Step 2: Open the PR against `master`**

Use the platform's PR-open command (see `.claude/tracker.json` → `gitHost.platform` if unsure —
`gh pr create` for GitHub, `glab mr create` for GitLab). Title:

    Split /new-feature into /new-feature (arch) + /new-spec (spec); rename change class L→C

Body:

    ## Summary
    - `/new-feature` is repurposed to evolve `docs/architecture/current/` only; produces an arch
      PR gated by `/arch-log-review`. Never writes a spec.
    - `/new-spec <name>` is a new command that initializes a feature spec from an existing arch
      node. Blocks with instructions when the target capability is a novelty.
    - Change classification renamed from `L0/L1/L2` to `C0/C1/C2` throughout `.claude/`, `docs/
      process/`, `docs/conventions/`, `README.md`, and `CLAUDE.md`. Modularity tree levels
      (`L1/L2/L3`) untouched.
    - New Levels section in `README.md` and `CLAUDE.md` explains both axes side by side.

    ## Test plan
    - [ ] Run `git grep -nE '\bL[012]\b' -- '.claude/' 'docs/process/' 'docs/conventions/' 'README.md' 'CLAUDE.md'` and confirm only whitelisted modularity refs remain (see plan Task 17).
    - [ ] Read `.claude/skills/new-feature/SKILL.md` and confirm it never mentions `spec.md`,
          `plan.md`, or brainstorming/writing-plans.
    - [ ] Read `.claude/skills/new-spec/SKILL.md` and confirm the novelty-guard block message
          exactly matches the design.
    - [ ] Read `docs/process/README.md` and confirm the per-feature flow diagram shows two entry
          commands with the arch-PR-first ordering.
    - [ ] Dry-run `/new-spec` mentally against a fictional module: confirm it correctly maps and
          confirms, or blocks with the correct instruction.

    Spec: `docs/superpowers/specs/2026-09-01-new-feature-new-spec-split-design.md`

- [ ] **Step 3: Notify user**

Print the PR URL. The user reviews and merges through the normal repo review process.

---

## Self-review notes

**Spec coverage:**
- Acceptance criterion 1 (arch PR content) → Task 15 step 1 (rewrite includes "commit only
  docs/architecture/**").
- Criterion 2 (duplicate check) → Task 15 step 1 (rewrite step 4).
- Criterion 3 (modularity-impact verdict, no auto-run) → Task 15 step 1 (rewrite step 8).
- Criterion 4 (spec PR content) → Task 16 step 2 (new file step 10).
- Criterion 5 (mapping resolution + block) → Task 16 step 2 (new file steps 2-3).
- Criterion 6 (mid-brainstorm drift) → Task 16 step 2 (new file step 6).
- Criterion 7 (global rename) → Tasks 2-11 + verification Task 17.
- Criterion 8 (Levels section) → Tasks 12-13.
- Criterion 9 (per-feature flow) → Task 14.
- Criterion 10 (AGENTS.md heading) → Task 3 step 1.
- Criterion 11 (no old vocab remaining) → Task 17 step 1.

**Placeholder scan:** Every step has concrete strings or commands. No "TBD", "implement later",
or "add appropriate handling."

**Type consistency:** Cross-checked — `Change level` heading is used consistently across Tasks 3,
5, 8, 15, 16. The `<slug>` and `<name>` template placeholders inside code blocks are intentional
skill-template syntax, not implementation gaps.
