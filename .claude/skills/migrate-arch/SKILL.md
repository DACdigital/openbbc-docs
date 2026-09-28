---
name: migrate-arch
description: Non-deterministic LLM-driven migration of arbitrary docs/architecture/current/ content into the schema shape at @.claude/skills/check-setup/arch-schema.md. Preserves gaps as ARCH_GAP markers, quarantines unmatched source, opens a review PR. Trigger logic uses /check-setup presence output.
disable-model-invocation: true
---

# /migrate-arch

## Purpose

Brownfield migration. Accepts any starting state under `docs/architecture/current/` — freestyle,
partial, empty, radically different from the schema — and produces the 20-file schema shape. Uses
LLM interpretation to bucket source content into lens files by meaning, not by filename or heading
pattern. Never invents content — anything not sourced is left as an explicit `ARCH_GAP` marker.
Unmatched source content is quarantined, never deleted.

Output is one PR the human reviews before merge. The LLM's bucketing is non-deterministic; PR
review is the correctness gate.

## Inputs

- Optional `--clean` flag. Skips the `_migration-quarantine/` audit archive and `git rm`'s all
  source files that aren't one of the 20 schema targets after bucketing. See step 7.

Otherwise the skill reads existing `docs/architecture/current/` and writes to a new branch.

## Trigger logic

Before doing any work, **invoke `/check-setup` in-session** and read the four-check gate output
(check 4 — Architecture schema conformance):

- **All 20 required files present** (presence sub-check passes) → print `nothing to migrate — /check-setup presence passes` and exit.
- **Some required files missing** (presence sub-check fails) → migration needed, proceed.
- **All files present but non-placeholder sub-check fails** → print `not a migration case — fill ARCH_GAP markers manually or via /new-feature` and exit.

## Steps

1. **Announce and confirm.** Print a summary of the source state (file count, top-level dirs
   under `docs/architecture/current/`) and confirm with the user: "Migrate this into the schema
   shape? [Y/n]". Abort on `n`.

2. **Read the schema.** Load `@.claude/skills/check-setup/arch-schema.md`. Extract the list of
   20 required files, each with its required-section headings and non-placeholder rules.

3. **Scan the source.** Recursively list every file under `docs/architecture/current/`,
   **excluding only** `_migration-quarantine/**` (already-processed content from a prior
   run). Files under `docs/architecture/logs/` are outside this scope entirely. Read each
   remaining file — including any files under `modularity/` and `estimation/` — and build
   an in-memory source inventory keyed by path. **Rationale:** modularity/ and estimation/
   files often contain per-service purpose text that legitimately feeds `c4/containers.md`
   and `bizbok/capabilities.md` subsections. The LLM should read them for bucketing; the
   sweep step (§6) is what protects them from being moved.

4. **Create the migration branch.** `git checkout -b arch/migrate-schema` from the current
   branch. If the branch already exists, abort and instruct the user to either delete it and
   re-run from the current default branch, or run from the same branch again to trigger
   Re-run behavior (see below).

5. **Bucket via LLM interpretation — per target file, per required section.**

   For each of the 20 target files: for each required section in that file: ask the LLM
   (yourself, this session) — "given this source inventory, which chunks belong in this
   section?" — outputting either extracted content + source path, or `NO_MATCH`.

   Rules for bucketing:
   - **One target file per LLM call.** Do not batch across targets — a miscategorization in
     one target should not cascade.
   - **The prompt names the schema anchor** and the section purpose from the schema doc.
   - **The prompt shows the required-section headings** so the extracted content matches the
     target shape.
   - **Never invent — but DO translate between vocabularies.** Translation is extraction,
     not invention. The source usually describes the system in one vocabulary (services,
     capabilities, ACL rules); the schema demands another (DDD bounded contexts, C4
     containers, BIZBOK stakeholders, information concepts). Translating between them is
     expected and correct:
     - Source describes a service with a distinct responsibility + data ownership → extract
       as a DDD bounded context in `ddd/contexts/<name>.md`.
     - Source describes an actor role interacting with the system → extract to
       `bizbok/stakeholders.md`.
     - Source describes an external system with a protocol → extract to
       `c4/integrations.md`.
     - Source describes a service's tech stack + owned data + published API → extract to a
       `### <Container>` subsection in `c4/containers.md`.
     - Source describes ACL rules or auth policies → extract to `ddd/access-model.md`.

     Invention is asserting facts the source doesn't imply at any level: making up NFR
     targets (e.g. "p99 < 100ms"), fabricating regulatory regimes, hallucinating aggregates
     or policy rules. If the source doesn't imply it, output `NO_MATCH`.

     **Rule of thumb:** if a competent architect reading the source would identify the same
     fact from context, it's translation (extract). If they'd say "the source doesn't say
     that", it's invention (`NO_MATCH`).
   - **Use the diagram conventions.** When bucketing produces content for a section that
     requires a diagram (`README § System at a glance`, `c4/context § Diagram`,
     `c4/containers § Diagram`, `c4/data-flows § <flow>`, `c4/deployment § Diagram`,
     `ddd/context-map § Diagram`, `bizbok/value-streams § <stream>` optional), consult
     `@.claude/skills/check-setup/arch-schema.md#diagram-conventions` for the mermaid
     syntax + template. Populate the template with the entities extracted from source —
     never invent new diagram styles.

