# Solution-Architecture Home

**Date:** 2026-08-12
**Status:** Approved design, pre-implementation
**Part of:** larger initiative to make project intent explicit and to enable just-in-time
modularization. Sub-project (1) of five (see below).

## Purpose

Add a mandatory home for **this project's solution architecture** — the *current target*, the
artifacts that describe it, and an append-only log of decisions that change it. Today, `/setup-project`
Phase 2 seeds `ROADMAP.md` from an "approved estimate", but the estimate is a task list — not a
definition of the finished product. Engineering has no place to answer *what are we building?*
beyond the feature-by-feature view in the ROADMAP.

This is distinct from:

- `docs/conventions/architecture.md` — *cross-project rules* (how DAC builds systems in general).
- `docs/<feature>/decisions.md` — *feature-scoped* decision log, per-feature.

The new artifact is *this project's* solution architecture, at project scope.

## Larger initiative — decomposition

This spec is sub-project (1) of five, agreed with the user on 2026-08-12:

1. **Solution-architecture home** (this doc).
2. **Modularity skill** — multi-level decomposition (architecture → services → contract modules →
   implementation shards) driving issue planning and code-review sharding.
3. **Sprint-driven ROADMAP + Phase 2 rework** — estimate becomes optional, architecture mandatory,
   ROADMAP grows milestone-by-milestone via re-runs of the modularity skill.
4. **Sub-issue / code-review-sharding mechanism** — modularity level-3 output flows into
   `/create-issues` and the tracker adapters.

Order: (1) → (2) → (3) → (4). This spec is a hard prerequisite for (2)–(4).

## Scope

- **In:** the on-disk layout (`docs/architecture/current/`, `docs/architecture/logs/`); required
  entry-point file; required log-entry format; Phase 2 adopt-or-scaffold flow; a new
  `architecture-log-review` agent and `/arch-log-review` command; `/check-setup` additions; two new
  hard rules in `docs/process/AGENTS.md`.
- **Out (YAGNI):** any machine-readable manifest in `current/`; CI/git-hook enforcement; automated
  diagram generation; automated vault summarization; cross-link/broken-link validation; support for
  editing past log entries.

## Writing principle

Concise, technical, complete. The process docs are the source of truth; `.claude/` skills and
agents are the execution adapter.

## Part 1 — Directory layout

    docs/architecture/
    ├── current/                          # free-form; the "vault"
    │   └── README.md                     # required entry point
    └── logs/                             # append-only change log
        └── YYYY-MM-DD-<codename>/
            ├── README.md                 # required
            └── <artifacts>               # optional: diagrams, exports, screenshots, before/after

`current/` semantics: the **current target architecture** — what the team has decided the project is
building toward *right now*. Not "what's running in prod today". Log entries record decisions that
change the target.

Contents of `current/` are free-form. Teams may keep an Obsidian vault, `.drawio` files, images,
sub-notes, or any mix. Only the entry point is standardized.

## Part 2 — `current/README.md` (required entry point)

Free-form Markdown. It's the "home note" — must link out to whatever else exists in `current/`
(component pages, diagrams, data-model files, deployment layout). Minimum content is deliberately
loose; the entry point exists so consumers — skills (modularity, new-feature brainstorming), humans,
`/check-setup` — have a stable starting file.

`/check-setup` treats an empty or placeholder-only `README.md` (only `TBD` / `<...>` tokens, no real
prose) as a fail.

## Part 3 — `logs/YYYY-MM-DD-<codename>/README.md` (required fields)

Parallel to per-feature `decisions.md` (see `docs/process/AGENTS.md`), with one extra field for
architecture scope:

    # <codename> — <one-line title>

    **Date**: YYYY-MM-DD
    **Codename**: <slug, matches dir name>

    **Driver**: <what prompted this change — external event, refactor need, feature spec, incident, modularity-skill output>

    **Decision**: <what was decided, one or two sentences>

    **Rationale**: <why this choice over alternatives>

    **Alternatives rejected**: <what else was considered and why it lost>

    **Impact**: <bulleted list of files/pages in `current/` that this change touches, and any downstream service repos affected>

    **Links**:
    - <spec PR / feature doc / tracker item / external doc that motivated or documents this>

Rules:

- **Codename**: short human handle in the dir name, e.g. `2026-08-12-split-billing-context/`. Format:
  lowercase kebab-case, `[a-z0-9-]+`, max ~40 chars. The date prefix keeps ordering; codename need
  not be globally unique.
- **Append-only**: log directories, once merged, are never edited or deleted (same rule as
  `decisions.md`). Corrections come as new entries that supersede.
- Additional artifacts (diagrams, exports, screenshots, before/after images) live alongside the
  `README.md` in the same dir as loose files.

## Part 4 — Phase 2 setup flow (adopt-or-scaffold)

Inserted into `/setup-project` Phase 2, before the (in sub-project 3, optional) estimate ingestion:

