# Modularity Skill — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add the `/modularize [slug]` skill that decomposes solution architecture into a
three-level tree of module nodes (`docs/architecture/current/modularity/`), plus its `--refine`
mode for add/edit/rename/remove, plus its automatic pairing with an arch-log entry per run.

**Architecture:** Docs-only. Adds one skill file, extends `docs/process/README.md` and repo
`README.md`. Sub-project (1) must be implemented first (arch home + arch-log-review). This skill
writes to `docs/architecture/current/modularity/` — which is under `current/` — so every run
triggers the arch-log-review rule from (1) automatically. The skill itself creates the paired log
entry to satisfy that rule.

**Tech Stack:** Markdown, YAML front-matter.

**Commit policy:** Per standing instruction, do not commit the spec or plan. The actual
implementation edits may be committed if the user opts in.

**Spec:** `docs/superpowers/specs/2026-08-12-modularity-skill-design.md`

**Prerequisite:** sub-project (1) plan complete (`architecture-log-review` agent exists,
`docs/architecture/current/` layout is established).

---

## File Structure

- **Create**:
  - `.claude/skills/modularize/SKILL.md` — the single skill file with the full flow.
- **Modify**:
  - `docs/process/README.md` — brief mention of `/modularize` in the tracker-adapter / process
    overview area.
  - `README.md` — add `/modularize` to the commands list.

No agent file (unlike sub-project 1) — decomposition happens inside the skill's interactive flow;
no separate read-only gate needed at PR time (arch-log-review from (1) covers PR-side enforcement).

---

## Task 1 — Create `/modularize` skill

**Files:**
- Create: `.claude/skills/modularize/SKILL.md`

- [ ] **Step 1: Write the skill file**

Create `.claude/skills/modularize/SKILL.md` with this content:

```markdown
---
name: modularize
description: Decompose a solution architecture node into children at the level below — Architecture → Service → Implementation. Reads architecture context, auto-proposes children, refines via Q&A, writes one README.md per child under docs/architecture/current/modularity/, and creates the paired arch-log entry. Supports --refine to add / edit / rename / remove children of an existing node.
disable-model-invocation: true
---

# /modularize `[<slug>] [--refine]`

## Purpose

Decompose the solution architecture into a tree of module nodes at three fixed levels:
Architecture (L1) → Service (L2) → Implementation (L3). Runs on user demand, one node at a time.
Writes files under `docs/architecture/current/modularity/` and always produces a paired
architecture-log entry (per `docs/process/AGENTS.md` and the `architecture-log-review` agent).

Tree is a **structural map only** — no state (assignees, sizing, milestones, deps, issue links).
Consumers (`/plan-sprint`, `/create-issues`, `/split-issue`) read the tree and layer state on top.

## Inputs

- `<slug>` (optional) — the target node's slug. Omit to target the root (`docs/architecture/current/modularity/`
  itself).
- `--refine` (optional flag) — operate on the target's existing children (add / edit / rename /
  remove) instead of producing fresh ones.

## Level rules

- **Root target** → produces L1 children (services / infra entities). Reads
  `docs/architecture/current/README.md` and files linked from it.
- **L1 target (`level: 1`)** → produces L2 children (contract modules within that service). Reads
  the target's `README.md` plus vault outlinks the user references.
- **L2 target (`level: 2`)** → produces L3 children (implementation shards for small-diff
  sub-issues). Reads the target's `README.md` plus parent chain.
- **L3 target (`level: 3`)** → refuses. Level 3 is terminal.
- **Root refine** (`/modularize --refine` with no slug) — operates on the root's direct children (L1 set).

## Steps

### Normal mode (no `--refine`)

1. **Locate the target.**
   - No arg → target = `docs/architecture/current/modularity/`. If the dir does not exist, create it
     as an empty container.
   - `<slug>` → find the dir matching the slug under
     `docs/architecture/current/modularity/` (search descendants). If multiple match, ask the user
     which one.
   - Read the target's `README.md` if present; parse front-matter to determine `level`.
2. **Refuse-with-hint on already-expanded nodes.** If the target already has subdirs (children),
   print `use --refine to modify existing children` and exit.
3. **Refuse on L3 targets.** If the target's front-matter has `level: 3`, print
   `L3 is the terminal level; no further decomposition` and exit.
4. **Load level-specific architecture context.**
   - Root → read `docs/architecture/current/README.md` and (up to depth 1) files it links to.
   - L1 → read the target's `README.md` and (up to depth 1) linked vault files.
   - L2 → read the target's `README.md` and its parent chain (L1 `README.md`, root `README.md`).
5. **Auto-propose N children.** From the loaded context, produce a proposal list, each with
   `{title, slug (draft), purpose (draft)}`. Slug format `[a-z0-9-]+`, kebab-case, max ~40 chars,
   unique within the target.
6. **Q&A refine, one at a time.** For each proposal, prompt keep / edit / drop. Edits prompt only
   for `title`, `slug`, `purpose`.
7. **Add-more loop.** After the last proposal, prompt "any missing children?". If yes, mini Q&A
   per new child (same three fields).
8. **Write child files.** For each accepted child, create the dir and `README.md`:

       docs/architecture/current/modularity/<target-path>/<child-slug>/README.md

   with front-matter:

       ---
       id: <child-slug>
       level: <target's level + 1, or 1 if target is root>
       parent: <target's slug, or "root" if target is root>
       title: <human title>
       ---

       # <title>

       ## Purpose
       <purpose text>

       ## Scope (in / out)
       <optional; empty if not filled during Q&A>

9. **Create the paired arch-log entry.** Write
   `docs/architecture/logs/YYYY-MM-DD-modularize-<target-slug>/README.md` (or
   `-modularize-root/` for root target) with:

       # modularize-<target-slug> — decomposed <target-title> into N children

       **Date**: YYYY-MM-DD
       **Codename**: modularize-<target-slug>

       **Driver**: /modularize run on <target-slug>

       **Decision**: decomposed <target-slug> into N children: <child-slug-1>, <child-slug-2>, ...

       **Rationale**: <captured from Q&A — one paragraph on why this shape>

       **Alternatives rejected**: <list of proposals the user dropped, one per line, with reason>

       **Impact**:
       - + modularity/<target-path>/<child-slug-1>/README.md
       - + modularity/<target-path>/<child-slug-2>/README.md
       - ...

       **Links**:
       - (user may add spec PRs, feature docs, or tracker items before commit)

10. **Print summary.** List of files written; the log-entry path; remind the user to review the
    diff and commit (the skill does not commit).

### `--refine` mode

1. **Locate the target** as in normal mode step 1.
2. **Refuse on L3 targets.** L3 targets have no children to refine.
3. **List existing children** as a numbered menu, showing `{slug, title}` per child.
4. **Prompt for an operation** — `add | edit <n> | rename <n> | remove <n> | done`. Loop until
   user picks `done`.
   - **add** — mini Q&A for a new child (title, slug, purpose); create the dir + README as in normal
     mode step 8.
   - **edit <n>** — prompt for new `title` and/or `purpose`; rewrite the child's `README.md`. Do
     not change `id` / `level` / `parent`.
   - **rename <n>** — prompt for new slug (validate format + uniqueness within parent); rename the
     child's dir; update the child's `id` field in front-matter; **cascade**: rewrite every
     descendant's `parent` field that referenced the old slug.
   - **remove <n>** — print the subtree preview (recursive tree of files under the child); prompt
     explicit `remove <slug>` confirmation; delete the subdir recursively.
5. **After the loop**, if any operation was performed, create a paired arch-log entry summarizing
   the ops (see log format in normal-mode step 9; `Driver` becomes `/modularize --refine run on
   <target-slug>`; `Impact` enumerates every add / edit / rename / remove).

## Config

Reads `.claude/tracker.json` for context only (no writes). Does not touch the tracker or the git
host. Writes only under `docs/architecture/current/modularity/` and `docs/architecture/logs/`.
```

