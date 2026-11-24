---
name: arch-change
description: Change docs/architecture/current/ — always asks first whether the change is new (adds something not yet in current/) or change (modifies, corrects, moves, or retires existing arch). Writes the current/ diff and the paired arch-log entry, stops for author review, hands tree-shape ops to /modularize --refine (default yes), gates the push on pre-push /arch-log-review, and opens the arch PR with the passing verdict as its first comment. Never writes a spec; for that use /new-spec.
disable-model-invocation: true
---

# /arch-change `<name>`

## Purpose

The single entry point for editing the solution architecture under
`docs/architecture/current/` — whether adding something new or changing, correcting, or retiring
something that already exists. Writes the `current/` diff and the paired arch-log entry at
`docs/architecture/logs/YYYY-MM-DD-<name>/README.md`, and opens the arch PR only after
`/arch-log-review` passes in pre-push mode.

This command does **not** write a feature spec. To spec the implementation of something already in
`current/`, use `/new-spec <name>`.

## Change types

Asked first, every run (step 2):

- **new** — adds something not yet in `current/`: a capability, bounded context, container,
  integration, stakeholder, information concept, value stream, NFR, assumption, constraint.
- **change** — modifies, corrects, moves, or retires something already in `current/`. A correction
  ("the doc was wrong") is a change whose rationale says so.

| | new | change |
|---|---|---|
| Target | duplicate check (step 4) | pick existing target(s) (step 4) |
| Q&A | what is added, driver, rationale, alternatives | what changes from → to, driver, rationale, alternatives (or "none — aligns doc with reality") |
| Log `Driver` | `new: <summary>` | `change: <summary>` |

Everything else is shared.

## Schema awareness

This command reads `@.claude/skills/check-setup/arch-schema.md` and routes edits to the correct
lens file(s) based on the subject of the change. Mapping (applies to add, modify, and retire alike):

| Subject | Files touched |
|---|---|
| Capability | `bizbok/capabilities.md` + `bizbok/value-streams.md` (if a journey is affected) + `glossary.md` (terms) |
| Bounded context | `ddd/context-map.md` + `ddd/contexts/<name>.md` |
| Service / container | `c4/containers.md` + `c4/deployment.md` (zone placement) + likely `ddd/contexts/<name>.md` + a modularity node (tree-shape op → handoff, step 9) |
| External integration | `c4/integrations.md` + `c4/context.md` |
| Access | `ddd/access-model.md` + likely `bizbok/stakeholders.md` (role) |
| Information concept | `bizbok/information-map.md` + `glossary.md` + likely a DDD context |
| Stakeholder | `bizbok/stakeholders.md` + `ddd/access-model.md` (≥1 policy per stakeholder, per cross-ref rule 1) |
| Value stream on an existing capability | `bizbok/value-streams.md` |
| Threat model / deployment topology | `c4/deployment.md` |
| Cross-cutting (NFR target, assumption, constraint, top-level topology narrative) | The corresponding top-level file — `nfrs.md`, `assumptions.md`, `constraints.md`, `glossary.md`, or top-level `README.md` |
| Purpose / Scope text of an existing L1/L2 node | That node's `modularity/<…>/README.md` (content only — no tree-shape op) |

**Fallthrough.** If the subject isn't in the table, consult
`@.claude/skills/check-setup/arch-schema.md` directly and pick the target file(s) whose schema
section the change touches. The table is a shortcut, not a whitelist.

**Cross-reference update rule.** A capability's capability→context/container map row references
existing (or newly-added) context + container files. A container's `### <Container>` subsection
links to a DDD context + a modularity node. When a change renames or retires something, update or
remove every cross-reference to it in the same diff.

**Multi-lens PR warning.** If the planned edits span all three lens dirs (bizbok/ + ddd/ + c4/),
emit during Q&A: "This change touches all three lens dirs. That is unusual — consider splitting it
into two arch changes. Continue anyway? [Y/n]".

**Gap discipline.** For every file created or edited, fill every required section you touch with
real content, or use the escape hatch `N/A because <reason>` (≥5 words). Never leave or introduce an
`ARCH_GAP` marker in a section this change writes.

