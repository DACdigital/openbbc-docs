# discovery

## Purpose

Turn a client frontend repo into a structured `.flow-map/` wiki that the wizard consumes to
seed agent v1. Two audiences: (1) the runtime agent that will drive MCP tools wrapping the
backend, and (2) the engineer (or downstream generator) who will build that MCP server —
because no MCP server exists yet and someone has to specify it. Fully agent-driven; the skill
*is* a procedure Claude Code executes. Runs on the discovery author's machine, out of
`open-bbcd`'s DB.

## Aggregates & entities

- **`.flow-map/` bundle** (aggregate root; `schema_version: 2`) — directory tree emitted by
  the skill:
  - `AGENTS.md` — entry point + retrieval indices.
  - `APP.md` — app-wide invariants, conventions, boundaries.
  - `glossary.md` — one-page pivot table (skill → user phrases → endpoints → flows).
  - `skills/<id>.md` — one per business-domain specialty (primary read for the runtime agent).
  - `flows/<id>.md` — one playbook per user journey (intent, no HTTP detail).
  - `endpoints/<id>.md` — one per discovered backend call; HTTP detail lives here; every
    entry carries `proposed: true` because ids are derived from frontend call sites and never
    validated against any external registry.
  - `<name>.zip` — the archive uploaded via the wizard; `open-bbcd` stores it inline on
    `agents.discovery_zip BYTEA` (migration 026).

<!-- migrated from _migration-quarantine/ARCHITECTURE.md § flow-map-compiler, DESIGN.md § Phase 0 on 2026-09-28. Updated 2026-09-28 to match the flow-map-compiler skill's current schema v2 layout (AGENTS.md, APP.md, glossary.md, skills/, flows/, endpoints/) — the older `capabilities/`, `agents/` shape from ARCHITECTURE.md is stale. -->

## Domain events

`N/A because this context lives entirely inside a Claude Code skill run against a target repo and emits its output as a directory + zip, not as event messages`

## Invariants

- **LOCKED — one skill = one business domain.** Skills aggregate endpoints that share a
  domain vocabulary and invariants (e.g. `shopping`, `account`). No skill-per-endpoint. Even
  a single-endpoint domain gets a skill at the domain level.
- **LOCKED — no HTTP detail outside `endpoints/`.** Method, path, params, response shape,
  auth, source — only in endpoint frontmatter. Skill and flow files never carry HTTP detail.
- **LOCKED — the runtime word "tool" must not appear inside `.flow-map/`.** Endpoint
  frontmatter uses `endpoint`/`endpoint-id`, skills use `suggested_endpoints[]`, flows
  reference skills. Downstream (`aikdm`, bundle, runtime agent) calls these things tools; the
  translation is intentional and happens outside the discovery layer.
- **LOCKED — endpoints are the complete inventory.** Every backend call surfaced by call-site
  discovery becomes an `endpoints/<id>.md`, even when no skill suggests it
  (`used_by_skills[]` may be `[]`).
- **LOCKED — `suggested_endpoints[]` is advisory.** Discovery proposes; `aikdm` may add,
  drop, or re-annotate when wiring tools downstream.
- **LOCKED — `<!-- HUMAN id="..." -->` blocks survive regeneration verbatim** on flows,
  skills, and endpoints. `<!-- AGENT id="..." -->` blocks are regenerated. Material outside
  any block is structural.
- **LOCKED — output confined to `.flow-map/`.** The skill never modifies source files
  outside that directory.
- **LOCKED anti-goals: never generate MCP server code, runtime agent prompts, or call any
  registry API. Never run target-repo code (no `npm run dev`, no tests). Never assume an MCP
  server exists.**
- **Contract triple stays in sync.**
  `bbc-discovery/flow-map-compiler/references/output-schemas.md` ↔
  `references/lint-contract.md` ↔ `assets/templates/*.tmpl` — the schema, the linter, and the
  templates evolve together.
- **Skill is fully agent-driven** — no scripts, no build step.
- **Discovery scans the frontend repo only** — never the backend. Backend surface is
  discovered indirectly via call-site analysis (fetch / axios / ky / @tanstack/react-query /
  swr / Apollo / urql / graphql-request / @trpc/client / openapi-fetch / Next.js Server
  Actions).

<!-- migrated from _migration-quarantine/ARCHITECTURE.md § flow-map-compiler, DESIGN.md § Phase 0 on 2026-09-28. Updated 2026-09-28 with LOCKED rules + anti-goals from the flow-map-compiler skill (bbc-discovery/flow-map-compiler/skills/flow-map-compiler/SKILL.md). -->

## Published surface

- **Consumed by `agent-lifecycle`:** the `.flow-map/` zip uploaded through the
  `/agents/new` wizard (schema-driven from `web/schemas/wizard-v1.yaml`); stored inline on
  `agents.discovery_zip BYTEA` (migration 026), not on a persistent volume.
- **Contract:** `bbc-discovery/flow-map-compiler/references/output-schemas.md` (schema
  version 2) + `references/lint-contract.md` (15 rules the agent walks before shipping).
- **Not exposed over REST.** The skill runs client-side inside Claude Code; `open-bbcd` sees
  only the uploaded zip.
- **Not an MCP server generator.** Turning `endpoints/<id>.md` into a runnable MCP surface
  is downstream engineering — see `bizbok/capabilities.md § mcp-over-rest-bridge` for the
  built-in `http_endpoint` bridge in `open-bbcd`, and `assumptions.md § Open questions` for
  the "MCP-server generator" placeholder.

<!-- migrated from _migration-quarantine/ARCHITECTURE.md § flow-map-compiler, DESIGN.md § Phase 0, § Phase I on 2026-09-28. Updated 2026-09-28. -->
