# Split `/new-feature` into `/new-feature` (arch) + `/new-spec` (spec) — plus C-prefixed change class

**Date:** 2026-09-01
**Status:** Approved design, pre-implementation
**Builds on:** Unified Arch Log + Layer Discipline (sub-project 5, merged `f54183b`). That work
fused arch changes into `/new-feature`; this spec undoes the fusion by cleanly separating the
architecture-evolution and spec-initialization concerns, and takes the opportunity to fix a
long-standing letter collision between two independent "level" axes.

## Business value / Why

The current `/new-feature` conflates two workflows: (a) evolving the solution architecture
(adding/changing `docs/architecture/current/`) and (b) writing an implementation spec. In practice
these have different actors, different rhythms, and different gates. Fusing them:

- Makes every spec run ask "does this touch `current/`?", including for features that plainly
  don't.
- Makes architecture evolution invisible unless a spec triggers it — there is no way to add a
  capability strategically without pretending it is also a feature spec.
- Removes any guard against speccing something that is not yet in the architecture (novelty specs
  get written, arch never catches up).
- Removes any guard against adding duplicate arch content (nothing checks if the "new" capability
  already exists as a module).

Additionally, the two axes both labelled `L0/L1/L2` (change classification for specs vs. tree
depth for modularity) cause reader confusion — the same letters mean different things depending
on context.

The split addresses all four problems. The rename removes the letter collision.

Success: after this ships, a reader of `docs/process/README.md` can tell in one pass which
command to run for a given intent, and cannot mistake a change-class label for a tree-depth
label.

## Change level (C0/C1/C2)

**C2 — architectural process change.** Alters the docs-repo command surface (adds `/new-spec`,
repurposes `/new-feature`), the axis vocabulary (`L0/L1/L2` → `C0/C1/C2` for change class), and
the per-feature flow described in `docs/process/`. Multiple review gates and existing skills/agents
reference the changed axis and must be updated in lockstep.

Note: this spec introduces the C-prefixed axis it uses in this section header. The C-vocabulary
lands in the codebase as part of this spec's Part 3; other in-flight specs written before this
one still use the L-prefixed axis for change class, and are left as historical artifacts
(see Risks).

## Scope

**In:**

- Repurpose `/new-feature <name>` to evolve `docs/architecture/current/` only (non-tree content
  plus L1/L2 tree adds); produces an **arch PR** gated by `/arch-log-review`.
- Add `/new-spec <name> [--issue <tracker-id>]` to initialize a feature spec from an existing
  arch node; produces a **spec PR** gated by `/spec-review` (plus `/arch-review` and
  `/contract-data-event-review` for C2 specs).
- Novelty guard in `/new-spec`: resolve-then-confirm; block with instruction to run `/new-feature`
  first when no mapping to an existing `current/` node exists.
- Duplicate-check step in `/new-feature`: semantic match against existing `current/` content and
  modularity-tree nodes before writing.
- Modularity-impact assessment at end of `/new-feature`: verdict + optional suggestion, never
  auto-runs `/modularize --refine`.
- Rename change classification from `L0/L1/L2` to `C0/C1/C2` across `.claude/`, `docs/process/`,
  `docs/conventions/`, `README.md`, `CLAUDE.md`.
- Add a **Levels** section to `README.md` and `CLAUDE.md` explaining both axes side-by-side
  (L1/L2/L3 tree depth + C0/C1/C2 change class).
- Update `docs/process/README.md` per-feature flow diagram and prose for the two entry commands
  and the "arch PR merges before spec PR" ordering when both apply.

**Out (YAGNI):**

- Any tooling to auto-detect when a spec should have been an arch change. The novelty guard
  blocks only when the user cannot map; it does not judge whether the mapping is a stretch.
- Migration tool for downstream projects. The split is command-surface only — no data migration
  is needed; users update muscle-memory and re-read the process doc.
- Any change to `/modularize`, `/arch-log-review`, `/spec-review`, `/arch-review`, or
  `/contract-data-event-review` beyond the C-prefix rename.
- Changing the arch-log format or modularity tree structure.
- Retro-classifying already-merged specs from L to C. They stand as historical artifacts.
- Any changes to spec.md file format beyond section-2's heading rename.

