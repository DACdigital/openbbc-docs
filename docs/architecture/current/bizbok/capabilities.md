# Capabilities

## L1 capabilities

1. **Discovery** — extract business flows + backend capabilities from a client frontend repo.
2. **Agent lifecycle management** — create, version, wire, and deploy AI agents.
3. **Feedback & dataset curation** — capture per-message feedback and roll it into versioned
   evaluation datasets.
4. **Evaluation** — score an agent version against a closed dataset version.
5. **Automated training** — improve an agent version via bounded hill-climb loops using eval
   as reward.
6. **Deployed agent runtime** — expose a deployed agent version to client frontends over
   AG-UI.

<!-- migrated from _migration-quarantine/DESIGN.md § Flow (phases 0–V), ARCHITECTURE.md § Components on 2026-09-28 -->

## L2 capabilities

**Under Discovery:**
- `flow-map-compilation` — scan a frontend repo, produce `.flow-map/` (flows, capabilities,
  agents, plus zip).

**Under Agent lifecycle management:**
- `agent-bundle-generation` — run the two-agent generator+critic loop over a `.flow-map/` +
  domain-expert inputs to emit an agent prompt bundle (aikdm `generate-agent`).
- `agent-configuration` — wizard + configurator: agent-level structural config (flows,
  endpoints, endpoint→backend wiring) and version-level prompt config.
- `mcp-backend-management` — CRUD over `tool_backends` and version-scoped MCP attachments;
  test-connection before saving.
- `version-management` — linked-list versioning (`agents.parent_version_id`); prompts editable
  per version.
- `agent-deployment` — mark a version DEPLOYED (DB-enforced singleton per agent chain).

**Under Feedback & dataset curation:**
- `backoffice-chat` — test any version in the BO chat (`/agent_versions/{id}/chat`), capture
  feedback per assistant message.
- `dataset-authoring` — DRAFT/CLOSED lifecycle; session assignment; cumulative seeding of the
  next DRAFT from CLOSED.
- `judge-criteria-capture` — per-message JSONB `judge_criteria` (migration 021); required for
  dataset close.

**Under Evaluation:**
- `eval-run` — kick off an eval (BO Evaluate button → PENDING row → `scripts/run_eval.sh`).
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
- `mcp-tool-dispatch` — resolve endpoint→backend at runtime and dispatch tool calls to MCP
  backends.

<!-- migrated from _migration-quarantine/ARCHITECTURE.md § Backoffice UI, § REST API, § MCP wiring, § Feedback + datasets, § Evals, § Training sessions, § Chat header overrides, DESIGN.md § Flow, § Resources on 2026-09-28 -->

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

<!-- migrated from _migration-quarantine/ARCHITECTURE.md § Components, § MCP wiring, DESIGN.md § Flow on 2026-09-28 -->
