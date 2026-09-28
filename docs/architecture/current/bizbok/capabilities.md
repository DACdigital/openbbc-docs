# Capabilities

## L1 capabilities

1. **Discovery** — scan a client frontend repo and propose an MCP tool surface for the
   backend the frontend talks to.
2. **Agent lifecycle management** — create, generate, wire, version, and deploy AI agents.
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

<!-- migrated from _migration-quarantine/DESIGN.md § Flow (phases 0–V), ARCHITECTURE.md § Components on 2026-09-28. Updated 2026-09-28 for OpenBBC PR #50 — added "Batch drainer operations" L1 to cover the alpha drainer and Helm CronJobs. -->

## L2 capabilities

**Under Discovery:**
- `flow-map-compilation` — scan a frontend repo, produce `.flow-map/` at schema v2:
  `AGENTS.md`, `APP.md`, `glossary.md`, `skills/<id>.md`, `flows/<id>.md`, plus one
  `endpoints/<id>.md` per discovered backend call (all `proposed: true`).

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

**Under Feedback & dataset curation:**
- `backoffice-chat` — test any version in the BO chat (`/agent_versions/{id}/chat`), capture
  feedback per assistant message.
- `dataset-authoring` — DRAFT/CLOSED lifecycle; session assignment; cumulative seeding of the
  next DRAFT from CLOSED.
- `judge-criteria-capture` — per-message JSONB `judge_criteria` (migration 021); required for
  dataset close.

**Under Evaluation:**
- `eval-run` — kick off an eval (BO Evaluate button → PENDING row → `scripts/run_eval.sh`
  one-shot, or the k8s eval `CronJob` drainer).
- `eval-scoring` — per-session simulator/target/tool_mock/judge pipeline; global pass-rate
  scoring.
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

<!-- migrated from _migration-quarantine/ARCHITECTURE.md § Backoffice UI, § REST API, § MCP wiring, § Feedback + datasets, § Evals, § Training sessions, § Chat header overrides, DESIGN.md § Flow, § Resources on 2026-09-28. Updated 2026-09-28 for OpenBBC PR #50 (mig 025 PENDING state, 026 discovery_zip inline, alpha drainer + eval/training drainers as k8s CronJobs, mcp-over-rest-bridge L2 capability, flow-map schema v2). -->

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

<!-- migrated from _migration-quarantine/ARCHITECTURE.md § Components, § MCP wiring, DESIGN.md § Flow on 2026-09-28. Updated 2026-09-28 for OpenBBC PR #50 (new drainers + mcp-over-rest-bridge rows). -->
