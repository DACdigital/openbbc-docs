# Schema for `docs/architecture/current/`

**Date:** 2026-09-18
**Status:** Draft — pending user review

## Business value / Why

`docs/architecture/current/` is the load-bearing artifact of every project in this template: it's what `/setup-project` scaffolds, what `/new-feature` evolves, what `/spec-review` cross-checks, what `/modularize` decomposes, and what the arch-log paired-entry rule guards. Today its shape is under-specified — the hard rule requires only `README.md`, `nfrs.md`, `assumptions.md`, and one `logs/` entry. Everything else is discretion.

The consequences show up in practice. The one real downstream project (`ai-sales-copilot-docs`) invented a freestyle layout (`00-Overview / 01-Architecture / 02-Infrastructure / ...`) — sensible, but idiosyncratic. New projects will each invent their own. Skills that need to read arch content (`/spec-review`'s Architecture sub-verdict, `/new-feature`, `/modularize`'s cross-references) have no schema to lean on, so they treat the arch dir as free text.

**This spec introduces one schema — three architectural lenses (BIZBOK, DDD, C4) + cross-cutting top-level files + the existing modularity tree — enforced by `/check-setup`, scaffolded by `/setup-project`, and migrated into via a new `/migrate-arch` skill.** Twenty required files minimum; each with defined content sections and explicit gap markers where content is missing. The schema names one home per fact, defines required content per file, and declares checkable cross-references between files. Downstream skills (`/new-feature`, `/spec-review`, `/modularize`) become schema-aware.

Pedagogical bonus: naming the top-level dirs after the three canonical solution-architecture frameworks (BIZBOK, DDD, C4) exposes teams to the frameworks by encounter rather than by teaching.

## Change level

**C2.** The spec changes:

