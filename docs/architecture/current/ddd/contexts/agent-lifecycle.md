# agent-lifecycle

## Purpose

Own the identity, versioning, wiring, and deployment status of AI agents, including their
portable identity across clusters (static IDs) and the declarative seed import/export that
carries a version from one cluster to another. Sits between
`discovery` (upstream — supplies the `.flow-map/`) and `deployed-runtime` (downstream — runs
the DEPLOYED version); partners with `evaluation` and `training` on version-scoped scoring
and improvement.

## Aggregates & entities

- **Agent** (aggregate root) — `agents` row. Owns `architecture` JSONB (flows, endpoints,
  endpoint→backend wiring), `capabilities[]` pass-through of the discovery output,
  `discovery_zip` (BYTEA, migration 026 — the uploaded `.flow-map/` archive stored inline),
  and the linked-list head of versions via `parent_version_id`. **Structural fields are
  frozen on first-version creation** (migration 017 split). The deprecated
  `discovery_file_path` column is retained but ignored. Optional write-once `static_id`
  (cross-cluster identity; see invariants).
- **Agent version** (entity within Agent aggregate) — `agent_versions` row. Owns
  `status ∈ {INITIALIZING, PENDING, DRAFT, TRAINING, READY, DEPLOYED}` (migration 025 added
  `PENDING`), editable `prompts` JSONB (`main_prompt` + skill prompts), MCP attachments
  (`agent_version_mcp_backend`, migration 015), the `agent_tool_enabled` flag (agent-tool
  checkbox), and deployed-flag semantics enforced at the agent-chain scope.
  `parent_version_id` names the **previous version in the agent's history** (not
  necessarily the version its content was copied from); the version no other version
  names is the agent's **tip**. Optional write-once `static_id`.
- **Tool backend** (aggregate root) — `tool_backends` row. `kind ∈ {http_endpoint, mcp_client}`,
  opaque `config` JSONB. `http_endpoint` = OpenBBC's built-in **MCP-over-REST bridge** (open-bbcd
  calls a plain REST endpoint and exposes it to the agent as an MCP tool). `mcp_client` =
  proxy to an existing MCP server. Independent life-cycle from any single agent — reused
  across agents. `name` is unique and serves as its static key. `managed_by_seed` marks a
  **managed backend** owned by the current seed set.
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
- **Seed set** (value object, not persisted) — a parsed `openbbc-seed-v1` document set:
  backends by `name`, agents by `static_id` (architecture, Inputs config, endpoint wiring
  by backend name), versions by `static_id` (prompts, agent-tool flag, MCP attachments +
  notes by backend name, sub-agent bindings by target static ID, deploy marker
  `none`/`if_none`/`force`). Self-contained and validated structurally before any write.
  Its versions' identity is checked against the DB by a **content hash** over prompts,
  agent-tool flag, MCP set and sub-agent bindings.

<!-- migrated from _migration-quarantine/ARCHITECTURE.md § PostgreSQL, § MCP wiring, DESIGN.md § Phase I, § Versioning on 2026-09-28. Updated 2026-09-28 for OpenBBC PR #50 (mig 025 PENDING state + 026 agents.discovery_zip BYTEA; clarified tool_backends kinds as bridge vs proxy). Updated 2026-11-24 for declarative-seed — static IDs, tip/previous-in-history lineage, managed backends, Seed set value object. -->

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
  the lifetime of the agent (migration 017 + 026). No writer rewrites a finalized
  architecture: the seed import inserts an agent's architecture once and never updates it,
  and `seed_bundle.py --force` no longer rewrites it once the agent is finalized. The seed
  import relies on this as its agent identity check (see
  [`../../assumptions.md`](../../assumptions.md)).
- **Endpoint→backend wiring is agent-scoped** — every version of the same agent sees the same
  wiring for a given endpoint id.
- **MCP attachments are version-scoped** — different versions may attach different MCPs (or
  the same MCP with different `note` guidance).
- **At most one DEPLOYED version per agent chain** (migration 011, DB-enforced singleton).
  Deploying a new version implicitly rotates the previous one.
- **Versions are immutable outside `INITIALIZING`/`DRAFT`.** Every per-version write
  (MCP visibility toggle and notes, agent-tool config) is refused with `409` once the
  version is `PENDING`, `READY`, `TRAINING` or `DEPLOYED`; the root version's Inputs config
  is stricter (writable only while `INITIALIZING`). The guard lives in the
  repository layer under a row lock on the version, so no caller (handler, script or
  import) can bypass it. Prompts are never edited in place — saving creates a new version.
  The one writer allowed on a `PENDING` version is alpha landing (`PENDING → READY`), and it
  writes only to `PENDING` versions (with `--force`, `INITIALIZING`/`PENDING`). This is what
  makes a static ID mean the same content on every cluster.
