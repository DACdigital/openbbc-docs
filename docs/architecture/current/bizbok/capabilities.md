# Capabilities

## L1 capabilities

1. **Discovery** — turn a client frontend repo into a complete business-and-technical
   understanding of the app: user journeys (`flows/<id>.md` — intent, sequencing,
   preconditions, invariants, failure modes), business-domain specialties (`skills/<id>.md`
   — the vocabulary and semantics a runtime agent needs to reason about the domain),
   app-wide invariants + conventions + boundaries (`APP.md`), a domain glossary
   (`glossary.md`), and the proposed backend surface each domain uses (`endpoints/<id>.md` —
   HTTP method / path / shapes plus proposed MCP tool names). This is the input material for
   **both** prompt generation (skills + flows + glossary become the runtime agent's context)
   **and** tool wiring (endpoints become `tool_backends` bindings). Not just tooling.
2. **Agent lifecycle management** — create, generate, wire, version, and deploy AI agents,
   and compose them into multi-agent topologies through the per-version agent tool.
3. **Feedback & dataset curation** — capture per-message feedback and roll it into versioned
   evaluation datasets.
4. **Evaluation** — score an agent version against a closed dataset version.
5. **Automated training** — improve an agent version via bounded hill-climb loops using eval
   as reward.
6. **Deployed agent runtime** — expose a deployed agent version to client frontends over
   AG-UI.
7. **Batch drainer operations** — asynchronously drain `PENDING` queues (alphas, evals,
   trainings) via flock-protected scripts run one-shot from operators or on `CronJob`
   cadence in Kubernetes.
8. **Artifact management** — accept, store, retrieve, and delete user-visible file objects
   (artifacts) exchanged inside chat and deployed sessions. Cross-cutting substrate: bytes
   live in a deployer-configured artifact store behind a pluggable adapter; `open-bbcd`
   holds only refs and per-session artifact metadata. Consumed by `Feedback & dataset
   curation` (via `chat-artifacts`) and `Deployed agent runtime` (via
   `deployed-runtime-artifacts`). Store registry is loaded
   **at boot from env vars** — there is no REST / BO surface for adding, removing, or
   reconfiguring stores at runtime; credentials handling matches the LLM-API-key pattern,
   not the `tool_backends` pattern.

<!-- migrated from _migration-quarantine/DESIGN.md § Flow (phases 0–V), ARCHITECTURE.md § Components on 2026-09-28. Updated 2026-09-28 for OpenBBC PR #50 — added "Batch drainer operations" L1 to cover the alpha drainer and Helm CronJobs. Updated 2026-09-28 for artifact-support — added "Artifact management" L1. Updated 2026-10-01 for sync-deployed-runtime-artifacts — per-session artifact metadata in Postgres. -->

## L2 capabilities

**Under Discovery:**
- `flow-map-compilation` — scan a frontend repo and produce `.flow-map/` at schema v2:
  - `AGENTS.md` — entry point + retrieval indices for the wiki.
  - `APP.md` — app-wide invariants, conventions, boundaries.
  - `glossary.md` — domain-vocabulary pivot table (skill ↔ user phrases ↔ endpoints ↔
    flows).
  - `skills/<id>.md` — one per business-domain specialty; primary read for the runtime
    agent; aggregates endpoints sharing a domain vocabulary and invariants.
  - `flows/<id>.md` — one playbook per user journey (intent, sequencing, preconditions,
    invariants, failure modes — no HTTP detail).
  - `endpoints/<id>.md` — one per discovered backend call; HTTP method / path / params /
    response shape / auth source / proposed MCP tool name; every entry `proposed: true`.

  Downstream:
  - The prompt-generation path consumes `AGENTS.md` + `APP.md` + `glossary.md` + `skills/`
    + `flows/` to produce `main_prompt`, `skills[]`, and `external_actions[]` in the aikdm
    bundle.
  - The tool-wiring path consumes `endpoints/` to populate agent-level
    `endpoint→backend` bindings.

**Under Agent lifecycle management:**
- `agent-bundle-generation` — run the two-agent generator+critic loop over a `.flow-map/` +
  domain-expert inputs to emit an agent prompt bundle (aikdm `generate-agent`).
- `agent-configuration` — wizard + configurator: agent-level structural config (flows,
  endpoints, endpoint→backend wiring, discovery zip inline via migration 026) and
  version-level prompt config.
- `mcp-backend-management` — CRUD over `tool_backends` (two kinds: `http_endpoint` = OpenBBC's
  built-in MCP-over-REST bridge, `mcp_client` = proxy to an existing MCP server) and
  version-scoped MCP attachments; test-connection before saving.
- `mcp-over-rest-bridge` — runtime capability provided by `open-bbcd`: exposes a plain REST
  endpoint as an MCP tool to the agent, letting operators skip building an MCP server for
  clients whose backends are REST-only.
- `version-management` — linked-list versioning (`agents.parent_version_id`); prompts
  editable per version; status state machine `INITIALIZING → PENDING → READY →
  TRAINING → READY → DEPLOYED` (migration 025).
- `agent-deployment` — mark a version DEPLOYED (DB-enforced singleton per agent chain).
- `agent-tool-configuration` — per-version checkbox enabling the agent tool
  (`agent_versions.agent_tool_enabled`) plus the allow-list of sub-agent bindings
  (`agent_version_subagent`: pinned target version, tool-facing name, prompt note).
  Bind-time checks: target is `READY` or `DEPLOYED`, and the binding keeps the topology a
  DAG. Lets admins build planner → coordinator → worker topologies from ordinary agents.

**Under Feedback & dataset curation:**
- `backoffice-chat` — test any version in the BO chat (`/agent_versions/{id}/chat`), capture
  feedback per assistant message.
- `dataset-authoring` — DRAFT/CLOSED lifecycle; session assignment; cumulative seeding of the
  next DRAFT from CLOSED.
- `judge-criteria-capture` — per-message JSONB `judge_criteria` (migration 021); required for
  dataset close.
- `chat-artifacts` — attach and receive artifacts inside a BO chat session. Two legs:
  admin uploads a file, which is staged as a pending artifact on the session (listable and
  removable) and claimed server-side into the next user turn — the turn body carries text
  only; MCP tool result carrying inline `ImageContent` / `EmbeddedResource` is unpacked into
  an `artifact_ref` content block on a `tool`-role message and recorded as a session
  artifact. A `{store_id, uri}` the LLM writes into `tool_input` is opaque — the framework
  never resolves it. There is **no** assistant-emission leg — the LLM does not itself
  generate binary content; only tools return artifacts back to the assistant. Artifact refs
  persist on `chat_messages.content` JSONB and the session's read scope in
  `chat_session_artifacts`; bytes live in the env-configured default artifact store (see
  `artifact-store-adapter`). Dataset close-draft captures refs verbatim (exported unchanged);
  eval replay of artifact-bearing sessions is text-only until an artifact-aware replay
  follow-up.

**Under Evaluation:**
- `eval-run` — kick off an eval (BO Evaluate button → PENDING row → `scripts/run_eval.sh`
  one-shot, or the k8s eval `CronJob` drainer).
- `eval-scoring` — per-session simulator/target/tool_mock/judge pipeline; global pass-rate
  scoring. For a version with the agent tool, the target step runs the real topology
  in-process from the transitive sub-agent bundles in `eval-input.yaml`
  (`mock_mcp_tools` applies to leaf MCP tools only; the agent tool itself is never
  mocked).
- `header-override-management-for-evals` — flat `header_overrides` map on the eval row
  (migration 023) proxied to real MCP calls.

**Under Automated training:**
- `training-session-lifecycle` — PENDING → IN_PROGRESS → DONE|FAILED; one active session per
  eval; DONE materialises a new agent version.
- `hill-climb-loop` — teacher/judge/eval loop with `patience`-based early stop and
  perfect-score shortcut.

**Under Deployed agent runtime:**
- `deployed-session-management` — create/list/rename/delete sessions per (agent, user).
- `ag-ui-turn-streaming` — POST /turn → SSE event stream.
- `mcp-tool-dispatch` — resolve endpoint→backend at runtime and dispatch tool calls to
  `tool_backends` (via the `http_endpoint` REST bridge OR the `mcp_client` MCP proxy).
- `deployed-runtime-artifacts` — same two-leg pattern as `chat-artifacts` on the production
  path (`deployed_sessions` + `deployed_messages` + `deployed_session_artifacts`). End user
  uploads artifacts, which are staged as pending artifacts on the session (listable and
  removable) and consumed by the next turn — the turn body carries text only; MCP tool
  results carrying inline `ImageContent` / `EmbeddedResource` are unpacked into
  `artifact_ref` content blocks and recorded as session artifacts. `{store_id, uri}`
  pointers in tool input are opaque to the framework. Tool-produced artifacts are surfaced
  to the frontend through the AG-UI wire (a `CUSTOM` `ARTIFACT_REF` event; see
  [`../ddd/contexts/deployed-runtime.md`](../ddd/contexts/deployed-runtime.md)). There is
  no assistant-emission leg. Access is session-scoped through the same trusted-`user_id`
  model as messages, plus a session-artifact row for every read.

- `sub-agent-dispatch` — runtime side of the agent tool, shared by BO chat and the deployed
  runtime (same `tools.Builder` path as `mcp-tool-dispatch`): create a child session, run
  the pinned target version's turn loop with a fresh context, return final text as the tool
  result (text only — no artifacts in either direction). Enforces `AGENT_TOOL_MAX_DEPTH` and
  `AGENT_TOOL_MAX_PARALLEL`; propagates `user_id` and (BO / eval) `header_overrides`; artifact
  scope is per session (the child's tool-result artifacts stay on the child session and are
  not visible to the root or the user); forwards sub-agent progress to the AG-UI stream as
  `STEP_STARTED` / `STEP_FINISHED` + child-tagged `TOOL_CALL_*` (sub-agent text tokens and
  child `ARTIFACT_REF`s are not streamed).

**Under Batch drainer operations:**
- `alpha-drainer` — `scripts/process_pending_alphas.sh` → `generate_alpha.sh` →
  `aikdm generate-agent` + `seed_bundle.py`. Enumerates PENDING agent versions via
  `GET /agent_versions.json?status=PENDING` and transitions each `PENDING → READY` on
  success. Needs `DATABASE_URL` because `seed_bundle.py` writes bundles directly to
  Postgres. Suggested cron `*/5 * * * *`; k8s CronJob `cronjob-alphas.yaml`.
- `eval-drainer` — `scripts/process_pending_evals.sh` → `run_eval.sh`. Suggested
  `*/10 * * * *`; k8s CronJob `cronjob-evals.yaml`.
- `training-drainer` — `scripts/process_pending_trainings.sh` → `train_from_session.sh --yes`.
  Suggested `*/15 * * * *`; k8s CronJob `cronjob-trainings.yaml`.

**Under Artifact management:**
- `artifact-store-adapter` — pluggable interface inside `open-bbcd` wrapping each artifact
  store behind a `put(bytes, mime) → uri`, `get(uri) → bytes` / `sign(uri, ttl,
  SignOptions{ContentType, ContentDisposition}) → signed_url` (response overrides; kinds that
  cannot set them ignore them), `delete(uri)`, `stat(uri)` (also drives upload dedup and the
  missing-blob render fallback) contract, plus a boot-time `probe()` that also fails boot if
  a missing key is not reported as not-found. First shipped kind: `s3_compatible` (covers AWS S3, MinIO, GCS-HMAC,
  R2, B2, any S3-API endpoint). **Store registry is loaded at boot from env vars** —
  `ARTIFACT_STORE_<ID>_KIND` plus kind-specific config vars (e.g. `_ENDPOINT`, `_BUCKET`,
  `_ACCESS_KEY`, `_SECRET_KEY` for `s3_compatible`); `ARTIFACT_STORE_DEFAULT=<ID>`
  nominates which store new writes go to. `<ID>` becomes the `store_id` on every
  `artifact_ref` block — must be stable across deploys or historical refs stop resolving.
  Reads route via ref's `store_id` (each blob knows its store); writes route to the
  default. Uploads are bounded by `ARTIFACT_MAX_UPLOAD_MB` (see
  [`../constraints.md`](../constraints.md)).

<!-- migrated from _migration-quarantine/ARCHITECTURE.md § Backoffice UI, § REST API, § MCP wiring, § Feedback + datasets, § Evals, § Training sessions, § Chat header overrides, DESIGN.md § Flow, § Resources on 2026-09-28. Updated 2026-09-28 for OpenBBC PR #50 (mig 025 PENDING state, 026 discovery_zip inline, alpha drainer + eval/training drainers as k8s CronJobs, mcp-over-rest-bridge L2 capability, flow-map schema v2). Updated 2026-09-28 for artifact-support — added chat-artifacts, deployed-runtime-artifacts, artifact-store-management, artifact-store-adapter. Updated 2026-09-30 for multiagent-tools — added agent-tool-configuration, sub-agent-dispatch; eval-scoring runs real topologies. Updated 2026-10-01 for sync-deployed-runtime-artifacts — staged uploads + two-leg artifacts in chat-artifacts / deployed-runtime-artifacts, text-only sub-agent-dispatch, adapter SignOptions + missing-key probe. -->

## Capability → context/container map

| L2 capability | DDD context | C4 container |
|---------------|-------------|--------------|
| `flow-map-compilation` | [discovery](../ddd/contexts/discovery.md) | [flow-map-compiler](../c4/containers.md#flow-map-compiler) |
| `agent-bundle-generation` | [agent-lifecycle](../ddd/contexts/agent-lifecycle.md) | [aikdm](../c4/containers.md#aikdm) |
| `agent-configuration` | [agent-lifecycle](../ddd/contexts/agent-lifecycle.md) | [open-bbcd](../c4/containers.md#open-bbcd) |
| `mcp-backend-management` | [agent-lifecycle](../ddd/contexts/agent-lifecycle.md) | [open-bbcd](../c4/containers.md#open-bbcd) |
| `version-management` | [agent-lifecycle](../ddd/contexts/agent-lifecycle.md) | [open-bbcd](../c4/containers.md#open-bbcd) |
| `agent-deployment` | [agent-lifecycle](../ddd/contexts/agent-lifecycle.md) | [open-bbcd](../c4/containers.md#open-bbcd) |
| `backoffice-chat` | [feedback-datasets](../ddd/contexts/feedback-datasets.md) | [open-bbcd](../c4/containers.md#open-bbcd) |
| `dataset-authoring` | [feedback-datasets](../ddd/contexts/feedback-datasets.md) | [open-bbcd](../c4/containers.md#open-bbcd) |
| `judge-criteria-capture` | [feedback-datasets](../ddd/contexts/feedback-datasets.md) | [open-bbcd](../c4/containers.md#open-bbcd) |
| `eval-run` | [evaluation](../ddd/contexts/evaluation.md) | [open-bbcd](../c4/containers.md#open-bbcd), [aikdm](../c4/containers.md#aikdm) |
| `eval-scoring` | [evaluation](../ddd/contexts/evaluation.md) | [aikdm](../c4/containers.md#aikdm) |
| `header-override-management-for-evals` | [evaluation](../ddd/contexts/evaluation.md) | [open-bbcd](../c4/containers.md#open-bbcd) |
| `training-session-lifecycle` | [training](../ddd/contexts/training.md) | [open-bbcd](../c4/containers.md#open-bbcd) |
| `hill-climb-loop` | [training](../ddd/contexts/training.md) | [aikdm](../c4/containers.md#aikdm) |
| `deployed-session-management` | [deployed-runtime](../ddd/contexts/deployed-runtime.md) | [open-bbcd](../c4/containers.md#open-bbcd) |
| `ag-ui-turn-streaming` | [deployed-runtime](../ddd/contexts/deployed-runtime.md) | [open-bbcd](../c4/containers.md#open-bbcd) |
| `mcp-tool-dispatch` | [deployed-runtime](../ddd/contexts/deployed-runtime.md) | [open-bbcd](../c4/containers.md#open-bbcd) |
| `mcp-over-rest-bridge` | [agent-lifecycle](../ddd/contexts/agent-lifecycle.md) | [open-bbcd](../c4/containers.md#open-bbcd) |
| `alpha-drainer` | [agent-lifecycle](../ddd/contexts/agent-lifecycle.md) | [aikdm-runner](../c4/containers.md#aikdm-runner) |
| `eval-drainer` | [evaluation](../ddd/contexts/evaluation.md) | [aikdm-runner](../c4/containers.md#aikdm-runner) |
| `training-drainer` | [training](../ddd/contexts/training.md) | [aikdm-runner](../c4/containers.md#aikdm-runner) |
| `chat-artifacts` | [feedback-datasets](../ddd/contexts/feedback-datasets.md), [artifacts](../ddd/contexts/artifacts.md) | [open-bbcd](../c4/containers.md#open-bbcd) |
| `deployed-runtime-artifacts` | [deployed-runtime](../ddd/contexts/deployed-runtime.md), [artifacts](../ddd/contexts/artifacts.md) | [open-bbcd](../c4/containers.md#open-bbcd) |
| `artifact-store-adapter` | [artifacts](../ddd/contexts/artifacts.md) | [open-bbcd](../c4/containers.md#open-bbcd) |
| `agent-tool-configuration` | [agent-lifecycle](../ddd/contexts/agent-lifecycle.md) | [open-bbcd](../c4/containers.md#open-bbcd) |
| `sub-agent-dispatch` | [deployed-runtime](../ddd/contexts/deployed-runtime.md), [feedback-datasets](../ddd/contexts/feedback-datasets.md) | [open-bbcd](../c4/containers.md#open-bbcd), [aikdm](../c4/containers.md#aikdm) |

<!-- migrated from _migration-quarantine/ARCHITECTURE.md § Components, § MCP wiring, DESIGN.md § Flow on 2026-09-28. Updated 2026-09-28 for OpenBBC PR #50 (new drainers + mcp-over-rest-bridge rows). Updated 2026-09-28 for artifact-support — four new rows across artifacts + feedback-datasets + deployed-runtime. Updated 2026-09-30 for multiagent-tools — agent-tool-configuration + sub-agent-dispatch rows. -->
