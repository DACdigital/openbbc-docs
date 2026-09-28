# discovery

## Purpose

Turn a client frontend repo into a structured `.flow-map/` wiki that the wizard consumes to
seed agent v1. Fully agent-driven — the skill *is* a procedure Claude Code executes; there is
no scripted pipeline. Runs on the discovery author's machine, out of `open-bbcd`'s DB.

## Aggregates & entities

- **`.flow-map/` bundle** (aggregate root) — directory tree emitted by the skill:
  - `flows/` — business flows, tool-name-free.
  - `capabilities/` — backend endpoints with proposed tool names.
  - `agents/` — proposed agent scaffolds.
  - `<name>.zip` — the archive uploaded via the wizard.

<!-- migrated from _migration-quarantine/ARCHITECTURE.md § flow-map-compiler, DESIGN.md § Phase 0 on 2026-09-28 -->

## Domain events

`N/A because this context lives entirely inside a Claude Code skill run against a target repo and emits its output as a directory + zip, not as event messages`

## Invariants

- **Contract triple stays in sync.**
  `bbc-discovery/flow-map-compiler/references/output-schemas.md` ↔
  `references/lint-contract.md` ↔ `assets/templates/*.tmpl` — the schema, the linter, and the
  templates evolve together.
- **Skill is fully agent-driven** — no scripts, no build step. The skill *is* the procedure.
- **Discovery scans the frontend repo only** — never the backend. Backend surface is
  discovered indirectly via call-site analysis.

<!-- migrated from _migration-quarantine/ARCHITECTURE.md § flow-map-compiler, DESIGN.md § Phase 0 on 2026-09-28 -->

## Published surface

- **Consumed by `agent-lifecycle`:** the `.flow-map/` zip uploaded through the
  `/agents/new` wizard (schema-driven from `web/schemas/wizard-v1.yaml`).
- **Contract:** `bbc-discovery/flow-map-compiler/references/output-schemas.md`.
- **Not exposed over REST.** The skill runs client-side; `open-bbcd` sees only the uploaded
  zip.

<!-- migrated from _migration-quarantine/ARCHITECTURE.md § flow-map-compiler, DESIGN.md § Phase 0, § Phase I on 2026-09-28 -->