## Contracts

No API / data / event contracts affected. This is a docs-repo internal process/tooling change.
All contract-facing artefacts (`.claude/tracker.json`, PR templates, tracker adapters) remain
unchanged in shape; only the vocabulary they reference (C0/C1/C2 vs. L0/L1/L2) shifts.

Skill / agent surface changes:

- `/new-feature <name>` — signature unchanged; behavior redirected to arch-evolution.
- `/new-spec <name> [--issue <tracker-id>]` — new command.
- Skill files under `.claude/skills/{new-feature,new-spec,spec-review,arch-review,contract-data-event-review,create-issues,plan-sprint,split-issue,split-pr}/SKILL.md`
  — vocabulary changes; the first two also change behavior.
- Agent files under `.claude/agents/{spec-review,arch-review,contract-data-event-review}.md`
  — vocabulary changes. `architecture-log-review.md` requires no change (it does not reference
  the change-class axis).

## Acceptance criteria

1. `/new-feature <name>` produces an arch PR containing only `docs/architecture/current/**` and
   `docs/architecture/logs/YYYY-MM-DD-<name>/README.md` changes. It never creates
   `docs/<name>/spec.md`.
2. `/new-feature` performs a duplicate check before writing and presents any overlapping existing
   `current/` or modularity content, giving the user the choice to abort in favour of
   `/modularize --refine`.
3. `/new-feature` emits a modularity-impact verdict at the end — either "no refine needed" or
   "suggest running `/modularize --refine <slug>` on X, Y" — and does not auto-run `/modularize`.
4. `/new-spec <name>` produces a spec PR containing only `docs/<name>/spec.md` (and optionally
   `plan.md`). It never writes to `docs/architecture/current/` or `docs/architecture/logs/`.
5. `/new-spec` resolves the mapped `current/` node from `<name>` argument, `--issue` ticket,
   current-conversation context, or modularity-tree fuzzy match; asks only when ambiguous or
   absent; blocks with the correct instruction when no mapping is possible.
6. Mid-brainstorm scope drift into net-new arch halts `/new-spec` with the same block message.
7. Every occurrence of `L0`, `L1`, or `L2` as **change classification** in `.claude/` and `docs/`
   is renamed to `C0`, `C1`, `C2`. The `L1/L2/L3` **modularity tree** axis is untouched.
8. `README.md` and `CLAUDE.md` contain a **Levels** section with both axes explained side-by-side.
9. `docs/process/README.md`'s per-feature flow shows two entry commands; when both are needed,
   the arch PR is described as merging before the spec PR is opened.
10. `docs/process/AGENTS.md` required-spec-sections list uses **Change level (C0/C1/C2)** for
    section 2.
11. No skill or agent file references the old change-class vocabulary after the rename.

## Risks & assumptions

**Risks:**

- **Grep-and-replace over-reach.** The rename could touch strings that happen to contain "L0",
  "L1", or "L2" for unrelated reasons (quoted examples, path fragments, modularity references).
  Mitigation: rename by an explicit file-and-string list built during the plan step, not by blind
  `sed`. Discipline: rename only where paired with words like "level", "spec", "trivial",
  "normal", "architectural", "fast lane", "route", "escalate", "reviewer", or in a table row
  about change types.
- **In-flight specs use old vocabulary.** The six 2026-08 specs under `docs/superpowers/specs/`
  reference `L0/L1/L2` as change class throughout. Decision: leave existing merged specs
  untouched (historical); the rename applies to `.claude/` execution surface + `docs/process/` +
  `docs/conventions/` + repo READMEs. In-flight plans not yet merged may adopt C-vocabulary on
  rebase.
- **Command muscle-memory.** Users used to `/new-feature` for spec init will run the arch-
  evolution command by accident. Mitigation: `/new-feature` prints a one-line "did you mean
  `/new-spec`?" hint at start, and `/new-spec` prints the reverse hint.
- **Novelty guard false blocks.** A legitimate spec might be blocked because the arch node
  exists but the user cannot articulate the mapping in the confirmation prompt. Mitigation: the
  `change` option in the confirmation UI lets the user pick from a list of existing modules;
  `none` is the only path to a block.

