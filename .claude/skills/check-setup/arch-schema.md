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

## Diagram conventions {#diagram-conventions}

Every required diagram in the schema uses one of the mermaid syntaxes below. Skills that
write diagram sections (`/migrate-arch`, `/new-feature`, `/setup-project` scaffolder)
consult this section for the syntax + a minimal template; each file schema below points
back here in its non-placeholder rule where a diagram is required. Keeping the syntax
uniform across projects means downstream renderers and `/arch-log-review` can rely on a
stable shape.

### `README.md § System at a glance`

Mermaid `flowchart TB`. ≥3 labelled boxes, at least one `subgraph` grouping owned
components, edges labelled with the protocol.

    ```mermaid
    flowchart TB
        User[End user]

        subgraph OURS[What we own]
            FE[Frontend]
            API[API service]
            DB[(Database)]
        end

        User -- HTTPS --> FE
        FE -- REST --> API
        API -- SQL --> DB
    ```

### `c4/context.md § Diagram`

Mermaid `C4Context` (native C4 primitives). `Person` for actors, `System` for the system
under design, `System_Ext` for external systems. Edges use `Rel`/`BiRel` with a technology
label.

    ```mermaid
    C4Context
        Person(user, "End user", "primary actor")
        System(sys, "System X", "what we ship")
        System_Ext(ext, "Third-party service", "SaaS")

        Rel(user, sys, "uses", "HTTPS")
        Rel(sys, ext, "reads from", "REST")
    ```

### `c4/containers.md § Diagram`

Mermaid `C4Container`. `Container`, `ContainerDb`, `Container_Ext` grouped in
`System_Boundary(...)`. Include the tech stack in the third parameter.

    ```mermaid
    C4Container
        Person(user, "End user")

        System_Boundary(sys, "System X") {
            Container(fe, "Frontend", "React", "Web UI")
            Container(api, "API", "Node/Express", "REST API")
            ContainerDb(db, "Database", "Postgres", "Owns core domain data")
        }

        Rel(user, fe, "uses", "HTTPS")
        Rel(fe, api, "REST", "JSON/HTTPS")
        Rel(api, db, "reads/writes", "SQL")
    ```

### `c4/data-flows.md § <flow>` (one per canonical flow)

Mermaid `sequenceDiagram`. One `participant` per container involved, `Note over` for
phase markers, `alt`/`par` for branching or parallel paths.

    ```mermaid
    sequenceDiagram
        participant U as User
        participant API as API service
        participant DB as Database

        U->>API: POST /resource
        API->>DB: INSERT
        DB-->>API: id
        API-->>U: 201 Created
    ```

### `c4/deployment.md § Diagram`

Mermaid `flowchart TB`. One `subgraph` per network zone, containers nested inside;
trust boundaries marked via a `stroke-dasharray` class on the boundary subgraph.

    ```mermaid
    flowchart TB
        subgraph PUBLIC[Public zone]
            LB[Load balancer]
        end

        subgraph APP[App zone]
            FE[Frontend]
            API[API]
        end

        subgraph DATA[Data zone — trust boundary]
            DB[(Database)]
        end

        LB --> FE
        FE --> API
        API --> DB

        classDef trust stroke-dasharray: 5 5
        class DATA trust
    ```

### `ddd/context-map.md § Diagram`

Mermaid `flowchart LR`. One node per bounded context (short kebab-case ID as label);
edges labelled with the relationship pattern using one-letter codes plus a short verb:
`U/D` (upstream/downstream), `ACL` (anti-corruption layer), `C` (conformist),
`P` (partnership), `OHS` (open host service).

    ```mermaid
    flowchart LR
        AC[access-control]
        IN[ingestion]
        RE[retrieval]
        AO[agent-orchestration]

        AC -- ACL: identity header --> AO
        AC -- ACL: identity header --> RE
        IN -- P: joint Qdrant schema --> RE
        RE -- OHS: MCP tools --> AO
    ```

