# agent-lifecycle

## Purpose

Own the identity, versioning, wiring, and deployment status of AI agents. Sits between
`discovery` (upstream — supplies the `.flow-map/`) and `deployed-runtime` (downstream — runs
the DEPLOYED version); partners with `evaluation` and `training` on version-scoped scoring
and improvement.

## Aggregates & entities

- **Agent** (aggregate root) — `agents` row. Owns `architecture` JSONB (flows, endpoints,
  endpoint→backend wiring), `capabilities[]` pass-through of the discovery output, and the
  linked-list head of versions via `parent_version_id`. **Structural fields are frozen on
  first-version creation** (migration 017 split).
- **Agent version** (entity within Agent aggregate) — `agent_versions` row. Owns editable
  `prompts` JSONB (`main_prompt` + skill prompts), MCP attachments (`agent_version_mcp_backend`,
  migration 015), and a deployed-flag semantics enforced at the agent-chain scope.
- **Tool backend** (aggregate root) — `tool_backends` row. `kind ∈ {http_endpoint, mcp_client}`,
  opaque `config` JSONB. Independent life-cycle from any single agent — reused across agents.
- **Endpoint→backend wiring** (entity within Agent aggregate) — `agent_endpoint_backend` row
  (migration 017). Agent-keyed, not version-keyed: same wiring for every version of the same
  agent for a given endpoint id.
- **MCP attachment** (entity within Agent-version) — `agent_version_mcp_backend` row.
  Version-keyed; carries an editable `note`.

<!-- migrated from _migration-quarantine/ARCHITECTURE.md § PostgreSQL, § MCP wiring, DESIGN.md § Phase I, § Versioning on 2026-09-28 -->

## Domain events

<!-- ARCH_GAP: source describes state changes but does not name explicit domain events (event bus / outbox / message topics). Handler code likely emits nothing beyond DB writes.
     Section: Domain events
     Fill with: the events this context intends to publish (e.g. `AgentVersionCreated`, `AgentDeployed`, `AgentDeploymentRotated`, `ToolBackendRegistered`) with payload shapes.
     See: .claude/skills/check-setup/arch-schema.md#ddd-context-file -->

## Invariants

- **Structural agent fields are frozen** after the first version is created. `agents.architecture`
  and `capabilities[]` never mutate for the lifetime of the agent (migration 017).
- **Endpoint→backend wiring is agent-scoped** — every version of the same agent sees the same
  wiring for a given endpoint id.
- **MCP attachments are version-scoped** — different versions may attach different MCPs (or
  the same MCP with different `note` guidance).
- **At most one DEPLOYED version per agent chain** (migration 011, DB-enforced singleton).
  Deploying a new version implicitly rotates the previous one.
- **Version linked list is append-only** — `agents.parent_version_id` grows; versions are
  never deleted.
- **Tool backend test-connection precedes save** — `POST /mcp/test` pings a backend before
  saving via `POST /mcp`.

<!-- migrated from _migration-quarantine/ARCHITECTURE.md § MCP wiring, DESIGN.md § Versioning, PRODUCTION.md § 2.1 Mark a version as deployed on 2026-09-28 -->

## Published surface

- **REST:**
  - `POST /agents/{agent_id}/deploy` / `POST /agents/{agent_id}/undeploy`
  - `GET/POST /mcp*` (`/mcp/new`, `/mcp/test`, `/mcp/{id}`, `/mcp/{id}/edit`, `/mcp/{id}/delete`)
  - `GET/POST /agents/{id}/configure/architecture/endpoints[/bulk|/{endpointID}/backend]`
  - `GET/POST /agent_versions/{id}/configure/architecture/mcp` and per-attachment
    `/toggle`, `/notes`
- **Backoffice UI routes:** `/agents/ui`, `/agents/new` (wizard, schema-driven from
  `web/schemas/wizard-v1.yaml`), `/agents/{agent_id}/configure/*`,
  `/agent_versions/{version_id}/configure/*`.
- **Consumed contract:** `.flow-map/` zip from `discovery` (schema in
  `bbc-discovery/flow-map-compiler/references/output-schemas.md`).
- **Emitted contract:** aikdm `bundle.yaml` schema (`aikdm/schemas/prompt-v1.yaml`).

<!-- migrated from _migration-quarantine/ARCHITECTURE.md § Backoffice UI, § REST API, § MCP wiring, DESIGN.md § Phase I on 2026-09-28 -->