**Assumptions:**

- The current fused `/new-feature` has no live production users outside this template repo (no
  downstream projects have adopted the sub-project (5) version yet — it is on `master` here but
  external adoption is unknown).
- `/modularize --refine` remains the correct tool for L2→L3 decomposition; this spec does not
  alter it.
- The C0/C1/C2 letters are acceptable. Alternatives considered — T-prefix ("type") and K-prefix
  ("kind") — rejected as less mnemonic; "C for change" reads naturally.
- No tracker or PR-template exports depend on the literal string "L0/L1/L2" as a machine-
  parseable label. (Verified: no such exports exist in this repo.)
- The `<name>` argument to `/new-spec` and `/new-feature` follows the same kebab-case convention
  as today; a `<name>` may match a modularity slug directly or be freshly chosen.

## Part 1 — `/new-feature` repurpose

`.claude/skills/new-feature/SKILL.md` gets a full rewrite. Approximate step list:

1. **Validate `<name>`** — kebab-case, and `YYYY-MM-DD-<name>` unique as an arch-log codename for
   today's date.
2. **Print reverse-hint**: "If you meant to write an implementation spec for an existing
   capability, use `/new-spec <name>` instead."
3. **Load context**: `current/README.md`, `nfrs.md`, `assumptions.md`, and a modularity tree
   summary (walk `current/modularity/**/README.md`, extract front-matter + Purpose, up to L2).
4. **Duplicate check**: semantic match of `<name>` and the user's initial description against
   loaded context. Present overlaps with three options:
   - (a) same as X → abort, print "run `/modularize --refine edit` on X".
   - (b) related but distinct → continue.
   - (c) unrelated → continue.
5. **Q&A**: what changes (which files under `current/`), driver, rationale, alternatives
   considered.
6. **Write the diff to `current/`**:
   - For L1/L2 tree adds → create the dir + `README.md` with modularity front-matter (`id`,
     `level`, `parent`, `title`) and the standard Purpose / Scope sections.
   - For non-tree edits (NFRs, assumptions, README/topology) → patch the target files directly.
7. **Write paired arch-log entry** at `docs/architecture/logs/YYYY-MM-DD-<name>/README.md` with
   all required fields per `docs/process/AGENTS.md` and `architecture-log-review`:
   Driver / Decision / Rationale / Alternatives rejected / Impact / Links.
8. **Modularity-impact assessment**: identify which touched `current/` paths sit inside
   `modularity/<slug>/` where `<slug>` has children. Emit one of two verdicts:
   - "No refine needed — no existing decomposed nodes were touched."
   - "Suggest running `/modularize --refine <slug>` on: X, Y. Reason: <purpose changed | new
     sibling added | scope narrowed>."
   Never auto-run `/modularize`.
9. **Open the arch PR**: branch, commit `docs/architecture/**` changes only, push, open PR
   against the default branch titled after `<name>`. `/arch-log-review` will comment.
10. If a comms channel is configured, post a short notification that the arch PR for `<name>` is
    open.

The command **never** writes `docs/<name>/spec.md`, `plan.md`, or any file outside
`docs/architecture/`.

## Part 2 — `/new-spec` new skill

Create `.claude/skills/new-spec/SKILL.md`. Approximate step list:

1. **Print reverse-hint**: "If you meant to add a new capability to the solution architecture,
   use `/new-feature <name>` instead."
2. **Resolve mapping** to a `current/` node:
   - `<name>` argument matches a modularity slug → mapped node = that slug.
   - `--issue <tracker-id>` present → pull the ticket via the configured tracker adapter, extract
     title and description; try to match against modularity nodes; the ticket body also seeds the
     brainstorm.
   - Prior conversation clearly names a module (only when the skill is invoked mid-conversation
     rather than in a fresh session) → propose that mapping.
   - Exactly one plausible modularity node by fuzzy match on `<name>` → propose that mapping.
   - None of the above → treat as ambiguous.