- The architecture-of-record contract (schema for every downstream project's `current/`).
- Multiple skills (`/check-setup`, `/setup-project`, `/new-feature`, `/spec-review`, `/modularize`, and a new `/migrate-arch`).
- Two hard rules in `docs/process/AGENTS.md` (layer discipline + one-way migration).

Contract-shape change affecting every downstream project's docs-repo state → C2 by definition.

## Scope

**In**

- The full on-disk schema for `docs/architecture/current/` — 20 required files (17 content + 3 lens READMEs), organized as top-level cross-cutting + three lens dirs (`bizbok/`, `ddd/`, `c4/`) + optional `estimation/` + existing `modularity/`.
- The per-file schema (required sections, non-placeholder rules) captured in a new `.claude/skills/check-setup/arch-schema.md`.
- `ARCH_GAP` marker convention + `N/A because <reason>` escape hatch.
- `/check-setup`'s four-check gate: presence, non-placeholder, cross-references, estimation coverage.
- Enforceable cross-reference rules between lens files.
- `/setup-project` Phase 2 scaffolding behavior (drops 18 files with `ARCH_GAP` markers + initial arch-log entry).
- New `/migrate-arch` skill: LLM-driven bucketing of freestyle arch content into schema shape, with gaps preserved and unmatched content quarantined.
- Updates to `/new-feature` (schema-aware evolution), `/spec-review` (Architecture sub-verdict uses lens vocabulary), `/modularize` (soft cross-ref header on modularity nodes).
- Layer-discipline hard rule update in `docs/process/AGENTS.md` (three-lens language).
- New hard rules: schema conformance is mandatory; migration is one-way via the skill.

**Out (YAGNI)**

- Schema versioning. Single schema; adapt or fail. Downstream repos either migrate or `/check-setup` fails from the moment they pull.
- Auto-migration on rebase. `/migrate-arch` runs on demand; `/sdd-rebase` does not silently reshape arch content.
- Machine-readable arch manifest (YAML/JSON alongside markdown). The markdown is the manifest; `/check-setup` parses it.
- Automated diagram generation from lens files.
- Cross-link broken-link validation beyond the cross-reference rules in this spec.
- `estimation/` internal schema — deferred to a follow-up spec. This spec only defines the coverage gate (if the dir exists, every L2 capability must appear).
- Editing lens-file schema per project. Projects don't override the schema; template maintainers do (and `/sdd-rebase` propagates).

## Writing principle

Concise, technical, complete. The process docs are the source of truth; `.claude/` skills and agents are the execution adapter. Where this spec disagrees with skill implementations, this spec wins.

## Design

### 1. Directory structure

```
docs/architecture/current/
├── README.md                    ← map of content + system-at-a-glance diagram
├── nfrs.md                      ← non-functional requirements
├── assumptions.md               ← explicit assumptions, locked design decisions
├── constraints.md               ← regulatory / compliance / hard limits (new)
├── glossary.md                  ← ubiquitous language + business vocab, one home (new)
│
├── bizbok/                      ← business layer
│   ├── README.md                ← explains the BIZBOK lens
│   ├── capabilities.md          ← capability map ("features" in business terms)
│   ├── stakeholders.md          ← actor types & business roles
│   ├── value-streams.md         ← user journeys/workflows → capabilities mapping
│   └── information-map.md       ← business-level data concepts
│
├── ddd/                         ← domain layer
│   ├── README.md                ← explains the DDD lens
│   ├── context-map.md           ← bounded contexts + relationships
│   ├── contexts/
│   │   └── <name>.md            ← one per bounded context. ≥1 required.
│   └── access-model.md          ← actor × capability × data policies
│
├── c4/                          ← system/infra layer
│   ├── README.md                ← explains the C4 lens
│   ├── context.md               ← C4 L1: system in the world
│   ├── containers.md            ← C4 L2: services + data stores
│   ├── data-flows.md            ← canonical end-to-end flows
│   ├── deployment.md            ← environments, zones, trust boundaries, threat model
│   └── integrations.md          ← external systems + protocols/contracts
│
├── estimation/                  ← OPTIONAL. Coverage-gated if present.
│
└── modularity/                  ← EXISTING. Unchanged tree.
    └── <L1>/[<L2>/]README.md      Header names its container + context.
```

**Gate count:** 20 required files minimum — 17 content files (5 top-level + 4 bizbok + 2 ddd (context-map, access-model) + ≥1 `ddd/contexts/<name>.md` + 5 c4) plus 3 lens READMEs. Plus modularity tree per existing rules.

**Lens READMEs** are load-bearing — each 100–300 words, explaining what its lens is for and pointing at the other two lens READMEs. A reader entering any door reaches the rest.

### 2. Per-file schemas

Each required file has (a) required sections, (b) a non-placeholder rule, (c) declared cross-references (§4).

`ARCH_GAP` markers in a required section count as unfilled and fail the gate. `N/A because <reason>` (≥5-word reason) counts as filled — a positive assertion of inapplicability.

**Top-level**

| File | Required sections | Non-placeholder rule |
|---|---|---|
| `README.md` | `## System at a glance` (mermaid); `## Map of content`; `## Reading order` | Diagram with ≥3 labelled boxes; map lists every required file |
| `nfrs.md` | `## Availability`; `## Performance`; `## Security`; `## Compliance`; `## Observability` | Each section: ≥1 concrete target or explicit N/A |
| `assumptions.md` | `## Scope assumptions`; `## Design decisions (locked)`; `## Open questions` | ≥1 bullet per section; decisions have rationale + date |
| `constraints.md` | `## Regulatory regimes`; `## Compliance obligations`; `## Hard technical limits` | Each regime cites applicability scope; limits quantified |
| `glossary.md` | Table: `Term \| Definition \| Owning context \| Aliases` | ≥5 terms; every information-map concept appears |

**bizbok/**

| File | Required sections | Non-placeholder rule |
|---|---|---|
| `README.md` | `## What BIZBOK is`; `## What lives here`; `## How it connects to DDD and C4` | 2–3 sentence explainer; cross-links to other lens READMEs |
| `capabilities.md` | `## L1 capabilities`; `## L2 capabilities`; `## Capability → context/container map` | ≥3 L1; every L1 has ≥1 L2; every L2 names its DDD context + C4 container |
| `stakeholders.md` | `## Stakeholder catalog` (table: Name \| Role \| Goals \| Scope) | ≥1 stakeholder with goals + scope |
| `value-streams.md` | `## Streams` (subsection per journey: trigger → steps → outcome) | ≥1 stream with ≥3 steps; every step names its capability |
| `information-map.md` | Table: `Concept \| Description \| Owning capability \| Regulatory tag` | ≥3 concepts; each in glossary.md |

**ddd/**

| File | Required sections | Non-placeholder rule |
|---|---|---|
| `README.md` | Same shape as bizbok/README.md | Cross-links to other lens READMEs |
| `context-map.md` | `## Contexts` (list); `## Relationships` (per pair); `## Diagram` | ≥1 context; every listed context has a matching `contexts/<name>.md`; every relationship labelled |
| `contexts/<name>.md` | `## Purpose`; `## Aggregates & entities`; `## Domain events`; `## Invariants`; `## Published surface` | ≥1 aggregate; ≥1 invariant; published surface may be "none" with reason |
| `access-model.md` | `## Model` (name the pattern); `## Policies` (table); `## Tenant scoping` | ≥1 policy per stakeholder |

**c4/**

| File | Required sections | Non-placeholder rule |
|---|---|---|
| `README.md` | Same shape as bizbok/README.md | Cross-links to other lens READMEs |
| `context.md` | `## Diagram` (mermaid C4-context); `## External actors`; `## External systems`; `## System boundary` | Diagram + ≥1 external actor + ≥1 external system + boundary statement |
| `containers.md` | `## Diagram` (mermaid C4-container); `### <Container>` subsection per container | ≥1 container; each with data-ownership statement + links to DDD context + modularity node |
| `data-flows.md` | Subsection per canonical flow with diagram + narrative | ≥1 flow |
| `deployment.md` | `## Environments`; `## Network zones`; `## Trust boundaries`; `## Threat model` (STRIDE-lite) | ≥1 environment; every container zoned; ≥1 threat per trust boundary |
| `integrations.md` | Table: `System \| Kind \| Protocol \| Auth \| SLA/regulatory`; `## Contracts` | ≥1 integration or explicit "no external integrations" |

**Optional & existing**

- `estimation/` — internal schema deferred. Gate: if the dir exists, every `bizbok/capabilities.md` L2 entry must appear in it.
- `modularity/<L1>/README.md` — unchanged tree. Soft-convention first line: `Container: [<name>](../../c4/containers.md#<anchor>) · Context: [<name>](../../ddd/contexts/<name>.md)`. Warn on absence; do not fail.

### 3. Enforcement — no versioning

Single schema, single truth. `/check-setup` enforces it directly; downstream repos either conform or fail. No coexistence path.

**Schema location:** `.claude/skills/check-setup/arch-schema.md`. Skills that read it (`/setup-project`, `/migrate-arch`, `/new-feature`) reference via `@.claude/skills/check-setup/arch-schema.md`. Single source; edited in one place. Not in `docs/architecture/` because that directory is *contents*, not *meta*.

**`/check-setup` gate — four checks, no branching:**

1. **Presence** — all 20 required files exist (17 content + 3 lens READMEs), with ≥1 `ddd/contexts/<name>.md`. Missing = fail.
2. **Non-placeholder** — scan every required section for `ARCH_GAP` markers. Each = one failure.
3. **Cross-references** — declared links resolve (§4). Dangling link = failure.
4. **Estimation coverage** (only if `estimation/` present) — every L2 capability covered.

**Warn (not fail):** `_migration-quarantine/` non-empty; missing modularity soft-convention header line.

**`ARCH_GAP` marker format:**

```html
<!-- ARCH_GAP: <one-line reason>
     Section: <required-section-name>
     Fill with: <concrete guidance>
     See: .claude/skills/check-setup/arch-schema.md#<anchor> -->
```

**Escape hatch:** literal `N/A because <reason>` (≥5-word reason) in a required section counts as filled.

**`arch-log-review` layer-discipline update.** The hard rule in `docs/process/AGENTS.md` currently describes arch content as "business capabilities, services and responsibilities, contracts, and deployment topology." Update wording to the three-lens language:

> `docs/architecture/current/` describes the product at three lens levels:
> - **Business** (`bizbok/`) — capabilities, stakeholders, value streams, information concepts
> - **Domain** (`ddd/`) — bounded contexts, aggregates, domain events, access model
> - **System** (`c4/`) — context, containers, data flows, deployment, integrations
>
> Plus cross-cutting concerns at the top level (nfrs, assumptions, constraints, glossary).
>
> It does NOT describe implementation choices (dep versions, docker-compose bumps, CI/lint, framework upgrades that don't change contracts, code refactors, small bugfixes) or delivery scheduling (sprint numbers, calendar dates, milestone refs, deadlines). Product-phase language (MVP, M1, Beta, GA) is allowed — it describes shape, not schedule.

Rule of thumb (unchanged): would it still matter a year from now? Then `current/`. Otherwise a spec, PR description, or code comment.

Reviewer soft signal: **a PR touching all three lens dirs at once is unusual and gets extra scrutiny** (usually means the change wasn't sliced cleanly).

**Report format** — `/check-setup` emits per failure:

```
docs/architecture/current/bizbok/capabilities.md § L2 capabilities:
  ARCH_GAP — required section unfilled
  Fill with: table of L2 capabilities with owning DDD context + C4 container
  See: .claude/skills/check-setup/arch-schema.md#bizbok-capabilities
```

### 4. Cross-reference rules

**Join philosophy:** every fact has exactly one home. Other files reference by link. `/check-setup` verifies links resolve; humans and `/spec-review` verify semantics.

**The mesh (directed):**

```
       glossary.md ←──── information-map.md
            ↑                    ↑
     capabilities.md ──→ contexts/<name>.md ←── containers.md
            ↑                    ↑                   ↓
     stakeholders.md      access-model.md     deployment.md
     value-streams.md ──→ capabilities.md     integrations.md
                                                     ↑
                                              context.md ─┘
                                                     ↑
                                          modularity/<L1>/
                                          → containers.md
                                          → contexts/<name>.md
```

**Enforceable rules (checked by `/check-setup`):**

| # | Rule | Failure mode |
|---|---|---|
| 1 | Every stakeholder in `bizbok/stakeholders.md` appears in ≥1 policy row in `ddd/access-model.md` | Stakeholder never policied |
| 2 | Every L2 capability in `bizbok/capabilities.md` names a DDD context resolving to `ddd/contexts/<name>.md` | Unresolved context link |
| 3 | Every L2 capability in `bizbok/capabilities.md` names a container resolving to a heading in `c4/containers.md` | Unresolved container link |
| 4 | Every step in `bizbok/value-streams.md` names a capability that exists in `bizbok/capabilities.md` | Unresolved capability link |
| 5 | Every concept in `bizbok/information-map.md` also appears in `glossary.md` | Concept not in glossary |
| 6 | Every context in `ddd/context-map.md` has a matching `ddd/contexts/<name>.md` | Missing context file |
| 7 | Every container in `c4/containers.md` links to a context resolving to `ddd/contexts/<name>.md` | Unresolved context back-link |
| 8 | Every container in `c4/containers.md` is placed in ≥1 zone in `c4/deployment.md` | Container unzoned |
| 9 | Every external system in `c4/context.md` appears in `c4/integrations.md` | External system missing integration entry |
| 10 | Every policy in `ddd/access-model.md` references a real stakeholder, capability, and information concept | Dangling policy reference |
| 11 | Every `modularity/<L1>/README.md` header names an existing container + context (soft — warn, don't fail) | Soft-convention drift |

**Not enforced (semantic; humans and `/spec-review`):**

- Are the policies in `access-model.md` sensible for the capabilities?
- Do the domain events match the value streams?
- Does the C4 decomposition mirror the DDD context map?
- Are the STRIDE threats complete for the trust boundaries?

### 5. `/setup-project` scaffolding (greenfield)

Phase 2 (Context) drops the full skeleton.

**Every required file** is created with the title heading, all required section headings in order, and one `ARCH_GAP` marker per required section pointing at the schema anchor with a fill-hint.

Example scaffold for `bizbok/capabilities.md`:

```markdown
# Capabilities

## L1 capabilities

<!-- ARCH_GAP: required section unfilled.
     Section: L1 capabilities.
     Fill with: top-level business capability areas (3+ typical).
     See: .claude/skills/check-setup/arch-schema.md#bizbok-capabilities -->

## L2 capabilities

<!-- ARCH_GAP: required section unfilled.
     Section: L2 capabilities.
     Fill with: nested capabilities under each L1 (1+ per L1).
     See: .claude/skills/check-setup/arch-schema.md#bizbok-capabilities -->

## Capability → context/container map

<!-- ARCH_GAP: required section unfilled.
     Section: Capability → context/container map.
     Fill with: table linking each L2 capability to its DDD context + C4 container.
     See: .claude/skills/check-setup/arch-schema.md#bizbok-capabilities -->
```

**Initial arch-log entry:** `docs/architecture/logs/YYYY-MM-DD-initial-scaffold/README.md` is created in the same commit with a real Driver / Decision / Rationale / Alternatives-rejected / Impact / Links block naming "adopt the schema-driven arch scaffold" as the decision. Satisfies the ≥1 log entry gate immediately and models the log format.

**No pre-filled content beyond scaffolding.** Even though `/setup-project` knows the system name and some Phase 0 answers, the scaffold uses `ARCH_GAP` uniformly. Reasons: consistency with brownfield migration; prevents humans from missing sections that look "already done"; keeps the `/check-setup` failure report as the human's punch list.

**Post-scaffold state:** `/check-setup` **fails immediately**. This is expected — the failure report *is* the punch list. Phase 6 (Check) reports the state as "gap-filled, expected — fill before feature work" rather than "setup broken."

**Adopt path:** when Phase 2 detects existing arch content (brownfield branch), it delegates to `/migrate-arch` instead of scaffolding fresh.

**Estimation & modularity:** `estimation/` not created at setup. `modularity/` empty tree; populated by `/modularize`.

### 6. `/migrate-arch` skill (brownfield)

**Trigger.** Run manually, or delegated from `/setup-project` Phase 2. Trigger logic uses `/check-setup`:

- Presence check passes → "nothing to migrate, exit."
- Presence check fails (files missing) → "migration needed, proceed."
- Presence passes but non-placeholder fails → "not a migration case; fill `ARCH_GAP`s manually."

**Process.**

1. **Scan** — read every file under existing `docs/architecture/current/` (recursively), **excluding only** `_migration-quarantine/**` (already-processed from a prior run). Files under `docs/architecture/logs/` are outside scope entirely. Build source inventory from the remainder — including `modularity/` and `estimation/` content, which legitimately feeds `c4/containers.md` and `bizbok/capabilities.md` subsections.
2. **Bucket via LLM interpretation.** For each of the 20 target files and each required section, prompt the LLM: "which source content belongs here?" Output extracted content + source path, or `NO_MATCH`. Per-section, not per-file — a single source file can feed many targets. One LLM call per target file so mis-assignment in one target doesn't cascade.
3. **Move source out of the way — before writing targets.** Sweep source-inventory paths so `docs/architecture/current/` is empty of source content at the paths the target-writer needs. Runs **before** the target-write so a source file at a colliding path (e.g., `README.md` at the current/ root) is preserved:
   - **Default (no `--clean`):** move sweep-set files to `docs/architecture/current/_migration-quarantine/<original-path>`. Extracted content will live in target files (written next, with provenance comments); the originals in quarantine are the audit trail.
   - **`--clean`:** skip quarantine, `git rm` every sweep-set file. The reshape lands without an audit archive.
   - **Sweep-set = scan-set minus the preserved set.** Preserved (read but never moved/rm'd): `modularity/**`, `estimation/**`, `_migration-quarantine/**`, and (out of scope) `docs/architecture/logs/**`. Modularity and estimation are project-owned; only `/modularize` writes modularity; the target-write reflects their content via provenance comments while the originals stay in place.
4. **Write targets.** Match → write section with content + provenance HTML comment (`<!-- migrated from 00-Overview/Scope.md on 2026-09-18 -->`). `NO_MATCH` → `ARCH_GAP` marker per §3 format. (After step 3 the target paths are empty.)
5. **Never invent.** If the source doesn't say something, the target section is `ARCH_GAP`. Never summarize adjacent content, never infer, never fill with "reasonable defaults."

**Output — one PR.** Branch `arch/migrate-schema`. Contents:

- **Reshape commit** — adds the 20-file schema with bucketed content + `ARCH_GAP` markers; moves all source (both extracted-from and unmatched) to `_migration-quarantine/`, or `git rm`s it with `--clean`.
- **Arch-log entry** — `docs/architecture/logs/YYYY-MM-DD-arch-schema-migration/README.md` with full block (satisfies `/arch-log-review`).
- **Migration report** — appended to PR description (not committed); checkable list of moves, gaps, and quarantined files.

Example migration report:

```
### Migration report

Moved:
  - 00-Overview/Scope.md → bizbok/capabilities.md § L1 capabilities
  - 00-Overview/Scope.md → bizbok/value-streams.md § Streams
  - 01-Architecture/System Overview.md → c4/context.md § Diagram

Gaps (ARCH_GAPs to fill before /check-setup passes):
  - [ ] bizbok/stakeholders.md § Stakeholder catalog
  - [ ] ddd/access-model.md § Policies
  - [ ] c4/deployment.md § Threat model

Quarantined (review + fold or delete):
  - _migration-quarantine/00-Overview/Design Decisions.md
  - _migration-quarantine/07-Configuration/Config Layer.md
```

**Human review flow.** The PR is the review gate. Reviewer:
1. Sanity-checks bucketing (LLM may miscategorize).
2. Empties the quarantine — folds each file into a lens or deletes. Content moved from quarantine into lens files becomes arch-log-worthy edits, folded into this same PR under the same arch-log entry.
3. Fills `ARCH_GAP` markers where the source had content the LLM missed, or leaves them for later.
4. Merges when `/check-setup` presence + cross-ref pass; non-placeholder gaps at merge time become the ongoing punch list, not a merge blocker.

**Re-run behavior.** Second run against partially-migrated repo:
- Presence passes → exit "nothing to migrate."
- Presence still fails (someone deleted a scaffolded file) → re-scaffold only the missing files with `ARCH_GAP`. Never overwrite existing populated content.

**Bucketing quality controls:**
- Prompt names the schema anchor + section purpose per target.
- Prompt shows the required-section headings so the LLM outputs to the right shape.
- One LLM call per target file (not global) — miscat in one target doesn't cascade.
- Every extracted chunk carries provenance — reviewer traces back easily.

### 7. Ripple to other skills

**`/new-feature` — biggest ripple.** Must:

1. **Read the schema** (`arch-schema.md`) to know which lens file(s) a feature-kind touches:
   - New capability → `bizbok/capabilities.md` + `bizbok/value-streams.md` (if adds a journey) + `glossary.md` (new terms)
   - New bounded context → `ddd/context-map.md` + new `ddd/contexts/<name>.md`
   - New service/container → `c4/containers.md` + `c4/deployment.md` (zone placement) + likely a new DDD context + new modularity node
   - New external integration → `c4/integrations.md` + `c4/context.md`
   - Access change → `ddd/access-model.md` + likely `bizbok/stakeholders.md` (new role)
   - New info concept → `bizbok/information-map.md` + `glossary.md` + likely a DDD context change
2. **Update cross-references** — a new capability entry must reference existing (or newly-added) context + container files.
3. **Fill new `ARCH_GAP`s** on every added file — never leave them without cause.
4. **Multi-lens PRs get a soft warning** — the "unusual" signal from §3.
5. **Paired arch-log entry** — unchanged existing rule.

**`/spec-review` Architecture sub-verdict.** Checklist updated to lens vocabulary:
- Does the spec change a capability? → does `bizbok/capabilities.md` reflect it?
- Does the spec cross a bounded context? → does `ddd/context-map.md` show the relationship?
- Does the spec introduce a new information concept? → does `bizbok/information-map.md` + `glossary.md` name it?
- Does the spec change access? → does `ddd/access-model.md` show the new policy?
- Does the spec change a container's data ownership? → does `c4/containers.md` reflect it?

Rule (unchanged): drift = finding; significant drift = escalate to C2 + require `/new-feature` first. Only vocabulary updates.

**`/modularize` — soft-convention header.** Every `modularity/<L1>/README.md` file written by `/modularize` must start:

```
Container: [<name>](../../c4/containers.md#<anchor>) · Context: [<name>](../../ddd/contexts/<name>.md)
```

`/check-setup` warns on absence; `/modularize` fills on every write.

**Minor / no direct change:**

- `/new-spec` — reads lens files as prior art (existing); prompt refers to lens vocabulary.
- `/plan-sprint` — cross-links navigate cleanly. No skill change.
- `/create-issues`, `/split-issue` — no change.
- `/sdd-rebase` — `docs/architecture/` stays excluded (project-owned); `.claude/skills/check-setup/arch-schema.md` is rebased (template-owned) so downstream repos pick up schema updates automatically.

**Hard rules — `docs/process/AGENTS.md`:**

- Layer-discipline rule wording updated to three-lens language (§3).
- New rule: **architecture files must conform to the schema.** Deviations require a schema change (edit `arch-schema.md`) with C2 discipline.
- New rule: **migration is one-way via `/migrate-arch`.** Never edit `docs/architecture/current/` freehand to reshape it — run the skill, get a reviewable PR.

## Contracts

**Schema contract** — `.claude/skills/check-setup/arch-schema.md` becomes a stable contract. Every skill that reads it commits to its section anchors. Adding a required file or section is a breaking change (existing downstream repos fail `/check-setup`); removing one is safe.

**`/check-setup` output contract** — machine-parseable failure lines per §3 report format. Skills that shell out to `/check-setup` (currently: none, but `/setup-project` Phase 6 does) commit to that line format.

**`ARCH_GAP` marker contract** — literal HTML comment with fields Section, Fill with, See. Parseable by grep + regex. Downstream tooling may key off it.

**`N/A because` contract** — literal phrase, ≥5-word reason follows. Grep-detectable.

**Modularity soft-convention header contract** — literal first line format per §7. Warn-only in `/check-setup`; `/modularize` writes it.

## Acceptance criteria

1. **Schema is authoritative.** `.claude/skills/check-setup/arch-schema.md` exists, is the sole home of the schema, and defines all 18 required files + their sections + non-placeholder rules + cross-reference rules per §2 and §4.
2. **`/check-setup` runs the four-check gate** (presence, non-placeholder, cross-references, estimation-coverage-if-present) with no version branching and no legacy fallback.
3. **`/check-setup` emits the report format** in §3 for every failure.
4. **`/setup-project` Phase 2 scaffolds** all 20 required files (17 content + 3 lens READMEs) + initial arch-log entry per §5. Immediately after run, `/check-setup` fails on non-placeholder for every `ARCH_GAP`.
5. **`/migrate-arch` exists** as a skill and behaves per §6: LLM-driven bucketing, gap marking, quarantine, PR output, arch-log entry, trigger logic via `/check-setup`.
6. **`/new-feature` reads the schema** and touches lens files per §7's mapping. Multi-lens PRs surface the soft warning.
7. **`/spec-review` Architecture sub-verdict** uses lens vocabulary per §7.
8. **`/modularize` writes** the soft-convention header on every `modularity/<L1>/README.md` write.
9. **`docs/process/AGENTS.md` hard rules updated** per §3 and §7 (layer discipline + schema conformance + one-way migration).
10. **The single real downstream project (`ai-sales-copilot-docs`)** can be migrated via `/migrate-arch` producing a reviewable PR with lens-shaped content, gaps, and quarantine.
11. **`README.md` in this repo** references the schema and directs new-repo users to `/setup-project` (which now scaffolds the full shape).

## Risks & assumptions

**Risks**

- **LLM bucketing quality varies.** `/migrate-arch` may miscategorize on unusual freestyle inputs. Mitigation: PR-based review; provenance comments; one LLM call per target (isolated failure); the human is the final arbiter.
- **`ARCH_GAP` fatigue.** A fresh project has 17 content files × ~3 sections × 1 marker each ≈ 50+ gaps. Teams may resent the noise. Mitigation: `/check-setup` prints as punch-list, `/new-feature` is the natural filler, `N/A because` escape hatch is available for genuinely-inapplicable sections.
- **Cross-reference brittleness.** Renaming a context requires updating every file that links to it. Mitigation: rules 2/3/6/7/10 catch dangling links immediately; `/check-setup` reports the specific broken location.
- **Schema drift between skills.** Six skills read the schema; a change to `arch-schema.md` must reach all six. Mitigation: skills reference by `@` import (single source), not by copy; `/sdd-rebase` propagates schema updates automatically to downstream repos.
- **Multi-lens PRs are only *warned*, not blocked.** A legitimate cross-lens change (new bounded context that adds a capability, a container, and a data flow) touches three lenses correctly. Warning is soft to avoid false positives.
- **Estimation coverage gate is dormant.** No project has adopted `estimation/` under the schema yet. Coverage check is untested at scale.

**Assumptions**

- Only one meaningful downstream project exists today (`ai-sales-copilot-docs`). One-way migration is affordable; versioning is not.
- Teams adopting this template have (or will have) senior enough architects to fill lens files meaningfully. If a team doesn't know what a "bounded context" is, they'll either learn (the DDD README explains) or `N/A because` most of the ddd/ dir — both are acceptable outcomes.
- `/check-setup` currently has grep-level access to arch files; extending it to the four-check gate is a matter of skill logic, not new infra.
- LLM-based bucketing in `/migrate-arch` is acceptable non-determinism because the PR review step catches mistakes. The alternative (rule-based mapping) doesn't scale to arbitrary source shapes.
- Downstream repos will pull the schema update via `/sdd-rebase` and either migrate immediately or accept a broken `/check-setup` until they do.

## Implementation notes (non-normative)

Not part of the spec contract, but the following are practical planning hints for the implementation plan:

- The 17 content-file schemas are largely independent — the plan can parallelize per-file schema work.
- `/migrate-arch` is the highest-risk skill; ship it after `/check-setup` + `/setup-project` scaffolding so it can be dogfooded against a synthesized freestyle input.
- `docs/process/AGENTS.md` hard-rule updates should land last, after all skills implement the schema (otherwise the rules describe behavior that isn't present yet).
- The `README.md` in this repo needs a section under "Repo layout" describing the schema at a glance for new users.
