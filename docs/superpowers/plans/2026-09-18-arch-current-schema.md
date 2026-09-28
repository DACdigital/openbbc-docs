# Schema for `docs/architecture/current/` — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ship a single non-versioned schema for `docs/architecture/current/` — 20 required files across three architectural lenses (BIZBOK, DDD, C4) + top-level cross-cutting files — enforced by `/check-setup`, scaffolded by `/setup-project`, and migrated into via a new `/migrate-arch` skill. Downstream skills (`/new-feature`, `/spec-review`, `/modularize`) become schema-aware.

**Architecture:** Docs-only, no code. Adds `.claude/skills/check-setup/arch-schema.md` as the single source of truth for the schema. All skills reference it via `@` imports. `/check-setup` runs a four-check gate (presence, non-placeholder, cross-references, estimation coverage). `/setup-project` scaffolds all 20 files with `ARCH_GAP` markers. `/migrate-arch` (new skill) uses LLM interpretation to bucket freestyle arch content into schema shape, preserves gaps explicitly, and quarantines unmatched source.

**Tech Stack:** Markdown, YAML front-matter, HTML-comment markers.

**Commit policy:** Per standing instruction, do not commit the spec or plan themselves. Implementation edits may be committed at the end of each task; user opts in per commit.

**Spec:** `docs/superpowers/specs/2026-09-18-arch-current-schema-design.md`

**No test suite:** This repo has no automated test runner. "Testing" means running affected skills in a session, or `grep`/inspection verification of file structure. Each task includes explicit verification steps.

---

## File Structure

**Create:**
- `.claude/skills/check-setup/arch-schema.md` — the single schema source of truth (17 file schemas + 3 lens README specs + cross-ref rules + `ARCH_GAP` format + `N/A because` escape hatch).
- `.claude/skills/migrate-arch/SKILL.md` — new brownfield migration skill.

**Modify:**
- `.claude/skills/check-setup/SKILL.md` — replace arch checks (4, 5, 9) with a single schema-driven check that runs the four-check gate.
- `.claude/skills/setup-project/SKILL.md` — rewrite Phase 2 arch-scaffolding branch to drop all 20 files with `ARCH_GAP` markers; add brownfield delegation to `/migrate-arch`.
- `.claude/skills/new-feature/SKILL.md` — add schema-awareness (route edits to the right lens files per feature-kind; warn on multi-lens PRs).
- `.claude/skills/modularize/SKILL.md` — add soft-convention header line writer.
- `.claude/agents/spec-review.md` — update Architecture sub-verdict checklist to lens vocabulary.
- `.claude/agents/architecture-log-review.md` — update Layer discipline check to three-lens language.
- `docs/process/AGENTS.md` — layer-discipline hard rule → three-lens; new rules (schema conformance mandatory; migration one-way).
- `README.md` — add a "Schema for `current/`" section under Repo layout.

---

## Task 1 — Create the schema file (skeleton + top-level + top-level cross-cutting)

**Files:**
- Create: `.claude/skills/check-setup/arch-schema.md`

- [ ] **Step 1: Create the schema file with the skeleton and top-level cross-cutting schemas**

Create `.claude/skills/check-setup/arch-schema.md` with this content. Source for section content: `docs/superpowers/specs/2026-09-18-arch-current-schema-design.md` §2 (Per-file schemas) — read that for the authoritative table content.

```markdown
# Arch Schema — `docs/architecture/current/`

**Purpose.** Single, non-versioned schema for `docs/architecture/current/`. Every skill that
reads or writes arch content references this file via `@.claude/skills/check-setup/arch-schema.md`.
When this schema changes, downstream repos pick it up via `/sdd-rebase`.

**Consumers:** `/check-setup` (enforces), `/setup-project` (scaffolds), `/migrate-arch`
(buckets into), `/new-feature` (routes edits to the right lens files).

**Overall shape.** 20 required files minimum — 17 content files + 3 lens READMEs —
organized as top-level cross-cutting + three lens dirs (`bizbok/`, `ddd/`, `c4/`), plus
optional `estimation/` and the existing `modularity/` tree.

## Directory layout

    docs/architecture/current/
    ├── README.md
    ├── nfrs.md
    ├── assumptions.md
    ├── constraints.md
    ├── glossary.md
    ├── bizbok/
    │   ├── README.md
    │   ├── capabilities.md
    │   ├── stakeholders.md
    │   ├── value-streams.md
    │   └── information-map.md
    ├── ddd/
    │   ├── README.md
    │   ├── context-map.md
    │   ├── contexts/
    │   │   └── <name>.md            (at least one)
    │   └── access-model.md
    ├── c4/
    │   ├── README.md
    │   ├── context.md
    │   ├── containers.md
    │   ├── data-flows.md
    │   ├── deployment.md
    │   └── integrations.md
    ├── estimation/                  (optional; coverage-gated if present)
    └── modularity/                  (existing; unchanged; soft-header convention)

## `ARCH_GAP` marker format {#arch-gap-marker-format}

Standardized HTML comment; invisible in rendered markdown; greppable.

    <!-- ARCH_GAP: <one-line reason>
         Section: <required-section-name>
         Fill with: <concrete guidance>
         See: .claude/skills/check-setup/arch-schema.md#<anchor> -->

**Escape hatch.** The literal phrase `N/A because <reason>` (with ≥5-word reason) in a
required section counts as filled — a positive assertion of inapplicability.

## Non-placeholder rule

For each required section listed below, a filled section contains either:
- Concrete content matching the section's shape, OR
- The literal `N/A because <reason>` phrase with ≥5-word reason.

An `ARCH_GAP` marker in a required section = unfilled = `/check-setup` fails.

---

## Top-level files

### `README.md` {#readme}

**Required sections (in order):**
- `## System at a glance` — one mermaid diagram
- `## Map of content` — links to every required file below
- `## Reading order` — suggested review order for new readers

**Non-placeholder rule:** diagram present with ≥3 labelled boxes; map-of-content lists every required file.

**Frontmatter (optional):** `title`, `updated`.

### `nfrs.md` {#nfrs}

**Required sections (in order):**
- `## Availability`
- `## Performance`
- `## Security`
- `## Compliance`
- `## Observability`

**Non-placeholder rule:** each section has ≥1 concrete target OR `N/A because <reason>`.

### `assumptions.md` {#assumptions}

**Required sections (in order):**
- `## Scope assumptions`
- `## Design decisions (locked)`
- `## Open questions`

**Non-placeholder rule:** ≥1 bullet per section; each locked decision has rationale + date.

### `constraints.md` {#constraints}

**Required sections (in order):**
- `## Regulatory regimes`
- `## Compliance obligations`
- `## Hard technical limits`

