# /create-issues as a planning-time issue creator

**Date:** 2026-09-09
**Status:** Approved design, pre-implementation

## Business value / Why

Today `/create-issues` is gated on a merged spec PR — it reads the spec's Scope/Contracts,
proposes a parent + sub-issue decomposition, and creates the whole tree in the tracker in one shot.
In practice DMs and PLs create tickets **during planning**, well before any spec exists: a sprint
is picked, features are agreed, tickets are needed so the sprint has a home in the tracker from
day one. The current gate forces two bad choices — either the tracker stays empty until the first
spec merges (delivery visibility gap), or DMs create tickets by hand outside the flow (tooling
drift).

This spec reshapes `/create-issues` into a single-purpose planning-time command: **one call
creates one tracker issue for one feature**. Nothing about hierarchy, nothing about specs,
nothing about decomposition. `/split-issue` already handles decomposition when it becomes
necessary; `/create-issues` and `/split-issue` become clean, single-purpose commands with no
overlap.

## Change level (C0/C1/C2)

**C1.** Process reshape across three skills + one process doc. Drops a gate but introduces no
new API/data/event/contract; architecture layer untouched.

## Scope (in / out)

**In**

- `.claude/skills/create-issues/SKILL.md` — remove the spec-merged gate, remove spec-reading,
  remove L3-derived sub-issue materialization. Steps become: resolve ROADMAP row → create one
  tracker issue → update the row's `Issues` + `Link`. Rewrite `description` frontmatter.
- `.claude/skills/plan-sprint/SKILL.md` — final step prompts "Create tracker issues for these N
  features now? [Y/n]" and delegates to `/create-issues` on Y, once per newly appended row.
- `.claude/skills/split-issue/SKILL.md` — no behavior change. Add one paragraph documenting that
  L3 tree paths (`docs/architecture/current/modularity/<...>/<l2-slug>/`) are valid scope
  grounding context when handed in by the user during planning — the model uses them for the
  auto-propose step, same way it uses `--pr`. No new flag; existing `--groups` covers the
  machine-consumable path.
- `docs/process/README.md` — per-feature flow diagram reorders: planning → `/create-issues` →
  `/new-spec` → spec PR gate → devs implement → `/split-issue <id>` **only if** scope needs
  splitting. Move the `/create-issues` bullet from its current post-spec position into planning.

**Out**

- Renaming any skill. `/create-issues` name stays; the semantic tightening happens in the
  frontmatter and the Purpose section.
- Any change to `/split-issue`'s auto-propose / Q&A / `--groups` / `--refine` mechanics.
- Any change to `docs/process/AGENTS.md`. The existing "docs → tracker is one-way" and
  "do not suggest `/roadmap-sync` outside planning" rules already cover the boundary this
  reshape reinforces.
- Backward-compat shims for the old gate. Downstream projects pick up the new behavior via
  `/sdd-rebase`; the old file is replaced, not deprecated in place. Tracker items created under
  the old flow are unaffected.
- Any change to `/roadmap-sync`, `/status`, `/modularize`, `/new-spec`, or `/new-feature`.

## Contracts

- **API / HTTP:** none.
- **Data model:** none. Tracker payloads unchanged — `/create-issues` still writes one work
  item / issue via the same adapter branches (plane / github / gitlab), just without the
  spec-derived children under it. `ROADMAP.md` columns unchanged.
- **Events:** none.

The change is purely a workflow / process one — skill frontmatter, step order, and one process
doc.

## Acceptance criteria

1. `/create-issues <name>` runs successfully when only a `ROADMAP.md` row for `<name>` exists —
   no spec file present, no spec PR open or merged. It creates exactly one tracker issue and
   fills the row's `Issues` + `Link` columns.
2. `/create-issues <name>` fails with a clear error when no `ROADMAP.md` row for `<name>` exists.
   (This is the only gate remaining.)
3. `/plan-sprint` on new rows ends with the "Create tracker issues for these N features now?
   [Y/n]" prompt. On Y, one `/create-issues` call is issued per newly appended row, sequentially.
   On n, no tracker writes happen; `/create-issues <name>` is still callable standalone later.
4. `/split-issue` unchanged in behavior — auto-propose, Q&A, `--refine`, `--groups`, `--dry-run`
   all work as before. The new documentation paragraph on L3 tree paths is present but does not
   change any code path or flag.
5. `docs/process/README.md` per-feature flow diagram places `/create-issues` in the planning
   section, before `/new-spec`; `/split-issue` appears as a conditional post-spec-merge branch
   ("if scope needs splitting"), not in the default flow.
6. No skill or process file mentions `/create-issues` as a post-spec follow-up (to `/new-spec`,
   `/spec-review`, or spec-merge). Every surviving mention refers to the planning-time semantics.
   The "opt-in L3 sub-issues" flow is gone from `create-issues/SKILL.md` entirely.
7. `grep -rn '/create-issues' .claude/ docs/` returns only references consistent with the new
   framing.

## Risks & assumptions

**Risks**

- **Downstream projects mid-workflow.** A downstream project may be in the middle of using
  `/create-issues` post-spec — after `/sdd-rebase` merges the new behavior, the gate is gone
  and the L3 sub-issue prompt is gone. Users who relied on that opt-in flow now need to run
  `/split-issue` explicitly. Mitigated by: (a) documenting the change clearly in the
  `/sdd-rebase` PR body, (b) `/split-issue`'s existing L3-friendly conversational path.
- **`/plan-sprint` bulk-create at scale.** If a DM plans a large sprint (say 20 features) and
  answers Y, twenty sequential tracker writes happen. This matches the existing `/split-issue`
  discipline of "one at a time so numbering is stable" and is acceptable; documented in the
  `/plan-sprint` step. A partial failure mid-loop leaves some tickets created and others not —
  the ROADMAP row's `Issues` column reflects reality (created rows filled, uncreated rows
  blank), and the DM re-runs `/plan-sprint --resume` or `/create-issues <name>` per gap.

**Assumptions**

- Every planned feature has a `ROADMAP.md` row before `/create-issues` is called. `/plan-sprint`
  ensures this; standalone use also assumes it.
- The tracker adapter (plane / github / gitlab) is configured in `.claude/tracker.json`. This is
  a `/setup-project` precondition, already enforced by `/check-setup`.
- L3 tree nodes, when present, live under `docs/architecture/current/modularity/<...>/<l2-slug>/`.
  Established by `/modularize`, unchanged here.
- Users who need sub-issue decomposition know to invoke `/split-issue <id>` explicitly. The
  per-feature flow diagram in `docs/process/README.md` makes this visible.