- [ ] **Step 2: Verify**

Read the file back end-to-end. Confirm:
- Front-matter has `disable-model-invocation: true`.
- Purpose calls out "structural map only — no state".
- Level rules cover all four cases (root, L1, L2, L3).
- Steps sections have normal-mode (10 steps) and `--refine` mode (5 steps).
- Log-entry format matches the spec.

- [ ] **Step 3: Commit (optional)**

If user opts to commit: stage `.claude/skills/modularize/SKILL.md` and commit with message
`Add /modularize skill for three-level solution decomposition`.

---

## Task 2 — Mention `/modularize` in process README and repo README

**Files:**
- Modify: `docs/process/README.md`
- Modify: `README.md`

- [ ] **Step 1: Add a modularity section to `docs/process/README.md`**

Find the `## Where each gate happens` section end and the `## Onboarding (one-time)` section start.
Between them, insert a new section:

```
## Modularity (planning-time)

Between spec-approval and issue-creation, the team can decompose a feature's L2 module into L3
implementation shards using `/modularize <l2-slug>`. This produces tree files under
`docs/architecture/current/modularity/` with a paired arch-log entry per run — enforced by
`architecture-log-review`. Modularity is **optional per feature** and never runs automatically;
`/create-issues` will use L3 nodes when they exist (opt-in per feature) and otherwise creates one
issue per feature.

See `.claude/skills/modularize/SKILL.md` for the full flow. Three levels only: Architecture (L1) →
Service (L2) → Implementation (L3).
```

- [ ] **Step 2: Add `/modularize` to `README.md` commands list**

Find in `README.md`:

```
- `/setup-workspace` — provision service repos (create from template/exemplar or clone) and index them.
- `/propagate-rules` — push repo/language-specific rules into the service repos via PR.
- `/check-setup` — read-only audit of onboarding completeness.
```

Insert after these three lines (before `Then /new-feature`):

```
- `/modularize [<slug>]` — decompose the solution architecture into a three-level tree of module
  nodes; runs on demand per node.
```

- [ ] **Step 3: Verify**

Read both files, confirm the new content is present and formatted consistently with surrounding
sections.

- [ ] **Step 4: Commit (optional)**

Message: `Document /modularize in process README and repo README`.

---

## Self-review checklist

- **Spec coverage:**
  - Part 1 (directory layout) — handled at runtime by the skill (Task 1, step 8). ✅
  - Part 2 (node schema) — Task 1, step 8 defines the front-matter and body. ✅
  - Part 3 (invocation shape) — Task 1, Level rules + Steps. ✅
  - Part 4 (mechanic: hybrid decomposition + paired log entry) — Task 1, steps 1–10. ✅
  - Part 5 (`--refine` mode) — Task 1, `--refine` mode section. ✅
  - Part 6 (interaction with sub-project 1) — no code change needed; the skill writes under
    `current/` and arch-log-review from (1) automatically covers it. ✅
  - Part 7 (downstream consumers) — no task; consumers arrive in sub-projects (3), (4). ✅
  - Part 8 (ripples) — Tasks 1, 2. ✅
- **Placeholder scan:** none.
- **Type consistency:** skill name `modularize`, command `/modularize`, target dir
  `docs/architecture/current/modularity/`, log-dir prefix `modularize-<target-slug>` — consistent.

No gaps found. Plan is ready.