**Non-placeholder rule:** each regime cites applicability scope; limits are quantified.

### `glossary.md` {#glossary}

**Required content:** a single table with columns `Term | Definition | Owning context | Aliases`.

**Non-placeholder rule:** ≥5 terms; every concept in `bizbok/information-map.md` also appears here.

---
```

- [ ] **Step 2: Append the bizbok/ schemas to the same file**

Append this to `.claude/skills/check-setup/arch-schema.md`:

```markdown
## bizbok/ — business layer

### `bizbok/README.md` {#bizbok-readme}

**Required sections:** `## What BIZBOK is`; `## What lives here`; `## How it connects to DDD and C4`.

**Non-placeholder rule:** 2–3 sentence lens explainer; cross-links to `ddd/README.md` and `c4/README.md`.

### `bizbok/capabilities.md` {#bizbok-capabilities}

**Required sections (in order):**
- `## L1 capabilities`
- `## L2 capabilities`
- `## Capability → context/container map`

**Non-placeholder rule:** ≥3 L1 capabilities; every L1 has ≥1 L2; every L2 in the map names its DDD context (as a link to `ddd/contexts/<name>.md`) and its C4 container (as a link to `c4/containers.md#<anchor>`).

### `bizbok/stakeholders.md` {#bizbok-stakeholders}

**Required content:** a single `## Stakeholder catalog` section with a table `Name | Role | Goals | Scope`.

**Non-placeholder rule:** ≥1 stakeholder with goals + scope filled.

### `bizbok/value-streams.md` {#bizbok-value-streams}

**Required sections:** `## Streams`, containing one subsection per user journey with the shape `trigger → steps → outcome`.

**Non-placeholder rule:** ≥1 stream with ≥3 steps; every step names the capability from `bizbok/capabilities.md` it exercises (as an inline link).

### `bizbok/information-map.md` {#bizbok-information-map}

**Required content:** a single table `Concept | Description | Owning capability | Regulatory tag`.

**Non-placeholder rule:** ≥3 concepts; each concept also appears in `glossary.md`.

---
```

- [ ] **Step 3: Append the ddd/ schemas**

Append this to `.claude/skills/check-setup/arch-schema.md`:

```markdown
## ddd/ — domain layer

### `ddd/README.md` {#ddd-readme}

**Required sections:** `## What DDD is`; `## What lives here`; `## How it connects to BIZBOK and C4`.

**Non-placeholder rule:** 2–3 sentence lens explainer; cross-links to `bizbok/README.md` and `c4/README.md`.

### `ddd/context-map.md` {#ddd-context-map}

**Required sections (in order):**
- `## Contexts` — list of bounded contexts, each linking to its `contexts/<name>.md`
- `## Relationships` — per pair of contexts: upstream/downstream / ACL / conformist / partnership
- `## Diagram` — mermaid diagram

**Non-placeholder rule:** ≥1 context; every listed context has a matching `contexts/<name>.md`; every relationship in the pair table is labelled with its pattern.

### `ddd/contexts/<name>.md` {#ddd-context-file}

**Required sections (in order):**
- `## Purpose`
- `## Aggregates & entities`
- `## Domain events`
- `## Invariants`
- `## Published surface` — commands / queries / events other contexts consume

**Non-placeholder rule:** ≥1 aggregate; ≥1 invariant; published surface may state `N/A because <reason>` (per the escape hatch) if the context is truly internal.

### `ddd/access-model.md` {#ddd-access-model}

**Required sections (in order):**
- `## Model` — name the pattern (RBAC / ABAC / hybrid)
- `## Policies` — table `Actor | Capability | Info concept | Ops | Conditions`
- `## Tenant scoping`

**Non-placeholder rule:** ≥1 policy per stakeholder in `bizbok/stakeholders.md`.

---
```

- [ ] **Step 4: Append the c4/ schemas**

Append this to `.claude/skills/check-setup/arch-schema.md`:

```markdown
## c4/ — system/infra layer

### `c4/README.md` {#c4-readme}

**Required sections:** `## What C4 is`; `## What lives here`; `## How it connects to BIZBOK and DDD`.

**Non-placeholder rule:** 2–3 sentence lens explainer; cross-links to `bizbok/README.md` and `ddd/README.md`.

### `c4/context.md` {#c4-context}

**Required sections (in order):**
- `## Diagram` — mermaid C4-context diagram
- `## External actors` — link each to `bizbok/stakeholders.md`
- `## External systems` — link each to `c4/integrations.md`
- `## System boundary`

**Non-placeholder rule:** diagram + ≥1 external actor + ≥1 external system + system-boundary statement.

### `c4/containers.md` {#c4-containers}

**Required sections (in order):**
- `## Diagram` — mermaid C4-container diagram
- One `### <Container>` subsection per container, each containing: purpose, tech stack, data ownership, published API / events, link to its DDD context (`ddd/contexts/<name>.md`), link to its modularity node (`modularity/<slug>/README.md`) if the tree is populated

**Non-placeholder rule:** ≥1 container; each has a data-ownership statement (not `N/A`).

### `c4/data-flows.md` {#c4-data-flows}

**Required content:** one subsection per canonical flow (e.g. `## Write path`, `## Read path`, `## Event fan-out`), each with a sequence or flowchart diagram + narrative.

**Non-placeholder rule:** ≥1 canonical flow.

### `c4/deployment.md` {#c4-deployment}

**Required sections (in order):**
- `## Environments`
- `## Network zones`
- `## Trust boundaries`
- `## Threat model` — STRIDE-lite table `Boundary | Threat | Mitigation`

**Non-placeholder rule:** ≥1 environment; every container in `c4/containers.md` is placed in ≥1 zone; threat model has ≥1 threat per trust boundary.

### `c4/integrations.md` {#c4-integrations}

**Required sections (in order):**
- One table `System | Kind | Protocol | Auth | SLA/regulatory`
- `## Contracts` — versioned schemas per integration

**Non-placeholder rule:** ≥1 integration OR the explicit statement `N/A because this system has no external integrations of any kind`.

---
```

- [ ] **Step 5: Append the estimation / modularity / cross-reference / gate sections**

Append this to `.claude/skills/check-setup/arch-schema.md`:

```markdown
## Optional & existing

### `estimation/` {#estimation}

**Presence:** optional. When present, its internal schema is deferred to a follow-up spec.

**Coverage rule:** if the directory exists, every L2 capability in `bizbok/capabilities.md` must appear (by name) in ≥1 file inside `estimation/`.

### `modularity/<L1>/README.md` {#modularity-header}

