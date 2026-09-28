# agent-lifecycle

## Purpose

Own the identity, versioning, wiring, and deployment status of AI agents. Sits between
`discovery` (upstream — supplies the `.flow-map/`) and `deployed-runtime` (downstream — runs
the DEPLOYED version); partners with `evaluation` and `training` on version-scoped scoring
and improvement.

## Aggregates & entities

- **Agent** (aggregate root) — `agents` row. Owns `architecture` JSONB (flows, endpoints,
  endpoint→backend wiring), `capabilities[]` pass-through of the discovery output,
  `discovery_zip` (BYTEA, migration 026 — the uploaded `.flow-map/` archive stored inline),
  and the linked-list head of versions via `parent_version_id`. **Structural fields are
  frozen on first-version creation** (migration 017 split). The deprecated
  `discovery_file_path` column is retained but ignored.
- **Agent version** (entity within Agent aggregate) — `agent_versions` row. Owns
  `status ∈ {INITIALIZING, PENDING, DRAFT, TRAINING, READY, DEPLOYED}` (migration 025 added
  `PENDING`), editable `prompts` JSONB (`main_prompt` + skill prompts), MCP attachments
  (`agent_version_mcp_backend`, migration 015), and deployed-flag semantics enforced at the
  agent-chain scope.
- **Tool backend** (aggregate root) — `tool_backends` row. `kind ∈ {http_endpoint, mcp_client}`,
  opaque `config` JSONB. `http_endpoint` = OpenBBC's built-in **MCP-over-REST bridge** (open-bbcd
  calls a plain REST endpoint and exposes it to the agent as an MCP tool). `mcp_client` =
  proxy to an existing MCP server. Independent life-cycle from any single agent — reused
  across agents.
- **Endpoint→backend wiring** (entity within Agent aggregate) — `agent_endpoint_backend` row
  (migration 017). Agent-keyed, not version-keyed: same wiring for every version of the same
  agent for a given endpoint id.
- **MCP attachment** (entity within Agent-version) — `agent_version_mcp_backend` row.
  Version-keyed; carries an editable `note`.

<!-- migrated from _migration-quarantine/ARCHITECTURE.md § PostgreSQL, § MCP wiring, DESIGN.md § Phase I, § Versioning on 2026-09-28. Updated 2026-09-28 for OpenBBC PR #50 (mig 025 PENDING state + 026 agents.discovery_zip BYTEA; clarified tool_backends kinds as bridge vs proxy). -->

## Domain events

N/A because OpenBBC does not emit domain events on any transport. State transitions in
this context are Postgres-only — `INSERT` / `UPDATE` against `agents`, `agent_versions`,
`agent_endpoint_backend`, `agent_version_mcp_backend`, `tool_backends` — and downstream
consumers poll the REST surface for status changes (the alpha drainer, for instance,
enumerates `GET /agent_versions.json?status=PENDING`).

## Invariants

- **Structural agent fields are frozen** after the first version is created.
  `agents.architecture`, `agents.capabilities[]`, and `agents.discovery_zip` never mutate for
  the lifetime of the agent (migration 017 + 026).
- **Endpoint→backend wiring is agent-scoped** — every version of the same agent sees the same
  wiring for a given endpoint id.
- **MCP attachments are version-scoped** — different versions may attach different MCPs (or
  the same MCP with different `note` guidance).
- **At most one DEPLOYED version per agent chain** (migration 011, DB-enforced singleton).
  Deploying a new version implicitly rotates the previous one.
- **Version linked list is append-only** — `agents.parent_version_id` grows; versions are
  never deleted.
- **Version status state machine** (migration 025): `INITIALIZING → PENDING → READY` via the
  alpha drainer; `READY → TRAINING → READY` on hill-climb (transient); `READY → DEPLOYED` on
  deploy; `DEPLOYED → READY` on rotation. Wizard Finalize is the `INITIALIZING → PENDING`
  transition; `seed_bundle.py` (invoked from `process_pending_alphas.sh`) is the
  `PENDING → READY` transition and writes both the bundle and the status flip in the same
  Postgres transaction.
- **Tool backend test-connection precedes save** — `POST /mcp/test` pings a backend before
  saving via `POST /mcp`.
- **`http_endpoint` and `mcp_client` are the only backend kinds.** `http_endpoint` bridges a
  REST endpoint as MCP; `mcp_client` proxies to an existing MCP server. Any other integration
  pattern needs a new kind + new migration.

<!-- migrated from _migration-quarantine/ARCHITECTURE.md § MCP wiring, DESIGN.md § Versioning, PRODUCTION.md § 2.1 Mark a version as deployed on 2026-09-28. Updated 2026-09-28 for OpenBBC PR #50 (mig 025 status state machine). -->

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
