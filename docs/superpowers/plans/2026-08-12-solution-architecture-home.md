# Solution-Architecture Home — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add `docs/architecture/` as a mandatory home for the project's target solution
architecture, an append-only log format for arch changes, an agent/command that enforces the log
rule on PRs, and the corresponding `/check-setup` + AGENTS.md updates.

**Architecture:** Docs-only. Adds one review agent (mirrors existing spec-review /
architecture-review), one command wrapper skill, and extends four existing files (setup-project,
check-setup, process/README, process/AGENTS, README). No code, no tests.

**Tech Stack:** Markdown, YAML front-matter for skills/agents.

**Commit policy:** Per standing instruction, do not commit the spec or plan. The actual
implementation edits (skill/agent/process files) may be committed as normal work if the user opts
in — plan includes commit steps as guidance, not requirement.

**Spec:** `docs/superpowers/specs/2026-08-12-solution-architecture-home-design.md`

---

## File Structure

- **Create**:
  - `.claude/agents/architecture-log-review.md` — read-only review agent for the paired-log-entry rule.
  - `.claude/skills/arch-log-review/SKILL.md` — user-invoked command; invokes the agent, posts the verdict.
- **Modify**:
  - `.claude/skills/setup-project/SKILL.md` — Phase 2 gains an "Import solution architecture (adopt-or-scaffold)" step + writes initial arch-log entry. Keeps the rest of Phase 2 (estimate ingest + ROADMAP seed) intact — that gets reworked in sub-project (3).
  - `.claude/skills/check-setup/SKILL.md` — adds two checks: "Architecture home present" and "Initial log entry present". Total check count goes 7 → 9.
  - `docs/process/README.md` — mention `docs/architecture/` in the "Onboarding" section and add a one-liner about the new arch-log gate.
  - `docs/process/AGENTS.md` — two new hard rules: "Architecture changes require a log entry" and "Solution-architecture home is mandatory".
  - `README.md` — add `docs/architecture/` to the repo-layout section.

No test files. Verification is by re-reading modified files.

---

## Task 1 — Create the `architecture-log-review` agent

**Files:**
- Create: `.claude/agents/architecture-log-review.md`

- [ ] **Step 1: Write the agent file**

Create `.claude/agents/architecture-log-review.md` with this content:

```markdown
---
name: architecture-log-review
description: Reviews any PR that touches docs/architecture/current/ for the paired log-entry rule — must add exactly one new docs/architecture/logs/YYYY-MM-DD-<codename>/README.md with the required fields, matching the changes to current/. Emits a PASS/PASS_WITH_ISSUES/FAIL verdict for the invoking command to post as a comment on the PR. Use when /arch-log-review is invoked on a PR that touches current/.
tools: Read, Grep, Glob
---

# Architecture Log Review

You are the **architecture-log-review** gate for this repo's spec-driven delivery process. You run
on any PR that touches `docs/architecture/current/`. You **read and judge only** — never edit,
never approve or merge. Your final message is the verdict; the invoking command posts it to the
PR.

## Inputs

Read, in this order:

1. `docs/process/README.md` and `docs/process/AGENTS.md` — recall the append-only log rule and the
   required fields.
2. `docs/superpowers/specs/2026-08-12-solution-architecture-home-design.md` — the design that
   defines the log format and the enforcement rules (Parts 3 and 5).
3. The PR diff for the PR under review.

## Checks

Run these in order and stop at the first fail (still emit the verdict for what you saw):

1. **Log entry added.** Does the PR add exactly one new
   `docs/architecture/logs/YYYY-MM-DD-<codename>/README.md`? Not zero, not two.
2. **Directory name convention.** Does the new log's dir name match `YYYY-MM-DD-<codename>` where
   codename is lowercase kebab-case (`[a-z0-9-]+`, max ~40 chars)?
3. **Required fields present.** Does the new log's `README.md` include all required fields — Date,
   Codename, Driver, Decision, Rationale, Alternatives rejected, Impact, Links — in the format
   defined in the spec (Part 3)?
4. **Impact matches diff.** Does the `Impact` field list the specific `current/` files the PR
   touched? Vague "everything under current/" without a real reason fails.
5. **Append-only invariant.** Are past log entries under `docs/architecture/logs/` untouched? Any
   edit or delete to existing log entries is a fail.

## Verdict format

Emit a single message with:

- **Verdict:** `PASS` / `PASS_WITH_ISSUES` / `FAIL`
- **Findings:** bulleted list per failed check, with the exact rule violated and a pointer to the
  offending file / line.
- **Fixes:** one-liner per finding — the specific edit needed.
```