**Soft-convention (warn-only):** every `modularity/<L1>/README.md` written by `/modularize` should have as its first content line:

    Container: [<name>](../../c4/containers.md#<anchor>) · Context: [<name>](../../ddd/contexts/<name>.md)

`/check-setup` warns on absence; does not fail.

---

## Cross-reference rules {#cross-reference-rules}

**Join philosophy.** Every fact has exactly one home. Other files reference it by link. `/check-setup` verifies links resolve; humans and `/spec-review` verify semantics.

### Enforced by `/check-setup` (fail on violation)

1. Every stakeholder in `bizbok/stakeholders.md` appears in ≥1 policy row in `ddd/access-model.md`.
2. Every L2 capability in `bizbok/capabilities.md` names a DDD context that resolves to `ddd/contexts/<name>.md`.
3. Every L2 capability in `bizbok/capabilities.md` names a container that resolves to a heading in `c4/containers.md`.
4. Every step in `bizbok/value-streams.md` names a capability that exists in `bizbok/capabilities.md`.
5. Every concept in `bizbok/information-map.md` also appears in `glossary.md`.
6. Every context listed in `ddd/context-map.md` has a matching `ddd/contexts/<name>.md`.
7. Every container in `c4/containers.md` links to a context that resolves to `ddd/contexts/<name>.md`.
8. Every container in `c4/containers.md` is placed in ≥1 zone in `c4/deployment.md`.
9. Every external system in `c4/context.md` appears in `c4/integrations.md`.
10. Every policy in `ddd/access-model.md` references a real stakeholder, capability, and information concept.

### Warned (not failed)

11. Every `modularity/<L1>/README.md` header names an existing container + context.

### Not enforced (human / `/spec-review` concern)

- Are the policies in `access-model.md` sensible for the capabilities?
- Do the domain events match the value streams?
- Does the C4 decomposition mirror the DDD context map?
- Are the STRIDE threats complete for the trust boundaries?

---

## `/check-setup` gate — four checks, no branching

1. **Presence** — all 20 required files exist (17 content + 3 lens READMEs), with ≥1 `ddd/contexts/<name>.md`. Missing = fail.
2. **Non-placeholder** — scan every required section for `ARCH_GAP` markers. Each = one failure with file:section + fill-hint.
3. **Cross-references** — the 10 enforced rules above. Dangling = fail.
4. **Estimation coverage** (only if `estimation/` present) — every L2 capability covered.

**Warn (not fail):** `_migration-quarantine/` non-empty; rule 11 (missing modularity soft-convention header).

### Report format

    docs/architecture/current/bizbok/capabilities.md § L2 capabilities:
      ARCH_GAP — required section unfilled
      Fill with: table of L2 capabilities with owning DDD context + C4 container
      See: .claude/skills/check-setup/arch-schema.md#bizbok-capabilities
```

- [ ] **Step 6: Verify anchor coverage**

Run:

```bash
grep -c '^### `' /home/john/dev/dac-docs-template/.claude/skills/check-setup/arch-schema.md
```

Expected: exactly `22` (5 top-level + 5 bizbok + 4 ddd + 6 c4 + estimation + modularity-header). If the count is off, re-check that the file was appended (not overwritten) at each step.

Then run:

```bash
grep -c '{#' /home/john/dev/dac-docs-template/.claude/skills/check-setup/arch-schema.md
```

Expected: `22` (one anchor per schema section). Fix any missing anchors before proceeding.

- [ ] **Step 7: Commit**

```bash
git add .claude/skills/check-setup/arch-schema.md
git commit -m "$(cat <<'EOF'
feat(arch-schema): add schema file for docs/architecture/current/

The single source of truth for the 20-file schema across three lenses
(bizbok/ddd/c4) plus top-level cross-cutting files. Referenced by
/check-setup, /setup-project, /migrate-arch, and /new-feature.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 2 — Update `/check-setup` to enforce the schema

**Files:**
- Modify: `.claude/skills/check-setup/SKILL.md` (replaces old arch checks 4, 5, 9 with a schema-driven check)

- [ ] **Step 1: Replace the "Checks" section**

Open `.claude/skills/check-setup/SKILL.md`. Locate the `## Checks` section (currently 10 numbered checks). Replace checks 4, 5, and 9 with a single new check "Architecture schema conformance". Renumber accordingly so the file has 8 checks total.

The new check block:

```markdown
4. **Architecture schema conformance** — read the schema at
   `@.claude/skills/check-setup/arch-schema.md` and enforce its four-check gate:
   - **Presence** — all 20 required files exist (17 content + 3 lens READMEs) under
     `docs/architecture/current/`, with ≥1 `ddd/contexts/<name>.md`. Missing = fail.
   - **Non-placeholder** — scan every required section (per the schema) for `ARCH_GAP` markers;
     each is a failure. The literal `N/A because <reason>` with ≥5-word reason counts as filled.
   - **Cross-references** — the 10 enforced rules in the schema (`## Cross-reference rules {#cross-reference-rules}` →
     "Enforced"). Dangling link = fail.
   - **Estimation coverage** — only if `estimation/` exists. Every L2 capability from
     `bizbok/capabilities.md` must appear in ≥1 file inside `estimation/`.

   Also **warn (do not fail)**:
   - `_migration-quarantine/` is non-empty (invoke `/migrate-arch` review or manually empty).
   - Any `modularity/<L1>/README.md` missing the soft-convention header line.

   Fix hint: for a fresh scaffold, `/setup-project` (Phase 2). For a brownfield input, run
   `/migrate-arch`. For unfilled `ARCH_GAP` markers, fill them manually or via `/new-feature`.

5. **Initial arch-log entry present** — at least one
   `docs/architecture/logs/YYYY-MM-DD-*/README.md` exists. Fix: `/setup-project` (Phase 2) creates
   an initial-scaffold log entry.
```

Delete the prior standalone check 9 ("Arch-vault NFRs + Assumptions present") — it is subsumed by the new check 4's presence subcheck.

Renumber the remaining checks (former 6, 7, 8, 10 become 6, 7, 8, 9 — since one check was deleted and three were folded into one).

- [ ] **Step 2: Update the "Output" section**

Locate the `## Output` section:

```markdown
## Output

A checklist, one line each `✅|⚠️|❌ <check> — <detail> [fix: <command>]`, then a one-line summary
`N/10 checks pass`.
```

Replace with:

```markdown
## Output

A checklist, one line each `✅ | ⚠️ | ❌ <check> — <detail> [fix: <command>]`, then a one-line
summary `N/9 checks pass`.

Where check 4 (architecture schema conformance) fails, the report expands with one line per
failed sub-check in the machine-parseable format from the schema doc:

    docs/architecture/current/<path> § <Section>:
      ARCH_GAP — required section unfilled
      Fill with: <hint>
      See: .claude/skills/check-setup/arch-schema.md#<anchor>

or (for cross-reference failures):

    docs/architecture/current/<path> § <Section>:
      Cross-ref violation — <rule N description>
      Fix with: <concrete guidance — usually "add the target file" or "correct the link">
      See: .claude/skills/check-setup/arch-schema.md#cross-reference-rules
```