- **Version linked list grows by appending at the tip.** Every path that creates a version
  from an existing one — prompt save, prompt land, training completion, seed import — sets
  `parent_version_id` to the agent's current **tip**, not to the version the content came
  from (content is still copied from the source). So `parent_version_id` means "previous in
  history", the chain stays strictly linear, and every new version gets the next `v[N]`;
  forking from a non-tip version (e.g. the DEPLOYED one) succeeds. All forks, the seed
  import and deploy serialize on the agent row: lock the agent first, then version rows in
  ascending id order, and read the tip only after taking the agent lock. Concurrent forks
  therefore land as consecutive versions. Versions and agents can be deleted
  (`POST /agent_versions/{version_id}/delete`, `POST /agents/{agent_id}/delete`). A delete is
  refused with `409` when it would remove a sub-agent binding target, the pinned version of
  a deployed child session, or a version with locked BO sessions pinned to it. An agent
  delete ignores references from its own cascade set: its versions' bindings, its deployed
  session trees, and BO session trees rooted on its versions.
- **Version status state machine** (migration 025): `INITIALIZING → PENDING → READY` via the
  alpha drainer; `READY → TRAINING → READY` on hill-climb (transient); `READY → DEPLOYED` on
  deploy; `DEPLOYED → READY` on rotation. The seed import is a second entry point: it
  inserts versions **directly as `READY`** (skipping the alpha drainer) under an agent it
  inserts **directly finalized**. Wizard Finalize is the `INITIALIZING → PENDING`
  transition; `seed_bundle.py` (invoked from `process_pending_alphas.sh`) is the
  `PENDING → READY` transition and writes both the bundle and the status flip in the same
  Postgres transaction.
- **Agent-tool config is version-scoped on the caller.** Different versions of the same
  agent may enable / disable the agent tool and bind different sub-agents, because the
  version's prompt decides how the tool is used. Config is writable only while the caller
  is `INITIALIZING`/`DRAFT` (`409` otherwise). A fork (prompt save, land, training
  complete) copies it onto the new version. A seeded version takes its agent-tool config
  and bindings **from the seed file**, not from a parent.
- **Sub-agent targets are pinned versions.** `target_version_id` is never re-resolved at
  call time (no "follow DEPLOYED"); the target must be `READY` or `DEPLOYED` when the
  binding is saved.
- **Agent topology is a DAG.** It is acyclic by construction: only `INITIALIZING`/`DRAFT`
  callers bind, only `READY`/`DEPLOYED` targets are bound, and status only moves forward. A
  binding that would make the caller reachable from the target (including self-binding) is
  also refused at save (repo layer), as defence in depth. **The seed import is the one
  sanctioned exception** to "only `INITIALIZING`/`DRAFT` callers bind": it binds on `READY`
  callers, but only on versions created by that import and only inside the import
  transaction, after all of them exist, with every other bind-time check and the cycle
  check run over the **whole DB graph**. A `READY` version therefore gains bindings only at
  the moment it is created. See [`../../constraints.md`](../../constraints.md).
- **New versions inherit agent-tool config verbatim.** When training (or any other fork
  path) materialises a new version from a source version, `agent_tool_enabled` and every
  `agent_version_subagent` row are copied unchanged. Seeded versions are the exception
  (config comes from the file).
- **Tool backend test-connection precedes save** — `POST /mcp/test` pings a backend before
  saving via `POST /mcp`. Seed upserts skip test-connection (the target may not be
  reachable at boot).
- **Managed backends are locked.** A backend with `managed_by_seed` set cannot be edited or
  deleted through the backoffice or REST (`409`). The seed import owns it: on every boot it
  upserts the backend by `name` from the interpolated file (the file wins, including taking
  over a backoffice-created backend of the same name), sets the flag, and clears it on
  managed backends the seed set no longer declares (or on all of them when seeding is off),
  leaving the rows in place. A backend's `kind` is fixed once created; a seed declaring a
  different kind is an error.
- **Static IDs are portable identity.** `agents.static_id` and `agent_versions.static_id` are
  optional, globally unique per table, lower-case, never UUID-shaped, and write-once: once
  set they never change or clear while the row exists. They are set only by export or by
  the seed import. A route value that looks like a UUID (case-insensitively) is resolved by
  id, anything else by static ID; stored keys stay UUIDs.