3. **Confirm or ask**:
   - Confidently mapped → report: "This spec will attach to `<slug>` (L`<level>`, `<path>`).
     Continue? `[Y/n/change]`."
   - `change` selected, or ambiguous (zero or multiple matches) → present candidate list with a
     `none` option.
   - `none` chosen or no matches at all → **BLOCK** with:
     > "This looks like a novelty. Run `/new-feature <name>` first to add it to
     > `docs/architecture/current/`, then re-run `/new-spec <name>`."
4. **Load context**: mapped node's `README.md` + parent chain (walk `parent` back to root),
   `current/nfrs.md`, `current/assumptions.md`; if a tracker ticket was pulled, include its body.
5. **Run `superpowers:brainstorming`** with the required-sections override from
   `docs/process/AGENTS.md`: Business value / Why, Change level (C0/C1/C2), Scope (in/out),
   Contracts, Acceptance criteria, Risks & assumptions.
6. **Mid-brainstorm drift safeguard**: if the design surfaces net-new arch not covered by the
   mapped node (e.g., proposes a whole new service, or a new contract on a different service),
   halt with the same block message as step 3.
7. **Write `docs/<name>/spec.md`** with the six required sections.
8. **Propose the change level (C0/C1/C2)** per `docs/process/README.md`'s change classification,
   with a one-line justification. Write it into the spec's "Change level (C0/C1/C2)" section.
   Ambiguous → pick higher. Never route through the C0 fast lane if the change touches DB, API,
   events, security, tenant isolation, or an upgrade.
9. **Optionally run `superpowers:writing-plans`** to produce `docs/<name>/plan.md` from the
   approved spec (implementation detail; not reviewed).
10. **Open the spec PR**: branch, commit `docs/<name>/{spec.md[,plan.md]}` only, push, open PR
    against the default branch titled after `<name>`. `/spec-review` will comment; if C2 is
    proposed, `/arch-review` and `/contract-data-event-review` also apply.
11. If a comms channel is configured, post a short notification that the spec PR for `<name>` is
    open.

The command **never** writes to `docs/architecture/current/` or `docs/architecture/logs/`.

## Part 3 — C-prefixed change class rename

Global rename `L0/L1/L2` → `C0/C1/C2` **only where the axis is change classification**. Explicit
file-and-string list to be built during the plan step; illustrative here:

- `docs/process/README.md`:
  - Change-classification table row labels `L0 — trivial` / `L1 — normal` / `L2 — architectural`
    → `C0 — trivial` / `C1 — normal` / `C2 — architectural`.
  - Prose references: "L2 spec", "L1/L2", "L0-only fast lane" → C-prefixed equivalents.
  - Per-feature flow diagram: `[L2] /arch-review + /contract-data-event-review` → `[C2] …`.
- `docs/process/AGENTS.md`:
  - Required-spec-sections item 2 heading `Level (L0/L1/L2)` → `Change level (C0/C1/C2)`.
  - Every hard-rule mention (`L0/L1`, `L1/L2`, `L0-only`, etc.) → C-prefixed.
- `.claude/skills/spec-review/SKILL.md`, `arch-review/SKILL.md`,
  `contract-data-event-review/SKILL.md`, `create-issues/SKILL.md`, `plan-sprint/SKILL.md`,
  `split-issue/SKILL.md`, `split-pr/SKILL.md`: rename every change-class reference.
- `.claude/agents/spec-review.md`, `arch-review.md`, `contract-data-event-review.md`: rename
  every change-class reference. `architecture-log-review.md` unchanged.
- `README.md`, `CLAUDE.md`: add the **Levels** section (see Part 4). Also rename any inline
  change-class references.
- `docs/conventions/*.md`: grep for change-class references, update if found.

**Rename discipline** — do NOT touch any occurrence where `L1/L2/L3` refers to the modularity
tree (inside `/modularize`, `/create-issues`, modularity-tree prose). Match `L0`, `L1`, or `L2`
only where paired with "level", "spec", "trivial", "normal", "architectural", "fast lane",
"route", "escalate", "reviewer", or in a table row about change types.

## Part 4 — Levels section for `README.md` and `CLAUDE.md`