1. **Ask** the user whether they have an existing solution-architecture set to import.
2. **If yes (adopt)** — accept one of:
   - a local directory path;
   - a git URL to clone from;
   - an Obsidian vault export path.

   Copy contents into `docs/architecture/current/`. Verify a `README.md` exists at the root; if not,
   prompt the user to name (or promote) one before continuing.
3. **If no (scaffold)** — write a starter `docs/architecture/current/README.md` with these skeleton
   sections:

       # <project> — solution architecture (target)

       ## Context
       <what problem does this solve; who uses it>

       ## Components
       <services / packages / infra entities and their responsibilities>

       ## Data
       <ownership, key entities, storage>

       ## Deployment
       <where and how it runs>

       ## Open questions
       <things not yet decided>

   Halt Phase 2 until the user has filled at least one section with real content. The check is the
   "non-placeholder heuristic" — no `TBD`, no `<...>`-only content, at least one non-empty section.
4. **Log the initial state** — create `docs/architecture/logs/YYYY-MM-DD-init/README.md`:
   `Decision: initial architecture recorded`, `Driver: project setup`, `Impact: all of current/`,
   `Alternatives rejected: none`. Establishes the append-only chain from day zero.

## Part 5 — Enforcement: `architecture-log-review` agent + `/arch-log-review` command

Following the pattern of `spec-review`, `architecture-review`, and `contract-data-event-review`:

- **`.claude/agents/architecture-log-review.md`** — a read-only review agent.
  - **Trigger**: any PR that touches `docs/architecture/current/`.
  - **Checks** (in order):
    1. Does the PR add exactly one new `docs/architecture/logs/YYYY-MM-DD-<codename>/README.md`?
    2. Does the new log's dir name match the `YYYY-MM-DD-<codename>` convention?
    3. Does the new log's `README.md` include all required fields (Date, Codename, Driver, Decision,
       Rationale, Alternatives rejected, Impact, Links)?
    4. Does the `Impact` field list the specific `current/` files the PR changed (no vague
       "everything under current/" without a real reason)?
    5. Are past log entries under `logs/` untouched (append-only invariant)?
  - **Verdict**: PASS / PASS_WITH_ISSUES / FAIL, posted as a PR comment by the invoking command.
  - **Read-only**: never edits files, never approves or merges.
- **`.claude/skills/arch-log-review/SKILL.md`** — the command. Invoked manually by the reviewer of a
  PR that touches `current/`. Parallel to `/spec-review` in shape.

Independence from feature-level gates:

- Not tied to L1/L2 classification — a pure technical refactor of `current/` with no feature spec
  still triggers `/arch-log-review`.
- When the driver *is* an L2 feature spec, the same PR carries the spec, the `current/` diff, and
  the paired log entry. `spec-review`, `architecture-review`, `contract-data-event-review`, and
  `architecture-log-review` all post their verdicts on the same PR side-by-side.

## Part 6 — `/check-setup` integration

Add two checks to `.claude/skills/check-setup/SKILL.md`:

- **Architecture home present** — `docs/architecture/current/README.md` exists and has
  non-placeholder content (see the non-placeholder heuristic from Part 4). Fix: `/setup-project`
  (Phase 2).
- **Initial log entry present** — at least one `docs/architecture/logs/YYYY-MM-DD-*/README.md`
  exists. Fix: same.

The existing check-list count moves from 7 to 9.

## Part 7 — Hard rules to add to `docs/process/AGENTS.md`

Two new rules under the "Hard rules" section:

- **Architecture changes require a log entry.** Any PR that modifies `docs/architecture/current/`
  must add a new `docs/architecture/logs/YYYY-MM-DD-<codename>/README.md` in the same PR, with all
  required fields present. `architecture-log-review` enforces this.
- **The solution-architecture home is mandatory.** From Phase 2 onward,
  `docs/architecture/current/README.md` must exist with non-placeholder content, and at least one
  entry must exist in `docs/architecture/logs/`. `/check-setup` reports the absence of either as a
  fail.

## Part 8 — Ripple summary (files this spec's implementation will touch)

- **Add**: `docs/architecture/current/README.md` (skeleton, if scaffolded);
  `docs/architecture/logs/YYYY-MM-DD-init/README.md`; `.claude/agents/architecture-log-review.md`;
  `.claude/skills/arch-log-review/SKILL.md`.
- **Modify**: `.claude/skills/setup-project/SKILL.md` (Phase 2 flow);
  `.claude/skills/check-setup/SKILL.md` (two new checks); `docs/process/README.md` (mention the new
  gate + the new artifact home); `docs/process/AGENTS.md` (two new hard rules); `README.md` (repo
  layout section — add `docs/architecture/`).
- **Consume** (in later sub-projects): the modularity skill (sub-project 2) reads
  `current/README.md` as its top-level input; `/new-feature` brainstorming (existing) may read
  `current/` for context.

## Open questions

None blocking implementation. Two low-priority follow-ups:

- Should the scaffold skeleton be templated further (per stack / per repo type)? Punt until sub-project
  2 clarifies what the modularity skill needs.
- Should `/arch-log-review` be auto-invoked by CI on relevant PRs, rather than run manually? Punt —
  the manual invocation matches the rest of the process today.