- [ ] **Step 3: Verify check count**

Run:

```bash
grep -c '^[0-9]\{1,2\}\. \*\*' /home/john/dev/dac-docs-template/.claude/skills/check-setup/SKILL.md
```

Expected: `9` (the new total).

Run:

```bash
grep 'N/10 checks pass' /home/john/dev/dac-docs-template/.claude/skills/check-setup/SKILL.md
```

Expected: no matches (the `10` should have been replaced with `9`).

- [ ] **Step 4: Commit**

```bash
git add .claude/skills/check-setup/SKILL.md
git commit -m "$(cat <<'EOF'
feat(check-setup): enforce arch schema conformance

Replace the three legacy arch checks (README present, initial log entry,
NFRs+Assumptions present) with a single schema-driven check that runs
the four-check gate defined in arch-schema.md (presence, non-placeholder,
cross-references, estimation coverage). Total checks: 10 → 9.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 3 — Rewrite `/setup-project` Phase 2 to scaffold the schema

**Files:**
- Modify: `.claude/skills/setup-project/SKILL.md` (Phase 2)

- [ ] **Step 1: Replace Phase 2 arch-scaffolding branch**

Open `.claude/skills/setup-project/SKILL.md`. Locate `### Phase 2 — Project context` and, within it, step 2 (`**Import solution architecture (adopt-or-scaffold).**`).

Replace steps 2 and 4 (the current scaffold + NFRs/Assumptions seeding) with this single reorganized step 2:

