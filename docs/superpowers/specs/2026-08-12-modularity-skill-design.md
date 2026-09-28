# Modularity Skill

**Date:** 2026-08-12
**Status:** Approved design, pre-implementation
**Depends on:** `docs/superpowers/specs/2026-08-12-solution-architecture-home-design.md` (sub-project 1
of this initiative). This is sub-project (2) of five (see the (1) spec for the full decomposition).

## Purpose

A skill that decomposes a project's solution architecture into a tree of modules, at three fixed
levels — **Architecture → Service → Implementation**. Enables:

- Splitting large scopes into reviewable, ship-able pieces.
- Ordering and prioritising work across a team.
- Growing the plan **just-in-time**: only expand a node when its work is imminent. The tree is a
  structural artifact; downstream planning (ROADMAP, milestones, tracker issues) reads it and layers
  state on top — the tree itself carries no assignments, deps, milestones, or estimates.
- Recording every structural change to the plan via the same append-only architecture log defined in
  sub-project (1) — every `/modularize` run produces a paired log entry, so the log becomes the
  audit trail for architectural evolution.

## Scope

- **In:** the `docs/architecture/current/modularity/` tree layout; node README.md schema; the
  `/modularize [slug]` command with its level-aware behavior; the hybrid decomposition mechanic
  (auto-propose then Q&A refine); the `--refine` mode covering add / edit / rename / remove; the
  automatic pairing with an `arch-log` entry every run; the level-specific context each invocation
  reads.
- **Out (YAGNI):** the planning command that sets `status: done` (postponed — this spec scaffolds
  the field so the planning command can slot in later); any placement of assignees, dependencies,
  milestones, sizing, or issue links inside the tree; multi-parent nodes; automated node-move
  detection (moves happen as remove-then-add); batched invocation across all nodes at a level;
  automated `status` inference from tracker state.

## Writing principle

Concise, technical, complete. `docs/architecture/current/modularity/` is a machine-consumable
structural artifact; the skill is its execution adapter.

## Part 1 — Directory layout

The tree lives **inside** the arch vault, so any modularity change is automatically covered by the
`arch-log-review` rule from sub-project (1) — no rule broadening needed.

    docs/architecture/current/
    ├── README.md                              # vault entry point (from sub-project 1)
    ├── ...                                    # free-form vault contents (Obsidian, drawio, etc.)
    └── modularity/                            # the tree — this directory is the root
        ├── <l1-slug>/
        │   ├── README.md
        │   └── <l2-slug>/
        │       ├── README.md
        │       └── <l3-slug>/
        │           └── README.md
        └── ...

`docs/architecture/current/modularity/` **is the root** — its direct child directories are L1 nodes.
No `root/README.md`; the container is implicit.

## Part 2 — Node schema

Every node's `README.md` has YAML front-matter followed by Markdown body:

    ---
    id: <slug — matches the containing dir name>
    level: 1 | 2 | 3
    parent: <parent-slug | root>
    title: <human-readable name>
    status: done                     # optional; absent = open
    ---

    # <title>

    ## Purpose
    <one paragraph: what this module IS and why it exists>

    ## Scope (in / out)
    <optional; what's inside this module's boundary vs. outside>

**Rules:**

- **Slug format**: `[a-z0-9-]+`, kebab-case, lowercase, max ~40 chars. Unique within a parent.
- **Children**: not listed in front-matter or body. Auto-derived from the filesystem — the direct
  subdirs of `<node>/`.
- **`status: done` semantics**:
  - On a **leaf** (no subdirs) — set by the future planning command when a human confirms all
    created issues are closed AND those issues covered the module's requirements. Not set by
    `/modularize` itself.
  - On a **non-leaf** — either explicit for clarity, or computed by consumer tooling as `all
    children have status: done`.
- **JIT expansion state** is implicit — a node with no subdirs is not yet decomposed; a node with
  subdirs is expanded. No `expanded` flag needed.
- Nothing else in the tree — no assignees, sizes, deps, milestones, issue links, or arbitrary
  metadata. Those are layered by external systems (tracker, ROADMAP, per-feature specs).

## Part 3 — Invocation

    /modularize            # decompose root → produces L1 children (services / infra entities)
    /modularize <slug>     # decompose <slug> → produces the level-below children

The skill reads the target's `level` (or `root` if no arg) and picks its behavior:

- **Root target → produce L1 children.** Reads `docs/architecture/current/README.md` and any files
  it links out to. Proposes services / infra entities that make up the target architecture.
