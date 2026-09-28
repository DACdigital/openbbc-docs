@AGENTS.md

## Levels

Two independent "level" axes — do not confuse them. See `README.md#levels` for full tables.

- **Architecture tree levels (L1 / L2 / L3)** — depth in the modularity tree under
  `docs/architecture/current/modularity/`. Managed by `/modularize`.
  - **L1** Architecture — services / infra entities.
  - **L2** Service — contract modules within a service.
  - **L3** Implementation — issue-sized shards; optional scope-grounding for `/split-issue` when a feature ticket needs breaking down.
  Only L1 and L2 are architecture-of-record.
- **Change classification (C0 / C1 / C2)** — how much process a change goes through.
  - **C0** trivial — fast lane, no spec.
  - **C1** normal — spec → `/spec-review` → issues → code PR.
  - **C2** architectural/data/contract — same as C1; `/spec-review` covers architecture and contracts inline. Optional post-approval `/arch-review <feature>` syncs `docs/architecture/current/`.

The two axes are independent: an L2 module may host either a C1 or a C2 change.

## Project context

Filled by `/setup-project`:

- **Project:** `openbbc-docs`
- **Stack:** Go + Python (uv), PostgreSQL, htmx, AG-UI streaming, MCP-mediated backends; Claude Code plugin (`bbc-discovery/flow-map-compiler`)
- **Service repos:** `openbbc` (Go/Python monorepo — see `docs/repos/openbbc.md` after Phase 3)
- **Tracker:** platform `github`, org `DACdigital`, project `openbbc-docs`
- **Git host:** platform `github`, org `DACdigital`