**Diagram conventions.** When writing or updating a required diagram section, use the mermaid
syntax + template at `@.claude/skills/check-setup/arch-schema.md#diagram-conventions`. Every label
must pass the safe-label rules at `@.claude/skills/check-setup/arch-schema.md#safe-labels` —
substitute per the rule table rather than escape.

**Translate, don't invent.** When a change is described in service-vocabulary but a target file
demands DDD/BIZBOK/C4 vocabulary, translate (service with distinct data → DDD context; actor role →
stakeholder; external system → integration). Never fabricate facts the change doesn't imply.

**Tree shape is not ours.** This command never adds, renames, removes, or re-parents a modularity
node. Those are tree-shape ops owned by `/modularize --refine`, offered as the next step (step 9).

## Inputs

- `<name>` (required) — kebab-case codename for the arch change. Used as the arch-log entry slug
  (`YYYY-MM-DD-<name>`) and the branch name (`arch/<name>`).

## Steps

Other skills invoked below (`/modularize`, `/arch-log-review`) have
`disable-model-invocation: true` — "invoke" means read that skill's `SKILL.md` and follow it
in this session with the arguments given.

1. **Preflight.**
   - `<name>` is kebab-case. `git fetch origin`.
   - `git status --porcelain docs/architecture` must be empty — uncommitted arch edits would leak
     into this change (or, on resume, be skipped by the gate). Non-empty → list them, say "commit
     or amend first", and stop.
   - **Resume.** If branch `arch/<name>` exists and has a committed
     `docs/architecture/logs/*-<name>/README.md`, offer "Resume `<name>` at the arch-log gate?
     [Y/n]". Y → check out the branch; take `<codename>` from that log dir, the change type from
     its `Driver:` prefix, and the one-line title from its `# <name> — <title>` heading; jump to
     step 12. n → stop.
   - If `arch/<name>` exists **without** a committed log entry (an interrupted earlier run), stop:
     "finish or delete branch `arch/<name>` first".
   - Otherwise fix `<codename>` = `YYYY-MM-DD-<name>` (today); it must not exist under
     `docs/architecture/logs/` on `origin/<default>`.

2. **Ask the change type** — always, even if the user's message implies one:
   "Is this a **new** addition to the architecture, or a **change** to something already in it
   (modify, correct, move, retire)? [new/change]"