6. **Move source content out of the way (before writing targets).** Sweep every path in the
   source inventory from step 3 so `docs/architecture/current/` is empty of source content at
   the paths the target-writer needs. This step runs **before** step 7 so that a source file
   at a colliding path (e.g., `README.md` at the current/ root) is preserved before the
   target-write overwrites it:

   - **Default (no `--clean`):** move **every** source file — both extracted-from and
     unclaimed — to `docs/architecture/current/_migration-quarantine/`, preserving the
     original relative path. Extracted content will also live in target files (written in
     step 7, with provenance comments); the originals in quarantine are the audit trail.
   - **`--clean`:** skip quarantine entirely, `git rm` every source-inventory file. The
     migration commits the reshape without an audit archive. Use only when confident the
     LLM's bucketing (step 5) captured everything worth keeping.

   **Sweep-set is narrower than scan-set.** Both branches (default and `--clean`) protect
   these paths — they were **read** for bucketing in step 3 but are **never moved or removed**
   in step 6:
   - `modularity/**` — project-owned tree, written only by `/modularize`.
   - `estimation/**` — optional project content.
   - `_migration-quarantine/**` — was already excluded from scan; also never touched here
     (it's the destination, not a source).
   - `docs/architecture/logs/**` — outside `current/` entirely; append-only.

   Content that the LLM extracted from these preserved paths still lives in target files
   (with provenance comments like `<!-- migrated from modularity/<L1>/README.md ... -->`);
   the originals remain in place for their normal downstream consumers.

7. **Write the target files.** For each of the 20 required files:
   - Create the file at `docs/architecture/current/<path>` with the title heading + required
     section headings in order. (After step 6 the path is guaranteed empty.)
   - For each section with a match: write the extracted content, then append a provenance
     HTML comment: `<!-- migrated from <source-path> on YYYY-MM-DD -->`. If content came from
     multiple sources, list all of them.
   - For each section with `NO_MATCH`: write an `ARCH_GAP` marker with all four fields
     (`reason` / `Section` / `Fill with` / `See`) per the format at
     `@.claude/skills/check-setup/arch-schema.md#arch-gap-marker-format`:

           <!-- ARCH_GAP: <one-line reason>
                Section: <required-section-name>
                Fill with: <hint from the schema's non-placeholder rule for this file>
                See: .claude/skills/check-setup/arch-schema.md#<anchor> -->

8. **Write the paired arch-log entry** at
   `docs/architecture/logs/YYYY-MM-DD-arch-schema-migration/README.md`:

       # arch-schema-migration — reshape docs/architecture/current/ into schema

       **Date**: YYYY-MM-DD
       **Codename**: arch-schema-migration

       **Driver**: template pulled schema-enforcing /check-setup — migrate to conform

       **Decision**: reshape docs/architecture/current/ from freestyle into the 20-file schema
       defined in .claude/skills/check-setup/arch-schema.md

       **Rationale**: /check-setup now enforces the schema; existing freestyle content must be
       bucketed into schema shape (or quarantined for human review) before /check-setup can pass.

       **Alternatives rejected**:
       - Keep freestyle — rejected: /check-setup fails on missing required files.
       - Manual reshape — rejected: LLM bucketing preserves provenance and structure at scale.

       **Impact**:
       - reshaped: docs/architecture/current/** (20 files written per schema)
       - quarantined: docs/architecture/current/_migration-quarantine/** (unmatched source)
       - + docs/architecture/logs/YYYY-MM-DD-arch-schema-migration/README.md

       **Links**:
       - .claude/skills/check-setup/arch-schema.md
       - docs/superpowers/specs/2026-09-18-arch-current-schema-design.md

9. **Commit and push.** `git add docs/architecture/current/** docs/architecture/logs/YYYY-MM-DD-arch-schema-migration/`,
   commit, push the branch, open a PR against the current default branch.

10. **Append the migration report to the PR description** (not committed). Format:

        ### Migration report

        Moved:
          - <source path> → <target path> § <section>
          - ...

        Gaps (ARCH_GAPs to fill before /check-setup passes):
          - [ ] <target path> § <section>
          - ...

        Quarantined (review + fold or delete):
          - <_migration-quarantine path>
          - ...

11. **Print summary** to the user: the branch name, PR URL, count of moves / gaps / quarantined
    files. Remind the human to review bucketing, empty quarantine, fill gaps. If
    `docs/architecture/current/modularity/` exists and any `<L1>/README.md` in it lacks the
    schema's soft-convention header line (`Container: [<name>](../../c4/containers.md#<anchor>) · Context: [<name>](../../ddd/contexts/<name>.md)`),
    append a hint: `Modularity tree preserved. Run /modularize --refine on L1 nodes missing the
    soft-convention header to close /check-setup rule 11.`

## Re-run behavior

Idempotency-lite. On a second invocation:

- If `/check-setup` presence now passes → print `nothing to migrate` and exit.
- If presence still fails (rare — someone deleted a scaffolded file) → re-scaffold *only the
  missing files* per the greenfield procedure in
  `@.claude/skills/setup-project/SKILL.md` (Phase 2 step 2, `n (scaffold / greenfield)`
  branch) — including its single-table-file and lens-README special cases. Never overwrite
  existing populated content. Never
  re-run bucketing against already-populated targets.

## What this command never does

- Never invents content. `NO_MATCH` → `ARCH_GAP`, never a plausible fill.
- Never deletes source content **by default** — every source file is moved to
  `_migration-quarantine/` for audit. With the opt-in `--clean` flag, source files that
  aren't schema targets are `git rm`'d after bucketing; use only when confident.
- Never commits without opening a PR — the PR review is the correctness gate for LLM bucketing.
- Never touches `docs/architecture/logs/` beyond adding the paired migration entry.
- Never edits the schema at `@.claude/skills/check-setup/arch-schema.md` — schema changes are
  template-owned and land via `/sdd-rebase`, not per-project skills.

## Config

Reads `.claude/tracker.json` (comms only, optional notification). Writes to the migration branch;
does not touch the tracker or the default branch directly.