- **Seed import: create if missing, otherwise leave alone.** In one transaction, after
  migrations and before serving: backends are upserted (above); a missing agent is created
  with its architecture, endpoint wiring and `finalized_at`; an existing agent is left
  alone, and its architecture must equal the file's (canonical JSON) or the import fails; a
  missing version is created `READY` at the agent's tip (the root, carrying the agent's
  Inputs config, if the agent has none); an existing version is never written — a
  different agent is an error, a content-hash mismatch is a warning. New content needs a
  new static ID. Deploy markers apply the same rules as the deploy route (singleton,
  `READY` precondition, all endpoints mapped): `if_none` deploys only when the agent has
  nothing deployed; `force` deploys on every boot. Any error aborts the transaction and
  fails boot. Nothing is ever deleted by the import.
- **Export only attaches static IDs to immutable content.** A version is exportable only
  while `READY`, `TRAINING` or `DEPLOYED`. The export includes the whole closure (version,
  agent, transitive sub-agent versions and their agents, referenced backends), keeps
  existing static IDs (a conflicting supplied one is `409`), and persists generated ones in
  the same transaction, so re-exporting yields the same file. Backend header values are
  replaced by `${VAR}` placeholders.
- **`http_endpoint` and `mcp_client` are the only backend kinds.** `http_endpoint` bridges a
  REST endpoint as MCP; `mcp_client` proxies to an existing MCP server. Any other integration
  pattern needs a new kind + new migration. The agent tool is **not** a `tool_backends`
  kind — it is a built-in tool dispatched in-process against another agent version.

<!-- migrated from _migration-quarantine/ARCHITECTURE.md § MCP wiring, DESIGN.md § Versioning, PRODUCTION.md § 2.1 Mark a version as deployed on 2026-09-28. Updated 2026-09-28 for OpenBBC PR #50 (mig 025 status state machine). Updated 2026-09-30 for multiagent-tools — agent-tool invariants. Updated 2026-10-05 for sync-multiagent-feature — version/agent delete with 409 guards, config writable only on INITIALIZING/DRAFT, DAG acyclic by construction. Updated 2026-11-24 for declarative-seed — version immutability outside INITIALIZING/DRAFT at the repository layer, append-at-tip forks serialized on the agent row, READY entry via seed import, seed-import binding exception, managed backends, static IDs, seed import/export rules, finalized architecture never rewritten. -->

## Published surface

- **REST:**
  - `POST /agents/{agent_id}/deploy` / `POST /agents/{agent_id}/undeploy` — `agent_id` (and
    deploy's `version_id` body field) accept a static ID wherever they accept a UUID.
  - `POST /agent_versions/{version_id}/export` — returns an `openbbc-seed-v1` YAML attachment
    plus an `X-Seed-Env-Vars` header listing the `${VAR}` placeholder names. Optional JSON
    body: static IDs to assign to rows in the closure and the requested version's `deploy`
    marker (default `if_none`). `409` unless the version is `READY`/`TRAINING`/`DEPLOYED`,
    or on a static ID conflict; `404` for an unknown version. `version_id` may be a static ID.
  - Agent and version JSON objects carry an additive `static_id` field; `parent_version_id`
    means "previous in history".
  - `GET/POST /mcp*` (`/mcp/new`, `/mcp/test`, `/mcp/{id}`, `/mcp/{id}/edit`, `/mcp/{id}/delete`)
    — edit and delete return `409` on a managed backend.
  - `GET/POST /agents/{id}/configure/architecture/endpoints[/bulk|/{endpointID}/backend]`
  - `GET /agent_versions/{id}/configure/mcp`; writes
    `POST /agent_versions/{id}/architecture/mcp/{backendID}/toggle` and bulk
    `POST …/architecture/mcp/notes` — `409` outside `INITIALIZING`/`DRAFT`
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
- **CLI / boot input:** `SEED_PATH` (a seed file or a directory of them) is imported at
  `open-bbcd` boot; `open-bbcd seed validate [--no-env] <path>` (structural validation and
  interpolation without a DB; `--no-env` lists the required env var names instead) and
  `open-bbcd seed apply <path>` (full import against `DATABASE_URL`).
- **Consumed contract:** `.flow-map/` zip from `discovery` (schema in
  `bbc-discovery/flow-map-compiler/references/output-schemas.md`); `openbbc-seed-v1` seed
  files (versioned by their `format` key; unknown keys are errors).
- **Emitted contract:** aikdm `bundle.yaml` schema (`aikdm/schemas/prompt-v1.yaml`);
  `openbbc-seed-v1` seed files from export.

<!-- migrated from _migration-quarantine/ARCHITECTURE.md § Backoffice UI, § REST API, § MCP wiring, DESIGN.md § Phase I on 2026-09-28. Updated 2026-10-05 for sync-multiagent-feature — MCP and Agents-tab routes, version/agent delete. Updated 2026-11-24 for declarative-seed — export route, static IDs on deploy/undeploy, MCP writes and managed-backend 409s, seed CLI and file contract. -->