### `bizbok/value-streams.md § <stream>` (optional per stream)

Mermaid `sequenceDiagram` (for interaction flows) or `flowchart LR` (for step ladders).
Each step names the capability it exercises via a bracketed reference.

    ```mermaid
    flowchart LR
        T[Trigger:<br/>new lead] --> S1[Qualify<br/>lead-qualification]
        S1 --> S2[Score<br/>opportunity-scoring]
        S2 --> O[Outcome:<br/>opportunity created]
    ```

---

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

**Non-placeholder rule:** diagram present with ≥3 labelled boxes per `#diagram-conventions` (`flowchart TB`); map-of-content lists every required file.

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

**Non-placeholder rule:** ≥1 stream with ≥3 steps; every step names the capability from `bizbok/capabilities.md` it exercises (as an inline link); optionally render each stream as a diagram per `#diagram-conventions`.

### `bizbok/information-map.md` {#bizbok-information-map}

**Required content:** a single table `Concept | Description | Owning capability | Regulatory tag`.

**Non-placeholder rule:** ≥3 concepts; each concept also appears in `glossary.md`.

---
## ddd/ — domain layer

### `ddd/README.md` {#ddd-readme}

**Required sections:** `## What DDD is`; `## What lives here`; `## How it connects to BIZBOK and C4`.

**Non-placeholder rule:** 2–3 sentence lens explainer; cross-links to `bizbok/README.md` and `c4/README.md`.

### `ddd/context-map.md` {#ddd-context-map}

**Required sections (in order):**
- `## Contexts` — list of bounded contexts, each linking to its `contexts/<name>.md`
- `## Relationships` — per pair of contexts: upstream/downstream / ACL / conformist / partnership
- `## Diagram` — mermaid diagram

**Non-placeholder rule:** ≥1 context; every listed context has a matching `contexts/<name>.md`; every relationship in the pair table is labelled with its pattern; diagram per `#diagram-conventions` (`flowchart LR`, one-letter pattern codes).

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

**Non-placeholder rule:** diagram per `#diagram-conventions` (mermaid `C4Context`) + ≥1 external actor + ≥1 external system + system-boundary statement.

### `c4/containers.md` {#c4-containers}

**Required sections (in order):**
- `## Diagram` — mermaid C4-container diagram
- One `### <Container>` subsection per container, each containing: purpose, tech stack, data ownership, published API / events, link to its DDD context (`ddd/contexts/<name>.md`), link to its modularity node (`modularity/<slug>/README.md`) if the tree is populated

**Non-placeholder rule:** ≥1 container; each has a data-ownership statement (not `N/A`); diagram per `#diagram-conventions` (mermaid `C4Container` with `System_Boundary`).

### `c4/data-flows.md` {#c4-data-flows}

**Required content:** one subsection per canonical flow (e.g. `## Write path`, `## Read path`, `## Event fan-out`), each with a sequence or flowchart diagram + narrative.

**Non-placeholder rule:** ≥1 canonical flow, each rendered as a `sequenceDiagram` per `#diagram-conventions`.

### `c4/deployment.md` {#c4-deployment}

**Required sections (in order):**
- `## Environments`
- `## Network zones`
- `## Trust boundaries`
- `## Threat model` — STRIDE-lite table `Boundary | Threat | Mitigation`

**Non-placeholder rule:** ≥1 environment; every container in `c4/containers.md` is placed in ≥1 zone; threat model has ≥1 threat per trust boundary; deployment diagram per `#diagram-conventions` (`flowchart TB` with per-zone subgraphs and dashed trust-boundary styling).

### `c4/integrations.md` {#c4-integrations}

**Required sections (in order):**
- One table `System | Kind | Protocol | Auth | SLA/regulatory`
- `## Contracts` — versioned schemas per integration

**Non-placeholder rule:** ≥1 integration OR the explicit statement `N/A because this system has no external integrations of any kind`.

---
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