- [ ] **Step 2: Verify**

Read `.claude/agents/architecture-log-review.md` back. Confirm:
- Front-matter has `name`, `description`, `tools: Read, Grep, Glob`.
- Body has Inputs / Checks / Verdict-format sections.
- The 5 checks are stated in the order above.

---

## Task 2 — Create the `/arch-log-review` command skill

**Files:**
- Create: `.claude/skills/arch-log-review/SKILL.md`

- [ ] **Step 1: Write the skill file**

Create `.claude/skills/arch-log-review/SKILL.md` with this content:

```markdown
---
name: arch-log-review
description: On-demand review of a PR that touches docs/architecture/current/ — invokes the architecture-log-review agent, posts the verdict as a PR comment. Independent of L1/L2 — any PR touching current/ needs the paired log entry.
disable-model-invocation: true
---

# /arch-log-review `[<pr-url>]`

## Purpose

Runs `architecture-log-review` on a PR that touches `docs/architecture/current/`, posts the
verdict as a PR comment. Independent of feature-level gates (`spec-review`, `architecture-review`,
`contract-data-event-review`) — a pure refactor of `current/` with no feature spec still needs
this check.

## Inputs

- `<pr-url>` (optional) — full PR/MR URL. If omitted, use the current git branch's open PR/MR from
  the configured git-host.

## Steps

1. **Resolve target PR** — either from `<pr-url>` or the current branch's open PR (via git-host
   MCP).
2. **Fetch PR diff** — via git-host MCP; extract the file list.
3. **Guard: does PR touch `docs/architecture/current/`?** If no, print "no architecture changes;
   skipping" and exit with no verdict.
4. **Invoke** the `architecture-log-review` agent with the PR diff as input.
5. **Post the verdict** as a PR comment via git-host MCP.

## Config

Reads `.claude/tracker.json` → `gitHost` (platform + org). Read-only against `docs/`; writes only a
PR comment.
```

- [ ] **Step 2: Verify**

Read the file back. Confirm the front-matter has `disable-model-invocation: true` and the Steps
section reflects the 5 steps above.

---

## Task 3 — Extend `/setup-project` Phase 2 with arch adopt-or-scaffold

**Files:**
- Modify: `.claude/skills/setup-project/SKILL.md`

- [ ] **Step 1: Read the current Phase 2 section**

Read lines 51–62 of `.claude/skills/setup-project/SKILL.md`. Confirm the current three-step Phase 2
(fill CLAUDE.md → ingest estimate → seed ROADMAP) is intact.

- [ ] **Step 2: Insert the arch adopt-or-scaffold step**

Find in `.claude/skills/setup-project/SKILL.md`:

```
### Phase 2 — Project context

1. Fill the `CLAUDE.md` project-context section: project name, stack, tracker line (`platform`, `org`,
   `projectId`), git-host line (`platform`, `org`), comms line (if configured). Leave the service-repo
   list for Phase 3.
2. **Ingest the approved estimate** — via the estimate MCP if available (list/get the approved
   estimation), else ask the user for the export — pulling the feature/epic list, priorities, NFRs, and
   documented assumptions.
```

Replace with:

```
### Phase 2 — Project context

1. Fill the `CLAUDE.md` project-context section: project name, stack, tracker line (`platform`, `org`,
   `projectId`), git-host line (`platform`, `org`), comms line (if configured). Leave the service-repo
   list for Phase 3.
2. **Import solution architecture (adopt-or-scaffold).** Prompt the user: "Do you have an existing
   solution-architecture set to import?" [y/n].
   - **y (adopt)** — accept a local dir path, a git URL to clone from, or an Obsidian vault export
     path. Copy contents into `docs/architecture/current/`. Verify a `README.md` exists at the root;
     if not, prompt the user to name (or promote) one before continuing.
   - **n (scaffold)** — write a starter `docs/architecture/current/README.md` with sections
     `## Context`, `## Components`, `## Data`, `## Deployment`, `## Open questions`, each with a
     placeholder line. Halt Phase 2 until the user has filled at least one section with real
     (non-placeholder) content.
   - **Log the initial state** — create
     `docs/architecture/logs/YYYY-MM-DD-init/README.md` with `Decision: initial architecture
     recorded`, `Driver: project setup`, `Impact: all of current/`, `Alternatives rejected: none`
     (see the log format defined in `docs/superpowers/specs/2026-08-12-solution-architecture-home-design.md`
     Part 3).