New section titled **Levels** added identically to both files (CLAUDE.md may simply link back to
README.md's section to avoid drift). Draft body:

> This project uses two independent "level" axes. Do not confuse them.
>
> **Architecture tree levels (L1 / L2 / L3)** — depth in the modularity tree under
> `docs/architecture/current/modularity/`. Managed by `/modularize`.
>
> | Level | Name | What it holds |
> |-------|------|---------------|
> | **L1** | Architecture | services / infra entities (children of the tree root) |
> | **L2** | Service | contract modules within a service |
> | **L3** | Implementation | issue-sized shards to feed `/create-issues` |
>
> Only L1 and L2 are architecture-of-record; L3 is planning-time decomposition stored in the
> same tree for convenience.
>
> **Change classification (C0 / C1 / C2)** — how much process a change goes through.
>
> | Level | Name | When | Process |
> |-------|------|------|---------|
> | **C0** | trivial | typo, comment, one-liner, safe patch bump | fast lane — no spec |
> | **C1** | normal | typical functional change; no arch/DB/API/event impact | spec → `/spec-review` → issues → code PR |
> | **C2** | architectural | new bounded context; DB/API/event/contract change; tenant isolation; framework upgrade | C1 + `/arch-review` + `/contract-data-event-review` |
>
> **The two axes are independent.** An L2 module (Service level) may host either a C1 or a C2
> change; an L1 addition is usually a C2 change but not by definition.

## Part 5 — Process doc updates

`docs/process/README.md`:

- Replace the per-feature flow block with a two-entry version:

      select feature (from ROADMAP or ad-hoc)
        │
        ├─ if capability not yet in architecture:
        │    /new-feature   → docs/architecture/current/** + docs/architecture/logs/…
        │                   → arch PR → /arch-log-review → human merge  ← GATE
        │
        ├─ /new-spec        → docs/<feature>/spec.md
        │                   → spec PR → /spec-review [+ /arch-review, /contract-data-event-review for C2]
        │                   → human merge  ← GATE
        │
        ├─ /create-issues   → parent issue in tracker; opt-in sub-issues from L3 tree if present
        ├─ /roadmap-sync
        └─ devs implement in workspace/<repo>, code PRs merged
             └─ /split-issue <id> if PRs get too big

- Add a short subsection "Two entry points" explaining when to use `/new-feature` vs.
  `/new-spec`, and that when both apply the arch PR merges before the spec PR is opened.
- Rename change-class references to C-prefix (see Part 3).

`docs/process/AGENTS.md`:

- Rename change-class references to C-prefix.
- Restate rules as:
  - "The fast lane is **C0-only**."
  - "Ambiguous **change level** → pick the higher one."
  - "A reviewer can raise the **change level**, never lower it."
- No new hard rules for the split itself — the skill files carry the enforcement (novelty guard,
  duplicate check, PR-content discipline). The process doc merely narrates the flow.

## Part 6 — Ripples (file list)

**Modify (existing):**

- `.claude/skills/new-feature/SKILL.md` — full rewrite for the arch-evolution role.
- `.claude/skills/spec-review/SKILL.md` — C-prefix rename.
- `.claude/skills/arch-review/SKILL.md` — C-prefix rename.
- `.claude/skills/contract-data-event-review/SKILL.md` — C-prefix rename.
- `.claude/skills/create-issues/SKILL.md` — C-prefix rename.
- `.claude/skills/plan-sprint/SKILL.md` — C-prefix rename.
- `.claude/skills/split-issue/SKILL.md` — C-prefix rename.
- `.claude/skills/split-pr/SKILL.md` — C-prefix rename.
- `.claude/agents/spec-review.md` — C-prefix rename.
- `.claude/agents/arch-review.md` — C-prefix rename.
- `.claude/agents/contract-data-event-review.md` — C-prefix rename.
- `docs/process/README.md` — flow diagram, prose, table, C-prefix rename.
- `docs/process/AGENTS.md` — required-sections heading, hard rules, C-prefix rename.
- `README.md` — Levels section, inline C-prefix rename if any.
- `CLAUDE.md` — Levels section (or link), inline C-prefix rename if any.
- `docs/conventions/*.md` — C-prefix rename where change-class references appear.

**Create (new):**

- `.claude/skills/new-spec/SKILL.md`.

**Delete:** none.

Total: ~16 modified files + 1 new file.

## Open questions

None blocking implementation.
