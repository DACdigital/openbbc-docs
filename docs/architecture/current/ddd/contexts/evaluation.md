# evaluation

## Purpose

Score a specific agent version against a specific CLOSED dataset version. One `evals` row
per run + one `eval_sessions` row per simulated session. Emits a numeric score in `[0, 1]`;
downstream `training` consumes that score to gate the Train button.

## Aggregates & entities

- **Eval** (aggregate root) — `evals` row. Fields: `agent_version_id`, `dataset_version_id`,
  `status ∈ {PENDING, IN_PROGRESS, DONE, FAILED}`, `score` (double precision),
  `total_criteria`, `passed_criteria`, `error_message`, `aikdm_meta` (JSONB), `mock_mcp_tools`
  (bool, default true, migration 023), `header_overrides` (flat JSONB map, migration 023),
  timestamps `created_at`, `started_at`, `completed_at`.
- **Eval session** (entity within Eval) — `eval_sessions` row (migration 022). Fields:
  `score`, `total_criteria`, `passed_criteria`, `transcript` (JSONB), `judgments` (JSONB per
  criterion). For multi-agent versions `transcript` nests each sub-agent run's transcript
  under the root's `agent` tool call.

<!-- migrated from _migration-quarantine/ARCHITECTURE.md § Evals, DESIGN.md § Phase III on 2026-09-28 -->

## Domain events

N/A because OpenBBC does not emit domain events on any transport. State transitions in
this context are Postgres-only — `INSERT` / `UPDATE` against `evals` and `eval_sessions`
— and downstream consumers poll the REST surface for status changes (the eval drainer
enumerates `GET /evals.json?status=PENDING`; the training context reads the eval's
`status`, `score`, `agent_version_id`, and `dataset_version_id` directly via REST when
its Train gate opens).

## Invariants

- **Immutable once DONE.** `evals` rows in DONE state do not mutate; new runs create new
  rows.
- **Score formula is fixed:** global pass-rate `sum(passed_criteria) / sum(total_criteria)`
  across all sessions (migration 022). Every criterion counts equally; longer sessions weigh
  proportionally more.
- **`mock_mcp_tools = true` (default) replays / synthesises tool responses deterministically**;
  `false` triggers real MCP calls with `header_overrides` proxied to backends.
- **State machine:** `PENDING → IN_PROGRESS → DONE|FAILED`; `FAILED` is reachable from any
  state via `POST /evals/{id}/fail`.
- **Header overrides are flat `map[string]string`**, not per-backend (contrast with
  `chat_sessions.backend_header_overrides`) — because an eval targets one agent version + one
  dataset and a single header set is enough.
- **Score always computed against a CLOSED dataset version** — DRAFT is not evaluable.
- **Evals run the real topology.** `GET /evals/{id}/export.yaml` includes, transitively, the
  bundle + tool wiring of every pinned sub-agent version reachable from the evaluated
  version; `aikdm evaluate` runs sub-agents in-process. `mock_mcp_tools` applies to leaf
  MCP tools at every depth; the agent tool itself is never mocked. `header_overrides`
  apply at every depth. Judges score the root transcript only.

<!-- migrated from _migration-quarantine/ARCHITECTURE.md § Evals, § Chat header overrides on 2026-09-28. Updated 2026-09-30 for multiagent-tools — real-topology eval invariant. -->

## Published surface

- **REST:**
  - `POST /evals` (create; typically from BO Evaluate button)
  - `GET /evals`, `GET /evals/{id}`, `GET /evals.json?status=PENDING` (used by cron)
  - `GET /evals/{id}/export.yaml` (script fetch; carries a `subagents` section with the
    transitive pinned bundles + `agent_tool` caps)
  - `POST /evals/{id}/start`, `/result`, `/fail`
- **UI:** `/evals`, `/agent_versions/{id}/evals`, eval detail page (renders per-session
  breakdown, exposes the Train button when gate passes).
- **Emitted contract for `training`:** DONE eval with `score`, `agent_version_id`,
  `dataset_version_id`.

<!-- migrated from _migration-quarantine/ARCHITECTURE.md § REST API, § Evals, DESIGN.md § Phase III, § Phase IV on 2026-09-28 -->
