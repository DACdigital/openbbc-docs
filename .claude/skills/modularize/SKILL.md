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
Consumers (`/plan-sprint`, `/split-issue`) read the tree and layer state on top. `/create-issues` reads only the ROADMAP row, not the tree.

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

       Container: [<name>](../../c4/containers.md#<anchor>) · Context: [<name>](../../ddd/contexts/<name>.md)

       ## Purpose
       <purpose text>

       ## Scope (in / out)
       <optional; empty if not filled during Q&A>

   The soft-convention header line (`Container: … · Context: …`) is required by the arch schema
   at `@.claude/skills/check-setup/arch-schema.md#modularity-header` **only for L1 children**
   (root's direct children, at `modularity/<L1>/README.md`). Skip the header line entirely when
   writing L2 or L3 children — their relative paths would differ, and the schema doesn't require
   the header at those depths.

   If the Q&A did not identify a matching container or context yet (e.g. the tree is being
   seeded before `c4/containers.md` is populated), write the L1 header with plain-text
   placeholders: `Container: <TBD> · Context: <TBD>`. `/check-setup` warns on absence but does
   not fail; a `<TBD>`-form line counts as "present" for rule 11 and clears the warning.
   Upgrade `<TBD>` placeholders later by hand-editing the L1 README once the corresponding
   container/context files exist.

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
   - **add** — mini Q&A for a new child (title, slug, purpose; also container link + context
     link when adding an **L1** child under root — may be `<TBD>` if not yet mapped; skip
     these two fields when adding L2 or L3 children); create the dir + README as in normal
     mode step 8, including the soft-convention header line for L1 children only.
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
