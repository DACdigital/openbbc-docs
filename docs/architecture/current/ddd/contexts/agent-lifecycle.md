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
  (`agent_version_mcp_backend`, migration 015), the `agent_tool_enabled` flag (agent-tool
  checkbox), and deployed-flag semantics enforced at the agent-chain scope.
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
- **Sub-agent binding** (entity within Agent-version) — `agent_version_subagent` row on the
  **caller** version: `caller_version_id`, pinned `target_version_id` (FK
  `agent_versions`, may belong to any agent), tool-facing `name` (unique per caller; becomes
  a value of the agent tool's `subagent` enum), editable `note` (prompt guidance on when /
  how to delegate, rendered into the agent tool description). Only effective when the
  caller's `agent_tool_enabled = true`. The bindings reachable from a root version form its
  **agent topology**.

<!-- migrated from _migration-quarantine/ARCHITECTURE.md § PostgreSQL, § MCP wiring, DESIGN.md § Phase I, § Versioning on 2026-09-28. Updated 2026-09-28 for OpenBBC PR #50 (mig 025 PENDING state + 026 agents.discovery_zip BYTEA; clarified tool_backends kinds as bridge vs proxy). -->

## Domain events

N/A because OpenBBC does not emit domain events on any transport. State transitions in
this context are Postgres-only — `INSERT` / `UPDATE` against `agents`, `agent_versions`,
`agent_endpoint_backend`, `agent_version_mcp_backend`, `agent_version_subagent`,
`tool_backends` — and downstream
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
- **Version linked list grows by forking.** `agents.parent_version_id` links each new
  version to its parent. Versions and agents can be deleted
  (`POST /agent_versions/{version_id}/delete`, `POST /agents/{agent_id}/delete`). A delete is
  refused with `409` when it would remove a sub-agent binding target, the pinned version of
  a deployed child session, or a version with locked BO sessions pinned to it. An agent
  delete ignores references from its own cascade set: its versions' bindings, its deployed
  session trees, and BO session trees rooted on its versions.
- **Version status state machine** (migration 025): `INITIALIZING → PENDING → READY` via the
  alpha drainer; `READY → TRAINING → READY` on hill-climb (transient); `READY → DEPLOYED` on
  deploy; `DEPLOYED → READY` on rotation. Wizard Finalize is the `INITIALIZING → PENDING`
  transition; `seed_bundle.py` (invoked from `process_pending_alphas.sh`) is the
  `PENDING → READY` transition and writes both the bundle and the status flip in the same
  Postgres transaction.
- **Agent-tool config is version-scoped on the caller.** Different versions of the same
  agent may enable / disable the agent tool and bind different sub-agents, because the
  version's prompt decides how the tool is used. Config is writable only while the caller
  is `INITIALIZING`/`DRAFT` (`409` otherwise). A fork (prompt save, land, training
  complete) copies it onto the new version.
- **Sub-agent targets are pinned versions.** `target_version_id` is never re-resolved at
  call time (no "follow DEPLOYED"); the target must be `READY` or `DEPLOYED` when the
  binding is saved.
- **Agent topology is a DAG.** It is acyclic by construction: only `INITIALIZING`/`DRAFT`
  callers bind, only `READY`/`DEPLOYED` targets are bound, and status only moves forward. A
  binding that would make the caller reachable from the target (including self-binding) is
  also refused at save (repo layer), as defence in depth. See
  [`../../constraints.md`](../../constraints.md).
- **New versions inherit agent-tool config verbatim.** When training (or any other path)
  materialises a new version from a parent, `agent_tool_enabled` and every
  `agent_version_subagent` row are copied unchanged.
- **Tool backend test-connection precedes save** — `POST /mcp/test` pings a backend before
  saving via `POST /mcp`.
- **`http_endpoint` and `mcp_client` are the only backend kinds.** `http_endpoint` bridges a
  REST endpoint as MCP; `mcp_client` proxies to an existing MCP server. Any other integration
  pattern needs a new kind + new migration. The agent tool is **not** a `tool_backends`
  kind — it is a built-in tool dispatched in-process against another agent version.

<!-- migrated from _migration-quarantine/ARCHITECTURE.md § MCP wiring, DESIGN.md § Versioning, PRODUCTION.md § 2.1 Mark a version as deployed on 2026-09-28. Updated 2026-09-28 for OpenBBC PR #50 (mig 025 status state machine). Updated 2026-09-30 for multiagent-tools — agent-tool invariants. Updated 2026-10-05 for sync-multiagent-feature — version/agent delete with 409 guards, config writable only on INITIALIZING/DRAFT, DAG acyclic by construction. -->

## Published surface

- **REST:**
  - `POST /agents/{agent_id}/deploy` / `POST /agents/{agent_id}/undeploy`
  - `GET/POST /mcp*` (`/mcp/new`, `/mcp/test`, `/mcp/{id}`, `/mcp/{id}/edit`, `/mcp/{id}/delete`)
  - `GET/POST /agents/{id}/configure/architecture/endpoints[/bulk|/{endpointID}/backend]`
  - `GET /agent_versions/{id}/configure/mcp`; writes
    `POST /agent_versions/{id}/architecture/mcp/{backendID}/toggle` and bulk
    `POST …/architecture/mcp/notes`
  - `GET /agent_versions/{id}/configure/agents` — Agents tab (agent-tool checkbox, sub-agent
    binding list, target picker over `READY`/`DEPLOYED` versions of any agent; read-only
    outside `INITIALIZING`/`DRAFT`). Writes: `POST /agent_versions/{id}/architecture/agents/toggle`,
    `POST …/architecture/agents` (add binding), bulk `POST …/architecture/agents/notes`,
    `POST …/architecture/agents/{name}/delete`. They return HTML fragments: `409` when the
    version is locked or on a name/target conflict, `400` when the target is not runnable or
    a cycle would form.
  - `POST /agent_versions/{version_id}/delete` and `POST /agents/{agent_id}/delete` — `409`
    when a binding-target, deployed-child or locked-session reference would be removed.
- **Backoffice UI routes:** `/agents/ui`, `/agents/new` (wizard, schema-driven from
  `web/schemas/wizard-v1.yaml`), `/agents/{agent_id}/configure/*`,
  `/agent_versions/{version_id}/configure/*`.
- **Consumed contract:** `.flow-map/` zip from `discovery` (schema in
  `bbc-discovery/flow-map-compiler/references/output-schemas.md`).
- **Emitted contract:** aikdm `bundle.yaml` schema (`aikdm/schemas/prompt-v1.yaml`).

<!-- migrated from _migration-quarantine/ARCHITECTURE.md § Backoffice UI, § REST API, § MCP wiring, DESIGN.md § Phase I on 2026-09-28. Updated 2026-10-05 for sync-multiagent-feature — MCP and Agents-tab routes, version/agent delete. -->
