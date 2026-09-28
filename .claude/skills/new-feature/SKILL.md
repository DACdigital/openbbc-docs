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

## Schema awareness

This command reads `@.claude/skills/check-setup/arch-schema.md` and routes edits to the correct
lens file(s) based on the feature kind. Mapping:

| Feature kind | Files touched |
|---|---|
| New capability | `bizbok/capabilities.md` + `bizbok/value-streams.md` (if it adds a journey) + `glossary.md` (new terms) |
| New bounded context | `ddd/context-map.md` + new `ddd/contexts/<name>.md` |
| New service / container | `c4/containers.md` + `c4/deployment.md` (zone placement) + likely a new `ddd/contexts/<name>.md` + a new `modularity/<slug>/` node |
| New external integration | `c4/integrations.md` + `c4/context.md` |
| Access change | `ddd/access-model.md` + likely `bizbok/stakeholders.md` (new role) |
| New information concept | `bizbok/information-map.md` + `glossary.md` + likely a DDD context change |
| New stakeholder (standalone) | `bizbok/stakeholders.md` + `ddd/access-model.md` (≥1 policy per stakeholder, per cross-ref rule 1) |
| New value stream on existing capability | `bizbok/value-streams.md` |
| Threat model / deployment topology change | `c4/deployment.md` |
| Cross-cutting change (NFR target, assumption, constraint, top-level topology narrative) | The corresponding top-level file — `nfrs.md`, `assumptions.md`, `constraints.md`, `glossary.md`, or top-level `README.md` |

**Fallthrough.** If the feature kind isn't in the table, consult
`@.claude/skills/check-setup/arch-schema.md` directly and pick the target file(s) whose
schema section the change touches. The mapping table is a shortcut for common kinds, not an
exhaustive whitelist.

**Cross-reference update rule.** When adding a capability, its capability→context/container
map row references existing (or newly-added) context + container files. When adding a
container, its `### <Container>` subsection links to a DDD context + a modularity node.

**Multi-lens PR warning.** A single feature that legitimately touches all three lens dirs
(bizbok/ + ddd/ + c4/) is rare. If the planned edits span all three, emit this warning during
Q&A: "This change touches all three lens dirs. That is unusual — consider whether it should be
split into two features (e.g. add the capability first, then add the service). Continue anyway?
[Y/n]".

**Gap discipline.** For every new file created (e.g. a new `ddd/contexts/<name>.md`), fill
every required section with real content, or use the escape hatch `N/A because <reason>`
(≥5 words). Never leave an `ARCH_GAP` marker on a newly-added file — `ARCH_GAP` is a
scaffolder's placeholder for unknowns, not a valid state for a feature-add.

**Diagram conventions.** When writing or updating a required diagram section (e.g. the
system-at-a-glance in `README.md`, or the C4/DDD diagrams in `c4/context.md`,
`c4/containers.md`, `c4/data-flows.md`, `c4/deployment.md`, `ddd/context-map.md`), use the
mermaid syntax + template at `@.claude/skills/check-setup/arch-schema.md#diagram-conventions`.
Same convention across all projects; don't invent new diagram styles per feature.

**Translate, don't invent.** When an arch change is described in service-vocabulary but a
target file demands DDD/BIZBOK/C4 vocabulary, translation is expected — extract the
equivalent in the target's vocabulary (service with distinct data → DDD context; actor
role → stakeholder; external system → integration). Never fabricate facts the change
doesn't imply.

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
   - **Identify feature kind** from the Q&A (step 5) and consult the mapping table in the
     Schema awareness section. Touch exactly the lens file(s) listed for that kind.
   - **For lens-file edits** — patch the target files directly, respecting the schema at
     `@.claude/skills/check-setup/arch-schema.md`. Add / update sections per the schema. When
     adding a row to a table (e.g. capabilities.md → capability→context/container map),
     ensure the row references existing files (context file, container heading).
   - **For new `ddd/contexts/<name>.md` files** — create with every required section filled or
     N/A. Never leave `ARCH_GAP` on a new context file.
   - **For L1/L2 modularity-tree adds** — create the dir + `README.md` under
     `docs/architecture/current/modularity/<...>/<slug>/README.md` with front-matter
     (`id`, `level`, `parent`, `title`), the soft-convention header line
     (`Container: [...] · Context: [...]`), and the standard `## Purpose` / `## Scope (in / out)`
     sections.
   - **Multi-lens check.** After computing the file list, if it spans bizbok/ + ddd/ + c4/,
     emit the multi-lens warning and confirm with the user before writing.

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

- Never writes `docs/superpowers/specs/*-<name>-design.md` or
  `docs/superpowers/plans/*-<name>.md`.
- Never runs `superpowers:brainstorming` or `superpowers:writing-plans`.
- Never proposes a change level (C0/C1/C2). Change levels are a property of specs, not of
  arch changes. This PR is gated by `/arch-log-review` alone.
- Never auto-runs `/modularize` — step 8 only emits a verdict + suggestion.