- **L1 target → produce L2 children.** Reads the L1 node's `README.md` and the parts of the vault
  it references. Proposes contract modules for that service (e.g. "users endpoints", "products
  endpoints", "base API middleware").
- **L2 target → produce L3 children.** Reads the L2 node's `README.md` and its parent chain.
  Proposes implementation shards sized for small-diff sub-issues (e.g. "GET /users/{id}", "user
  input validation", "user model migration").
- **L3 target → refuses.** Level 3 is the terminal level; L3 nodes are not decomposed further, and
  `--refine` on an L3 node also refuses (no children to refine; adding children would violate the
  strict three-level rule).
- **Root refine (`/modularize --refine`)** operates on the root's direct children (the L1 set).

**Re-run on an already-expanded node (without `--refine`):** refuses by default and prints the
existing children with the hint `use --refine to modify`.

## Part 4 — Mechanic: hybrid decomposition

Per invocation on a target `T`:

1. **Load context.** Read `T`'s `README.md` (if not root) and the appropriate architecture context
   for `T`'s level (see Part 3).
2. **Auto-propose.** Produce N proposed children with `{title, slug, purpose (draft)}`. Level-aware:
   L1 proposes services; L2 proposes contract modules; L3 proposes small implementation shards.
3. **Q&A refine, one at a time.** For each proposal, ask the user: **keep / edit / drop**. Edits
   prompt only for `title`, `slug`, `purpose`. `status` is never set at this stage.
4. **Add-more loop.** After walking every proposal, ask "any missing children?". If yes, mini Q&A
   per new child (same three fields).
5. **Write files.** For every accepted child: create `docs/architecture/current/modularity/<...>/<T-path>/<child-slug>/README.md`
   with the schema from Part 2. `parent` is set to `T`'s slug (or `root`).
6. **Create the paired log entry.** Under `docs/architecture/logs/YYYY-MM-DD-modularize-<T-slug>/README.md`:
   - `Driver`: `/modularize run on <T-slug>` (plus any refactor context the user supplies inline).
   - `Decision`: "decomposed `<T-slug>` into N children: `<slug1>`, `<slug2>`, …".
   - `Rationale`: captured from the Q&A (why this decomposition, what was rejected).
   - `Alternatives rejected`: proposals the user dropped.
   - `Impact`: bulleted list of files written (`+ modularity/<T-path>/<child-slug>/README.md`).
   - `Links`: empty by default; user can add before commit.

   The user can edit the draft log entry before committing. The skill only writes files; the human
   reviews the diff, commits, and opens the PR that `arch-log-review` will review.

## Part 5 — `--refine` mode: add / edit / rename / remove

`/modularize <slug> --refine` opens a Q&A on `<slug>`'s existing children:

- **Add** — same mini-Q&A as Part 4 step 4 (title, slug, purpose).
- **Edit** — change `title` / `purpose` in the child's `README.md`. Non-destructive.
- **Rename** — change the child's dir name AND the child's `id` field AND cascade: every
  descendant's `parent` field that referenced the old slug is rewritten. Skill performs the cascade
  atomically so the tree stays consistent. Git records this as a delete+add + updates to child
  READMEs.
- **Remove** — delete the child's subdir with its entire subtree. Skill prints the subtree preview
  and requires explicit confirmation before deleting. Destructive; the paired log entry captures
  the *why*.

**Move (change parent)** is intentionally NOT a first-class operation. To move a node, use `remove`
under the old parent then `add` under the new one. The paired log entry documents the intent as
one refactor.

The paired log entry generated by any `--refine` run enumerates every change under `Impact`:

    Impact:
    - removed: modularity/service-a/ (subtree: 3 L2 children, 7 L3 leaves)
    - added: modularity/service-b/
    - added: modularity/service-c/
    - renamed: modularity/service-x/ → modularity/service-users/ (cascaded 4 descendants)

## Part 6 — Interaction with sub-project (1)

- `docs/architecture/current/modularity/` is under `current/`, so the `arch-log-review` rule from
  (1) — "any PR that touches `docs/architecture/current/` must add exactly one new log entry" —
  covers modularity PRs unchanged. **No spec ripple to (1).**
- The initial log entry created by `/setup-project` Phase 2 (spec (1), Part 4 step 4) covers the
  initial vault + any initial modularity that gets scaffolded alongside. Subsequent `/modularize`
  runs each produce their own log entry.

## Part 7 — Downstream consumers

This spec only defines the tree and its editor. Consumers arrive in later sub-projects:

- **Sub-project 3 (sprint-driven ROADMAP + Phase 2 rework)** reads the tree to answer: which L1/L2
  nodes are placeholders (no subdirs → need modularizing before their milestone starts); which have
  `status: done` (completeness rollup).
- **Sub-project 4 (sub-issue sharding)** reads L3 leaves per parent to create small-diff sub-issues
  under a feature issue.

Both sub-projects link back into the tree via slugs (referencing `<slug>` in issue titles or spec
front-matter). The tree itself remains free of forward links to these consumers.

## Part 8 — Ripple summary (files this spec's implementation will touch)

- **Add**: `.claude/skills/modularize/SKILL.md`; `.claude/agents/modularize.md` (if the mechanic is
  split into a decomposition subagent — TBD in implementation plan).
- **Modify**: `docs/process/README.md` (mention the new skill under a "Modularity" heading).
- **Consume** (in later sub-projects): the tree is read by sub-projects (3) and (4).

Sub-project (1) is NOT modified by this spec; its rules cover modularity by placement alone.

## Open questions

None blocking implementation. Two follow-ups worth noting:

- Whether to render an ASCII/HTML tree overview from the filesystem for humans/reviewers. Punt —
  the visual companion we used during design shows this works well ad-hoc; a persistent view can
  come later.
- Whether the auto-propose step should use a subagent (fresh context, level-specific prompt) or run
  in-line. Detail for the implementation plan.