3. **Ingest the approved estimate** — via the estimate MCP if available (list/get the approved
   estimation), else ask the user for the export — pulling the feature/epic list, priorities, NFRs, and
   documented assumptions.
```

Note: subsequent step numbering (previously `3.` for seed ROADMAP) becomes `4.`; renumber it.

- [ ] **Step 3: Renumber the third step**

Find the following in Phase 2 (was step 3, now needs to become step 4):

```
3. **Seed `ROADMAP.md`** — one row per feature/epic in the `Feature | Priority | Level | Spec | Issues |
   Milestone | Link` table (`Level` and later columns start empty). Add short "Non-functional
   requirements" and "Assumptions" sections below the table.
```

Change the leading `3.` to `4.`.

- [ ] **Step 4: Read back the whole Phase 2 section**

Confirm:
- Step 1: Fill CLAUDE.md
- Step 2: Import solution architecture (adopt-or-scaffold + log initial state)
- Step 3: Ingest the approved estimate
- Step 4: Seed ROADMAP.md
- No stray numbering or blank lines.

- [ ] **Step 5: Commit (optional)**

If user opts to commit: stage `.claude/skills/setup-project/SKILL.md` and commit with message
`Extend /setup-project Phase 2 with arch adopt-or-scaffold`.

---

## Task 4 — Add two checks to `/check-setup`

**Files:**
- Modify: `.claude/skills/check-setup/SKILL.md`

- [ ] **Step 1: Read current check list**

Read `.claude/skills/check-setup/SKILL.md` fully. Current check count is 7 (numbered 1–7 under
`## Checks`).

- [ ] **Step 2: Add two new checks after check 3 (Docs-repo identity)**

Find:

```
3. **Docs-repo identity** — `git remote -v` origin is not the template, and the docs repo exists on the
   git host. Fix: `/setup-project` (Phase 1).
4. **Repos cloned & indexed** — every repo has a `workspace/<repo>/` checkout, a `docs/repos/<repo>.md`
   index, and a row in `workspace/INDEX.md`. Fix: `/setup-workspace` (or `--refresh`).
```

Replace with:

```
3. **Docs-repo identity** — `git remote -v` origin is not the template, and the docs repo exists on the
   git host. Fix: `/setup-project` (Phase 1).
4. **Architecture home present** — `docs/architecture/current/README.md` exists with
   non-placeholder content (no `TBD`-only or `<...>`-only sections; at least one filled section).
   Fix: `/setup-project` (Phase 2).
5. **Initial arch-log entry present** — at least one `docs/architecture/logs/YYYY-MM-DD-*/README.md`
   exists. Fix: `/setup-project` (Phase 2).
6. **Repos cloned & indexed** — every repo has a `workspace/<repo>/` checkout, a `docs/repos/<repo>.md`
   index, and a row in `workspace/INDEX.md`. Fix: `/setup-workspace` (or `--refresh`).
```

- [ ] **Step 3: Renumber checks 5 → 7, 6 → 8, 7 → 9**

Find and update each in turn:
- `5. **Tracker project**` → `7. **Tracker project**`
- `6. **ROADMAP seeded**` → `8. **ROADMAP seeded**`
- `7. **Rules propagated**` → `9. **Rules propagated**`

- [ ] **Step 4: Update the Output section summary**

Find:

```
A checklist, one line each `✅|⚠️|❌ <check> — <detail> [fix: <command>]`, then a one-line summary
`N/7 checks pass`.
```

Replace `N/7` with `N/9`.

- [ ] **Step 5: Read the file end-to-end**

Confirm:
- Checks are numbered 1 → 9 with no gaps.
- New checks 4 and 5 are architecture-related.
- Output summary says `N/9 checks pass`.

- [ ] **Step 6: Commit (optional)**

Message: `Add architecture checks to /check-setup`.

---