```markdown
2. **Import solution architecture (adopt-or-scaffold).** Prompt the user: "Do you have an
   existing solution-architecture set to import?" [y/n].

   - **y (adopt / brownfield)** — accept a local dir path, a git URL to clone from, or an
     Obsidian vault export path. Copy contents into `docs/architecture/current/` verbatim,
     preserving the original layout. Then **delegate to `/migrate-arch`** to reshape into the
     schema (see the `/migrate-arch` skill for the flow — it opens a PR against the current
     branch). Phase 2 completes when `/migrate-arch`'s PR is either merged or the user opts to
     defer review; either way, `_migration-quarantine/` and unfilled `ARCH_GAP`s become the
     project's punch list.

   - **n (scaffold / greenfield)** — scaffold all 20 required files per the schema at
     `@.claude/skills/check-setup/arch-schema.md`. For each required file:
     1. Create the file with the title heading (system name for `README.md`; the file's role
        for others).
     2. Write every required section heading in the specified order.
     3. Under each required section, write one `ARCH_GAP` marker in the standard format:

            <!-- ARCH_GAP: required section unfilled.
                 Section: <section name>.
                 Fill with: <hint from schema>.
                 See: .claude/skills/check-setup/arch-schema.md#<anchor> -->

     4. Do not pre-fill any content beyond scaffolding — consistency with brownfield migration
        (both greenfield and brownfield land in "gap-filled fail state" the same way).

     **Single-table files** — `glossary.md`, `bizbok/stakeholders.md`, and
     `bizbok/information-map.md` declare `Required content` (a single table), not
     `Required sections`. For these, place one `ARCH_GAP` marker where the table would go
     and put the required column list in its `Fill with:` hint. The `Section:` field names
     the table's role (e.g. `Section: Stakeholder catalog table`).

     For `ddd/contexts/`, scaffold one placeholder file `ddd/contexts/example.md` following
     the same 4-step procedure using the required sections from
     `@.claude/skills/check-setup/arch-schema.md#ddd-context-file`. The human renames this
     file on first real context.

     Lens READMEs (`bizbok/README.md`, `ddd/README.md`, `c4/README.md`) get a 2–3 sentence
     explainer stub instead of `ARCH_GAP` — the schema tells the scaffolder what each lens
     is for, so this content is known at setup time. Include the cross-links to the other
     two lens READMEs.

     **Do NOT scaffold** anything under `modularity/` (populated later by `/modularize`)
     or `estimation/` (optional; not gated at setup).

   - **Log the initial state** — create
     `docs/architecture/logs/YYYY-MM-DD-initial-scaffold/README.md` with a real block:

           # initial-scaffold — adopt schema-driven arch scaffold

           **Date**: YYYY-MM-DD
           **Codename**: initial-scaffold

           **Driver**: project setup — /setup-project Phase 2

           **Decision**: adopt the schema-driven arch scaffold defined in
           .claude/skills/check-setup/arch-schema.md

           **Rationale**: every project starts from the same 20-file skeleton so downstream
           skills (/new-feature, /spec-review, /modularize) can rely on a stable shape.

           **Alternatives rejected**:
           - Freestyle (any layout) — rejected: makes tooling brittle and prevents cross-project
             navigation.

           **Impact**:
           - + docs/architecture/current/** (20 required files scaffolded)
           - + docs/architecture/logs/YYYY-MM-DD-initial-scaffold/README.md

           **Links**:
           - .claude/skills/check-setup/arch-schema.md
           - docs/superpowers/specs/2026-09-18-arch-current-schema-design.md
```

Then delete the current step 4 ("Seed the arch-vault NFRs + Assumptions files") — it is subsumed by the new step 2's scaffold. Renumber the remaining Phase 2 steps (former 3 → 3; former 5 → 4).

Also update the current step 3 (`**Ingest estimate — OPTIONAL.**`) — its `**y**` branch currently says "extract only NFRs and Assumptions from the estimate." Change the `**y**` branch to write the extracted content directly into `docs/architecture/current/nfrs.md` and `docs/architecture/current/assumptions.md`, replacing the `ARCH_GAP` markers left by step 2's scaffold. The extracted content should fill the schema's required sections (Availability / Performance / Security / Compliance / Observability for nfrs.md; Scope assumptions / Design decisions (locked) / Open questions for assumptions.md); any section the estimate doesn't cover keeps its `ARCH_GAP` marker.

- [ ] **Step 2: Update the Phase 2 closing paragraph**

The current final paragraph in Phase 2 reads:

```markdown
Phase 2 stops here. No modularization is run at setup. The DM/PL iterates on `current/` (with
paired arch-log entries) until the architecture is stable, then runs `/modularize` at root, then
`/plan-sprint`.
```

Replace with:

```markdown
Phase 2 stops here. Every scaffolded file has one `ARCH_GAP` marker per required section, so
`/check-setup` (Phase 6) will report the full punch list. This is expected — the report *is* the
list of gaps to fill. No modularization is run at setup. The DM/PL iterates on `current/` (with
paired arch-log entries per `/new-feature`) until the architecture is stable, then runs
`/modularize` at root, then `/plan-sprint`.
```

- [ ] **Step 3: Verify Phase 2 structure**

Run:

```bash
grep -A 2 '^### Phase 2' /home/john/dev/dac-docs-template/.claude/skills/setup-project/SKILL.md | head -20
```

Expected: Phase 2 header + first line of text visible.

Run:

```bash
grep -c '^[0-9]\+\. ' /home/john/dev/dac-docs-template/.claude/skills/setup-project/SKILL.md
```

Expected: the number of numbered steps across all phases should reflect the deletion (was ~30, should be ~28). Check the count is smaller by 2 (one deleted step + one embedded sub-step delta).

- [ ] **Step 4: Commit**

```bash
git add .claude/skills/setup-project/SKILL.md
git commit -m "$(cat <<'EOF'
feat(setup-project): scaffold arch schema in Phase 2

Rewrite Phase 2 step 2 to scaffold all 20 required files per the schema
(greenfield) or delegate to /migrate-arch (brownfield). Fold NFRs +
Assumptions seeding into the unified scaffold. Initial arch-log entry
now names schema adoption as its Decision.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 4 — Create `/migrate-arch` skill

**Files:**
- Create: `.claude/skills/migrate-arch/SKILL.md`

- [ ] **Step 1: Write the skill file**

Create `.claude/skills/migrate-arch/SKILL.md` with this content:

```markdown
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
   - **Never invent.** If the source doesn't say something, output `NO_MATCH`. Never summarize
     adjacent content into what "should" be there; never fill with "reasonable defaults";
     never infer.

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
```

- [ ] **Step 2: Verify skill structure**

Run:

```bash
head -5 /home/john/dev/dac-docs-template/.claude/skills/migrate-arch/SKILL.md
```

Expected: valid frontmatter with `name: migrate-arch`.

Run:

```bash
grep -c '^## ' /home/john/dev/dac-docs-template/.claude/skills/migrate-arch/SKILL.md
```

Expected: `7` (Purpose, Inputs, Trigger logic, Steps, Re-run behavior, What this command never does, Config).

- [ ] **Step 3: Commit**

```bash
git add .claude/skills/migrate-arch/SKILL.md
git commit -m "$(cat <<'EOF'
feat(migrate-arch): add brownfield arch migration skill

New skill that accepts arbitrary docs/architecture/current/ content and
buckets it into the schema shape via LLM interpretation. Preserves gaps
as ARCH_GAP markers, quarantines unmatched source. Opens a review PR.
Trigger logic uses /check-setup presence output.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 5 — Update `/new-feature` for schema awareness

**Files:**
- Modify: `.claude/skills/new-feature/SKILL.md`

- [ ] **Step 1: Add a "Schema awareness" section between Purpose and Inputs**

Open `.claude/skills/new-feature/SKILL.md`. Insert this new section immediately after the `## Purpose` section and before `## Inputs`:

```markdown
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
```

- [ ] **Step 2: Update the existing step 6 to reference the schema mapping**

Locate step 6 in the `## Steps` section:

```markdown
6. **Write the diff to `current/`**:
   - For **L1/L2 tree adds** — create the dir + `README.md` under
     `docs/architecture/current/modularity/<...>/<slug>/README.md` with front-matter
     (`id`, `level`, `parent`, `title`) and the standard `## Purpose` / `## Scope (in / out)`
     sections.
   - For **non-tree edits** (NFRs, assumptions, topology narrative) — patch the target files
     directly.
```

Replace with:

```markdown
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
```

- [ ] **Step 3: Verify**

Run:

```bash
grep -A 1 '## Schema awareness' /home/john/dev/dac-docs-template/.claude/skills/new-feature/SKILL.md | head -3
```

Expected: the section header + first line of body visible.

Run:

```bash
grep 'arch-schema.md' /home/john/dev/dac-docs-template/.claude/skills/new-feature/SKILL.md | wc -l
```

Expected: `≥2` (referenced in Schema awareness section + step 6).

- [ ] **Step 4: Commit**

```bash
git add .claude/skills/new-feature/SKILL.md
git commit -m "$(cat <<'EOF'
feat(new-feature): route edits to lens files per arch schema

Add a Schema awareness section with the feature-kind → files mapping.
Update step 6 to consult the mapping and warn on multi-lens PRs. Require
new files to be filled (not ARCH_GAP) at creation time.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 6 — Update `/modularize` to write the soft-convention header

**Files:**
- Modify: `.claude/skills/modularize/SKILL.md`

- [ ] **Step 1: Update the child-README template in step 8**

Open `.claude/skills/modularize/SKILL.md`. Locate step 8 in the Normal-mode section. The current template block reads:

```markdown
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
```

Replace with:

```markdown
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
```

Note the new second-body line (the soft-convention header). Below the template block, add this paragraph:

```markdown
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
```

- [ ] **Step 2: Add the same header line to the `--refine` add branch**

Locate the `--refine` mode step 4, the "**add**" bullet. Current text:

```markdown
   - **add** — mini Q&A for a new child (title, slug, purpose); create the dir + README as in normal
     mode step 8.
```

Replace with:

```markdown
   - **add** — mini Q&A for a new child (title, slug, purpose; also container link + context
     link when adding an **L1** child under root — may be `<TBD>` if not yet mapped; skip
     these two fields when adding L2 or L3 children); create the dir + README as in normal
     mode step 8, including the soft-convention header line for L1 children only.
```

- [ ] **Step 3: Verify**

Run:

```bash
grep -c 'Container:.*Context:' /home/john/dev/dac-docs-template/.claude/skills/modularize/SKILL.md
```

Expected: `≥1` (the header pattern appears in the template block).

- [ ] **Step 4: Commit**

```bash
git add .claude/skills/modularize/SKILL.md
git commit -m "$(cat <<'EOF'
feat(modularize): write soft-convention container/context header

Every L1/L2 modularity/README.md now leads with the schema's soft-convention
line linking to its C4 container + DDD context. Warn-only if missing per
/check-setup rule 11.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 7 — Update `spec-review` Architecture sub-verdict to lens vocabulary

**Files:**
- Modify: `.claude/agents/spec-review.md`

- [ ] **Step 1: Update the Architecture rubric bullets**

Open `.claude/agents/spec-review.md`. Locate the `### 3. Architecture (\`docs/conventions/architecture.md\`)` section. Replace its bullet list (from `- **Bounded contexts.**` through `- **Tenant isolation.**`) with:

```markdown
- **Capabilities & value streams** (`bizbok/capabilities.md`, `bizbok/value-streams.md`). Does
  the spec add or change a business capability, or introduce/rewire a user journey? If yes,
  do `bizbok/capabilities.md` and/or `bizbok/value-streams.md` reflect it? A spec that
  introduces a new capability or journey without a paired `/new-feature` update is a
  finding — the arch drift is real.
- **Bounded contexts** (`ddd/context-map.md`, `ddd/contexts/<name>.md`). Which context(s) does
  this feature touch or introduce? Does it respect existing boundaries per `ddd/context-map.md`?
  A new context requires a paired `/new-feature` first.
- **Information concepts** (`bizbok/information-map.md`, `glossary.md`). Does the spec introduce
  or rename a data concept? If yes, does `bizbok/information-map.md` + `glossary.md` name it?
- **Actors & access model** (`bizbok/stakeholders.md`, `ddd/access-model.md`). Does the spec
  introduce a new actor role, or change actor × capability × data policies? If yes, does
  `bizbok/stakeholders.md` list the role and does `ddd/access-model.md` show the new policy
  (per cross-ref rule 1: ≥1 policy per stakeholder)?
- **External integrations** (`c4/integrations.md`, `c4/context.md`). Does the spec introduce a
  new external system, third-party protocol, or SLA/regulatory dependency? If yes, do
  `c4/integrations.md` and `c4/context.md` reflect it?
- **Services / containers** (`c4/containers.md`). Is the change assigned to the right
  container(s)? Any new coupling that shouldn't exist, or a container taking on responsibility
  that belongs elsewhere? Does the container's declared data ownership in `c4/containers.md`
  still hold?
- **Deployment posture** (`c4/deployment.md`). Does the spec change trust boundaries or add a
  new zone-crossing? Does `c4/deployment.md` reflect it?
- **Dependency direction.** Any dependency violating the allowed direction per
  `docs/conventions/architecture.md` (shared/core depending on a feature-specific service;
  downstream reaching back upstream)?
- **Tenant isolation.** Is tenant scoping preserved for any new/touched data/behavior? Flag
  ambiguity as well as concrete violations.
```

- [ ] **Step 2: Update the escalation trigger**

Locate the `### 5. Escalation` section. Current text:

```markdown
Each sub-verdict can raise the change level (never lower it). If any of Naming / Architecture /
Contracts finds real impact the spec didn't declare, raise the overall level accordingly and state
so directly under the overall verdict.
```

Append this sentence at the end:

```markdown
In particular, if the spec's Architecture rubric finds capability / context / information-concept
/ access / container / deployment drift against `docs/architecture/current/` that would require
a `/new-feature` arch PR to reconcile, escalate to C2 and state so explicitly — the spec cannot
land before the arch is updated.
```

- [ ] **Step 3: Verify**

Run:

```bash
grep 'bizbok\|ddd\|c4' /home/john/dev/dac-docs-template/.claude/agents/spec-review.md | wc -l
```

Expected: `≥6` (lens dir names appear in the new bullets).

- [ ] **Step 4: Commit**

```bash
git add .claude/agents/spec-review.md
git commit -m "$(cat <<'EOF'
feat(spec-review): use lens vocabulary in Architecture sub-verdict

Update the Architecture rubric to name specific lens files (bizbok/,
ddd/, c4/) rather than generic 'bounded context / service boundary'
terms. Escalation now explicitly names the /new-feature arch-PR
requirement when drift is detected.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 8 — Update `architecture-log-review` to three-lens layer discipline

**Files:**
- Modify: `.claude/agents/architecture-log-review.md`

- [ ] **Step 1: Replace the Layer-discipline check body**

Open `.claude/agents/architecture-log-review.md`. Locate check 6 (`Layer discipline.`). Current text:

```markdown
6. **Layer discipline.** Is the change described by `Impact` at business / service / contract /
   deployment-topology level? Changes limited to any of: dependency versions, docker-compose or
   infra image bumps, CI configuration, lint rules, framework upgrades that don't change any
   contract, code-level refactors, small bug fixes — these are NOT architecture and belong in a
   spec or PR description instead. If the `Impact` list contains only such implementation-detail
   items, emit FAIL. If it mixes architecture and implementation-detail, emit PASS_WITH_ISSUES
   and instruct the author to split into an arch entry + a separate change.
```

Replace with:

```markdown
6. **Layer discipline.** Is the change described by `Impact` at one of the three lens levels?
   - **Business** (`bizbok/`) — capabilities, stakeholders, value streams, information concepts.
   - **Domain** (`ddd/`) — bounded contexts, aggregates, domain events, access model.
   - **System** (`c4/`) — context, containers, data flows, deployment, integrations.
   Cross-cutting concerns at the top level (`nfrs.md`, `assumptions.md`, `constraints.md`,
   `glossary.md`) also count as architecture.

   Changes limited to any of these are NOT architecture and belong in a spec, PR description,
   or code comment instead: dependency versions, docker-compose or infra image bumps, CI
   configuration, lint rules, framework upgrades that don't change any contract, code-level
   refactors, small bug fixes. Also non-architecture: delivery scheduling — sprint numbers,
   calendar dates, milestone references, deadlines. Product-phase language (MVP, M1, Beta,
   GA) is fine — it describes shape, not schedule.

   Rule of thumb: would the change still matter to someone reading the product plan a year
   from now? Then it belongs in `current/`. Otherwise a spec, PR description, or code comment.

   If the `Impact` list contains only architecture items, PASS. If it contains only
   implementation-detail or scheduling items, emit FAIL. If it mixes architecture and
   implementation-detail, emit PASS_WITH_ISSUES and instruct the author to split into an
   arch entry + a separate non-arch change.

7. **Multi-lens signal (signal-only, always emit).** If the PR touches all three lens dirs
   (`bizbok/`, `ddd/`, `c4/`) at once, emit PASS_WITH_ISSUES with a soft note: "This PR spans
   all three lenses — that is unusual and suggests the change may not be cleanly sliced.
   Consider splitting into per-lens PRs unless the change is genuinely cross-cutting (new
   bounded context that adds a capability, container, and access rule)." **Never emit FAIL
   from this check**; the multi-lens condition never fails a PR, only surfaces a nudge. Emit
   this signal even if an earlier gate (checks 1–6) already fired — the multi-lens observation
   is useful regardless of log-entry hygiene.
```

- [ ] **Step 2: Verify**

Run:

```bash
grep 'bizbok\|ddd\|c4' /home/john/dev/dac-docs-template/.claude/agents/architecture-log-review.md | wc -l
```

Expected: `≥4` (lens names in the updated check body).

Run:

```bash
grep -c '^[0-9]\+\. \*\*' /home/john/dev/dac-docs-template/.claude/agents/architecture-log-review.md
```

Expected: `7` (was 6; +1 for the new Multi-lens signal check).

- [ ] **Step 3: Commit**

```bash
git add .claude/agents/architecture-log-review.md
git commit -m "$(cat <<'EOF'
feat(arch-log-review): three-lens layer discipline + multi-lens signal

Replace the layer-discipline check body with three-lens language
(bizbok/ddd/c4 + top-level cross-cutting). Add a new soft check for
PRs touching all three lens dirs at once.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 9 — Update `docs/process/AGENTS.md` hard rules

**Files:**
- Modify: `docs/process/AGENTS.md`

- [ ] **Step 1: Replace the Architecture-layer-discipline hard rule**

Open `docs/process/AGENTS.md`. Locate the bullet starting with `- **Architecture layer discipline.**` (near the end of `## Hard rules`). Replace the entire bullet with:

```markdown
- **Architecture layer discipline.** `docs/architecture/current/` describes the product at three
  lens levels:
  - **Business** (`bizbok/`) — capabilities, stakeholders, value streams, information concepts.
  - **Domain** (`ddd/`) — bounded contexts, aggregates, domain events, access model.
  - **System** (`c4/`) — context, containers, data flows, deployment, integrations.

  Plus cross-cutting at the top level: `nfrs.md`, `assumptions.md`, `constraints.md`,
  `glossary.md`.

  It does NOT describe implementation choices — dependency versions, docker-compose or infra
  image bumps, CI configuration, lint rules, framework upgrades that don't change any contract,
  code-level refactors, small bug fixes. It also does NOT describe delivery scheduling — sprint
  numbers, calendar dates, quarters, tracker cycle or milestone references, deadlines, or
  "deferred to the next sprint." That information lives in `ROADMAP.md` and the tracker and
  decays too fast to be architecture-of-record.

  **Product-phase language is allowed:** MVP, M1, M2, Beta, GA, v1, "phase 1" describe the shape
  of the product at a delivery slice, not when that slice ships. Rule of thumb: if the change
  would still matter to someone reading the product plan a year from now, it belongs in
  `current/`; otherwise it belongs in a spec, a PR description, or a code comment.
  `architecture-log-review` enforces this.

  Reviewer soft signal: a single PR touching all three lens dirs (`bizbok/` + `ddd/` + `c4/`)
  at once is unusual and gets extra scrutiny — usually the change wasn't sliced cleanly.
```

- [ ] **Step 2: Add two new hard rules at the end of the `## Hard rules` bullet list**

Immediately after the "Architecture layer discipline" bullet you just replaced, append these two new bullets:

```markdown
- **Architecture files must conform to the schema.** `docs/architecture/current/` follows the
  schema at `.claude/skills/check-setup/arch-schema.md`: 20 required files, each with defined
  required sections and non-placeholder rules, enforced by `/check-setup`. Deviations require
  a schema change (edit `arch-schema.md`) under C2 discipline. Projects do not customize the
  schema per-project — template maintainers update it and `/sdd-rebase` propagates. New
  required files, sections, or cross-reference rules are breaking changes; removals are safe.
- **Reshape-from-freestyle is one-way via `/migrate-arch`.** Never manually reshape freestyle
  or pre-schema content in `docs/architecture/current/` into schema shape — run
  `/migrate-arch` and review its PR. The skill uses LLM interpretation, preserves gaps as
  `ARCH_GAP` markers, quarantines unmatched content in
  `docs/architecture/current/_migration-quarantine/`, and never invents content the source
  did not have. (Ordinary feature-driven edits to already-schema-shaped files via
  `/new-feature` are the normal path and are unaffected.)
```

- [ ] **Step 3: Also update the "Solution-architecture home is mandatory" rule**

Locate the bullet starting with `- **The solution-architecture home is mandatory.**`. Replace its body with:

```markdown
- **The solution-architecture home is mandatory.** From Phase 2 onward, all 20 required files
  under `docs/architecture/current/` per the schema must exist. Each required section is
  either filled with real content or filled with `N/A because <reason>` (≥5-word reason). An
  `ARCH_GAP` marker signals an unfilled section and fails `/check-setup`. At least one entry
  must exist in `docs/architecture/logs/`. `/check-setup` reports the absence of any required
  file, any unfilled `ARCH_GAP` marker, or any dangling cross-reference as a fail.
```

- [ ] **Step 4: Verify**

Run:

```bash
grep 'arch-schema.md' /home/john/dev/dac-docs-template/docs/process/AGENTS.md | wc -l
```

Expected: `≥2` (referenced in the two new bullets).

Run:

```bash
grep -c '^- \*\*' /home/john/dev/dac-docs-template/docs/process/AGENTS.md
```

Expected: `≥12` (was ~10; +2 new bullets).

- [ ] **Step 5: Commit**

```bash
git add docs/process/AGENTS.md
git commit -m "$(cat <<'EOF'
feat(hard-rules): three-lens layer discipline + schema conformance

Replace the Architecture layer discipline rule with three-lens language
(bizbok/ddd/c4). Add two new hard rules: architecture files must conform
to the schema at arch-schema.md; migration is one-way via /migrate-arch.
Tighten the mandatory-home rule to reference the 20-file schema.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 10 — Update `README.md` with a schema section

**Files:**
- Modify: `README.md`

- [ ] **Step 1: Add a new subsection under Repo layout**

Open `README.md`. Locate the `## Repo layout` section. Immediately after its closing triple-backtick block, insert this new subsection:

```markdown
### Schema for `docs/architecture/current/`

Every project's `docs/architecture/current/` follows a fixed 20-file schema across three
architectural lenses plus cross-cutting top-level files. The schema lives at
`.claude/skills/check-setup/arch-schema.md` and is enforced by `/check-setup`, scaffolded by
`/setup-project`, and migrated into via `/migrate-arch`.

Layout at a glance:

    docs/architecture/current/
    ├── README.md, nfrs.md, assumptions.md, constraints.md, glossary.md    ← top-level
    ├── bizbok/  — capabilities, stakeholders, value-streams, information-map    ← business layer
    ├── ddd/     — context-map, contexts/<name>.md, access-model                  ← domain layer
    ├── c4/      — context, containers, data-flows, deployment, integrations    ← system layer
    ├── estimation/  (optional; coverage-gated if present)
    └── modularity/  (existing; planning-time decomposition tree)

Missing files, unfilled `ARCH_GAP` markers, or dangling cross-references fail `/check-setup`.
See `.claude/skills/check-setup/arch-schema.md` for the full rules.
```

- [ ] **Step 2: Verify**

Run:

```bash
grep 'Schema for' /home/john/dev/dac-docs-template/README.md
```

Expected: `### Schema for \`docs/architecture/current/\``.

- [ ] **Step 3: Commit**

```bash
git add README.md
git commit -m "$(cat <<'EOF'
docs(readme): describe the arch schema under Repo layout

Add a Schema-for-current subsection with the 20-file layout at a glance
and pointers to the enforcing skill files.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 11 — End-to-end verification (dry-run)

**Files:**
- No file changes. This is a verification task using a temp directory.

- [ ] **Step 1: Create a sandbox project directory**

```bash
mkdir -p /tmp/arch-schema-sandbox && cd /tmp/arch-schema-sandbox
```

- [ ] **Step 2: Simulate greenfield scaffold — copy the empty structure the skill would produce**

Using the skill logic (do not actually run `/setup-project`; instead, manually create the 20 required files as it would), scaffold the schema:

```bash
mkdir -p docs/architecture/current/{bizbok,ddd/contexts,c4} docs/architecture/logs/2026-09-18-initial-scaffold

# Top-level files
for f in README.md nfrs.md assumptions.md constraints.md glossary.md; do
  touch docs/architecture/current/$f
done

# Lens dirs
for d in bizbok ddd c4; do
  touch docs/architecture/current/$d/README.md
done

# bizbok content
for f in capabilities.md stakeholders.md value-streams.md information-map.md; do
  touch docs/architecture/current/bizbok/$f
done

# ddd content
touch docs/architecture/current/ddd/context-map.md docs/architecture/current/ddd/access-model.md
touch docs/architecture/current/ddd/contexts/example.md

# c4 content
for f in context.md containers.md data-flows.md deployment.md integrations.md; do
  touch docs/architecture/current/c4/$f
done

# Log entry
touch docs/architecture/logs/2026-09-18-initial-scaffold/README.md

find docs/architecture/current -type f | wc -l
```

Expected: `20` (all 20 required files present).

- [ ] **Step 3: Verify the count matches the schema's stated total**

Cross-check:

```bash
find /tmp/arch-schema-sandbox/docs/architecture/current -type f -name '*.md' | sort
```

Expected — 20 files, matching the schema's list. Manually confirm the list is:

```
docs/architecture/current/README.md
docs/architecture/current/assumptions.md
docs/architecture/current/bizbok/README.md
docs/architecture/current/bizbok/capabilities.md
docs/architecture/current/bizbok/information-map.md
docs/architecture/current/bizbok/stakeholders.md
docs/architecture/current/bizbok/value-streams.md
docs/architecture/current/c4/README.md
docs/architecture/current/c4/containers.md
docs/architecture/current/c4/context.md
docs/architecture/current/c4/data-flows.md
docs/architecture/current/c4/deployment.md
docs/architecture/current/c4/integrations.md
docs/architecture/current/constraints.md
docs/architecture/current/ddd/README.md
docs/architecture/current/ddd/access-model.md
docs/architecture/current/ddd/context-map.md
docs/architecture/current/ddd/contexts/example.md
docs/architecture/current/glossary.md
docs/architecture/current/nfrs.md
```

If the list matches, the 20-file count in the schema is self-consistent.

- [ ] **Step 4: Simulate a freestyle input for `/migrate-arch` dry-run**

```bash
rm -rf /tmp/arch-schema-sandbox
mkdir -p /tmp/arch-schema-sandbox/docs/architecture/current/{00-Overview,01-Architecture,02-Infrastructure,10-Infosec}
cat > /tmp/arch-schema-sandbox/docs/architecture/current/README.md << 'EOF'
# System X — Design Vault
- Some old topology narrative
- Actors: End User, Ops
- Compliance: SOC2
EOF
cat > /tmp/arch-schema-sandbox/docs/architecture/current/00-Overview/Scope.md << 'EOF'
# Scope
This vault covers the AI layer of the copilot.
- Chat interface
- Knowledge base ingestion
- Retrieval

Stakeholders: End User, Ops Admin.
EOF
cat > /tmp/arch-schema-sandbox/docs/architecture/current/01-Architecture/System\ Overview.md << 'EOF'
# System Overview
Two services, one KB. Data flows from SharePoint to Qdrant nightly.
EOF
ls /tmp/arch-schema-sandbox/docs/architecture/current -R
```

Expected: a freestyle input with 3 files across 4 dirs. `/migrate-arch` would:
1. Detect presence check fails (missing all 20 required files).
2. Read the source inventory.
3. Bucket: `00-Overview/Scope.md` → likely `bizbok/capabilities.md § L1 capabilities`, `bizbok/value-streams.md § Streams`, `bizbok/stakeholders.md § Stakeholder catalog`.
4. `01-Architecture/System Overview.md` → likely `c4/context.md § Diagram` + `c4/containers.md § Diagram`.
5. Quarantine `01-Architecture/System Overview.md` contents that don't fit + the whole `02-Infrastructure/` and `10-Infosec/` dirs (empty in this sample, but they'd be moved verbatim if they had content).
6. Leave `ARCH_GAP` markers for everything the source didn't cover (e.g. `ddd/context-map.md § Relationships`, `c4/deployment.md § Threat model`).

Confirm the trigger logic (presence check → migrate) and the bucketing plan by walking through the skill's steps against this input. Do not actually run `/migrate-arch` in this sandbox — this is a paper walkthrough.

- [ ] **Step 5: Clean up the sandbox**

```bash
rm -rf /tmp/arch-schema-sandbox
```

- [ ] **Step 6: No commit for verification task**

Verification-only. Nothing to commit.

---

## Final self-review checklist

Before opening the PR from this branch:

- [ ] All 10 tasks committed.
- [ ] `git log --oneline | head -12` shows the expected 10 feat/docs commits + this task's plan (uncommitted).
- [ ] `grep -R 'arch-schema.md' .claude/ docs/process/ README.md | wc -l` returns `≥8` (schema referenced across 6 modified files and the schema file itself).
- [ ] Spec at `docs/superpowers/specs/2026-09-18-arch-current-schema-design.md` is unchanged (this plan modifies neither the spec nor the plan file itself).
- [ ] `docs/architecture/current/` in this repo remains empty — the template intentionally ships without a filled arch dir; downstream projects run `/setup-project` to scaffold it. If it accidentally got scaffolded during dev, revert.

## Notes for the executor

- **This plan modifies 8 files and creates 2 files.** Total commits: 10 (one per task, except the verification task).
- **No test framework exists in this repo.** Verification is grep-based structural checks.
- **The schema file (Task 1) is the foundation.** All other tasks reference its anchors. If Task 1 changes shape mid-plan, later tasks may need re-alignment — do not skip Task 1's step-6 anchor verification.
- **Multi-lens PR warning wording** appears in three places (new-feature, arch-log-review, hard rules). Keep the wording consistent if you tune it.
- **`ARCH_GAP` marker format** appears in the schema (Task 1), setup-project (Task 3), migrate-arch (Task 4). Same format everywhere. If you diverge, `/check-setup` cross-ref check may miss failures.