3. **Load context.** Read (skip any that don't exist):
   - `docs/architecture/current/README.md`, `nfrs.md`, `assumptions.md`, `constraints.md`,
     `glossary.md`.
   - Lens indexes: `bizbok/capabilities.md`, `bizbok/stakeholders.md`,
     `bizbok/information-map.md`, `ddd/context-map.md`, `c4/containers.md`, `c4/integrations.md`.
     Read further lens files (e.g. `ddd/contexts/<x>.md`) on demand once a target is known.
   - Modularity tree summary — walk `docs/architecture/current/modularity/**/README.md` up to depth
     2 (L1 and L2); extract each node's `id`, `title`, and Purpose section.

4. **Resolve the target.**
   - **new** → semantic match of `<name>` and the user's description against the loaded context.
     If something matching already exists, present it and ask: "This already exists as <X>. Switch
     to **change** on <X>? [Y/n]". Y → continue as **change** with <X> as the target. n → continue
     as **new** (related but distinct).
   - **change** → propose the existing target(s) (lens entries, NFR/assumption entries, nodes) that
     match the description; the user confirms or picks from a numbered list.

5. **Q&A**, one question at a time, per the type table above. Determine the subject(s) and the
   file list via the Schema awareness mapping. Run the multi-lens check on the file list.

6. **Create the branch** `arch/<name>` from `origin/<default>`.

7. **Write the diff to `current/`** — patch target files per the schema; create new
   `ddd/contexts/<name>.md` files with every required section filled or N/A; edit L1/L2 node
   Purpose/Scope text if in the file list. Apply the cross-reference update rule.

8. **Write the paired arch-log entry** at `docs/architecture/logs/<codename>/README.md`:

       # <name> — <one-line title>

       **Date**: YYYY-MM-DD
       **Codename**: <name>

       **Driver**: <new|change>: <one-line summary>

       **Decision**: <one-line summary of what changed in current/>

       **Rationale**: <captured from Q&A — one paragraph>

       **Alternatives rejected**: <list, one per line, with reason — or "None — this change aligns the doc with reality.">

       **Impact**:
       - <file 1 touched>
       - <file 2 touched>
       - ...

       **Links**:
       - (author may add related spec PRs or tracker items before merge)

9. **Tree-shape handoff.** Determine which tree-shape ops the change implies — e.g. a new service
   needs a new L1 node; a retired service needs its node removed; a moved responsibility may need a
   node renamed or a sibling added; a changed node purpose may need its children revisited. Prompt
   once per affected parent (never re-prompt one already offered); for root (L1 ops) omit the slug:

       Next: /modularize [<slug>] --refine --into-log <codename> — <reason>. Run it? [Y/n]

   Default **Y**. On Y, invoke `/modularize` with exactly those arguments; it appends its ops to
   this run's log entry. On n, print the command for later and continue. No tree-shape op implied
   → print "No tree-shape changes needed." and continue.

10. **Review checkpoint.** Show the full change, including new untracked files and anything the
    handoff wrote: run `git add -N docs/architecture` then print
    `git status --short docs/architecture` and `git diff origin/<default> -- docs/architecture`,
    plus the log entry. Ask: "Review the change. `ok` to continue, or tell me what to adjust."
    Adjustments → apply, update the log's Impact/Decision if affected, and ask again. Loop until
    `ok`.

11. **Commit locally** — stage only `docs/architecture/**` and commit
    (`arch: <name> — <one-line title>`). Do not push yet.

12. **Arch-log gate, before the push.** If a PR already exists for `arch/<name>`, stop here and
    tell the author to run `/arch-log-review` on-PR instead (it posts there). Otherwise invoke
    `/arch-log-review` with no PR — it runs in **pre-push mode** against the branch diff and
    returns its verdict into this session.
    - **`FAIL`** → **no push.** No PR, no comment, no comms notification. Print the verdict and its
      full findings verbatim and stop. Do not fix the findings yourself — the author edits, amends
      the commit, and re-runs `/arch-change <name>`, which resumes here (step 1).
    - **`PASS` / `PASS_WITH_ISSUES`** → prompt "Open the arch PR? [Y/n]" (default **Y**). On Y:
      push the branch, obtain the `gitHost` client per
      [Client resolution](../../../docs/process/README.md#client-resolution), open a PR against the
      default branch titled `arch: <name> — <one-line title>`, and **post the captured verdict
      verbatim as the PR's first comment**. Report any `PASS_WITH_ISSUES` findings in-session too.
      On n, stop and tell the author to re-run `/arch-change <name>` when ready — it resumes here
      and re-gates, so the posted verdict always matches the pushed commit.

13. If a comms channel is configured **and** step 12 opened the PR, obtain the `comms` client per
    [Client resolution](../../../docs/process/README.md#client-resolution) and post a short
    notification that the arch PR for `<name>` is open.

14. **Closing hint.** For type **new**:
    "After this merges, run `/new-spec <name>` to spec the implementation."

## Config

Reads `.claude/tracker.json` → `gitHost` (the docs repo lives on `gitHost`; used to open the arch
PR and post its verdict in step 12) and `comms` (optional notification only). Clients are obtained
per [Client resolution](../../../docs/process/README.md#client-resolution). Does not read `tracker`.

## What this command never does

- Never writes `docs/superpowers/specs/*` or an implementation plan.
- Never runs `superpowers:brainstorming` or `superpowers:writing-plans`.
- Never proposes a change level (C0/C1/C2) — that is a property of specs, not arch changes.
- Never adds, renames, removes, or re-parents a modularity node itself — always via the step 9
  handoff to `/modularize --refine`.
- Never checks downstream impact on specs or tracker issues — that is the PL's responsibility.
- Never pushes while `/arch-log-review`'s verdict is `FAIL`, and never posts a `FAIL` anywhere.