## Task 5 — Add two hard rules to `docs/process/AGENTS.md`

**Files:**
- Modify: `docs/process/AGENTS.md`

- [ ] **Step 1: Read current Hard rules section**

Read `docs/process/AGENTS.md` from the `## Hard rules` heading to the end. Confirm the existing 5
hard rules (bulleted list) are intact.

- [ ] **Step 2: Append two new rules at the end of the hard-rules bullet list**

Find the last existing rule:

```
- **`decisions.md` is the one exception, and it IS committed.** It is a decision log, not a review
  verdict — record decisions there as they are made, in every feature folder.
```

Append after it (still inside the same bullet list):

```
- **Architecture changes require a log entry.** Any PR that modifies `docs/architecture/current/`
  must add a new `docs/architecture/logs/YYYY-MM-DD-<codename>/README.md` in the same PR, with all
  required fields (Driver / Decision / Rationale / Alternatives rejected / Impact / Links) present.
  `architecture-log-review` enforces this.
- **The solution-architecture home is mandatory.** From Phase 2 onward,
  `docs/architecture/current/README.md` must exist with non-placeholder content, and at least one
  entry must exist in `docs/architecture/logs/`. `/check-setup` reports the absence of either as a
  fail.
```

- [ ] **Step 3: Verify**

Read the Hard rules section back. Confirm two new bullets appear at the end and the existing 5
rules are untouched.

- [ ] **Step 4: Commit (optional)**

Message: `Add architecture-log-review hard rules to AGENTS.md`.

---

## Task 6 — Mention `docs/architecture/` in `docs/process/README.md` and repo README

**Files:**
- Modify: `docs/process/README.md`
- Modify: `README.md`

- [ ] **Step 1: Update the Onboarding phases table in `docs/process/README.md`**

Find:

```
| 2 Context | fill `CLAUDE.md`; ingest the estimate; seed `ROADMAP.md` |
```

Replace with:

```
| 2 Context | fill `CLAUDE.md`; import solution architecture (adopt-or-scaffold); ingest the estimate; seed `ROADMAP.md` |
```

- [ ] **Step 2: Update the `## Where each gate happens` section**

Find:

```
- **Code gate (all implemented work)** — happens in the **service repos**, via normal human PR review.
  Not this repo's concern.
```

Insert a new bullet immediately before it:

```
- **Architecture-log gate (any PR touching `docs/architecture/current/`)** — happens **here**, on
  the PR that changes the arch. `/arch-log-review` posts its verdict as a PR comment. Independent
  of L1/L2 — a pure refactor still triggers it.
```

- [ ] **Step 3: Update `README.md` repo-layout section**

Find in `README.md`:

```
docs/process/         the process — L0/L1/L2, gates, per-feature flow (tool-neutral source of truth)
docs/conventions/      cross-repo standards (architecture, naming, api, persistence, events, testing)
```

Insert immediately after these two lines:

```
docs/architecture/    this project's target solution architecture (current/) + change log (logs/)
```

- [ ] **Step 4: Verify**

Read both files, confirm changes are as above.

- [ ] **Step 5: Commit (optional)**

Message: `Document docs/architecture/ in process README and repo README`.

---

## Self-review checklist

- **Spec coverage:**
  - Part 1 (directory layout) — no task; layout is created by `/setup-project` at runtime, not by us
    at implementation. ✅ (documented in Task 3 flow).
  - Part 2 (`current/README.md` required) — Task 3 (scaffold) + Task 4 (check-setup). ✅
  - Part 3 (log entry format) — Task 1 (agent enforces it) + Task 3 (scaffold init entry). ✅
  - Part 4 (Phase 2 flow) — Task 3. ✅
  - Part 5 (agent + command) — Tasks 1, 2. ✅
  - Part 6 (`/check-setup`) — Task 4. ✅
  - Part 7 (AGENTS.md rules) — Task 5. ✅
  - Part 8 (ripples) — Tasks 3, 4, 5, 6. ✅
- **Placeholder scan:** no TBD / "add appropriate error handling" / "similar to Task N". Every
  step names the exact text to change and the exact replacement.
- **Type consistency:** file paths, agent name (`architecture-log-review`), command name
  (`/arch-log-review`), and skill name (`arch-log-review`) are consistent across all tasks.

No gaps found. Plan is ready.
