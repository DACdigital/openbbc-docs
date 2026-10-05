# deployed-runtime

## Purpose

Serve the production agent to end users over AG-UI. One DEPLOYED agent version per chain;
sessions are scoped by an opaque, gateway-verified `user_id`. Downstream of `agent-lifecycle`
(consumes the DEPLOYED version); upstream of the external client backend (over MCP).

## Aggregates & entities

- **Deployed session** (aggregate root) — `deployed_sessions` row. Scoped by `(agent_id,
  user_id)`. Fields: `id`, `agent_id`, `user_id`, `title`, timestamps. **Does not** carry
  `header_overrides` today.
- **Child deployed session** (entity within the root Deployed session's tree) — a
  `deployed_sessions` row created by the agent tool for one sub-agent run. It holds
  `parent_session_id`, `parent_tool_call_id`, `depth`, the pinned target
  `agent_version_id`, and the **root's** `agent_id` and `user_id`. So a whole tree stays
  inside the root agent's partition: deleting the root's agent cascades the tree, and
  deleting a worker agent that a child pins is refused with `409`. The target version need
  not be the DEPLOYED one; it is whatever the binding pins. Root sessions keep
  `agent_version_id` NULL and resolve the DEPLOYED version on each turn.
- **Deployed message** (entity within Deployed session) — `deployed_messages` row. Turns;
  streamed over AG-UI Server-Sent Events. `content` is a typed content-block list (JSONB)
  matching `chat_messages.content`: `text` blocks + `artifact_ref` blocks. Refs point at
  the deployer's configured artifact store — see [`artifacts`](artifacts.md).
- **Deployed session artifact** (entity within Deployed session) — `deployed_session_artifacts`
  row (migration 028; `session_id → deployed_sessions(id) ON DELETE CASCADE`) recording that
  an artifact belongs to the session's read scope: `origin ∈ {upload, tool_result}`,
  `store_id`, `uri`, server-resolved `mime`, server-measured `size_bytes` / `sha256`,
  optional display `filename`, and `message_id`. `message_id NULL` = **pending** (origin
  `upload` only — staged, not yet consumed by a turn); otherwise the message carrying the ref
  (the claiming user message, or the tool-role message for `tool_result` rows). At most one
  pending row per blob per session (an identical upload while pending returns the existing
  row). Each child deployed session has its own rows — no inheritance with its root.

<!-- migrated from _migration-quarantine/ARCHITECTURE.md § Agent Runtime, PRODUCTION.md § 2 Integrating your frontend on 2026-09-28. Updated 2026-10-01 for sync-deployed-runtime-artifacts — Deployed session artifact entity. Updated 2026-10-05 for sync-multiagent-feature — child session carries the root's agent_id, root sessions keep agent_version_id NULL. -->

## Domain events

N/A because OpenBBC does not emit domain events on any transport. State transitions in
this context are Postgres-only — `INSERT` against `deployed_sessions` (root and child),
`deployed_messages` and `deployed_session_artifacts` (plus the pending → consumed claim
`UPDATE`), `DELETE` cascades on session teardown — and downstream consumers do
not exist beyond the end user (who reads via the AG-UI SSE stream, a transport-layer
protocol, not a domain-event bus). The AG-UI wire chunks (`RUN_STARTED`,
`TEXT_MESSAGE_START` / `CONTENT` / `END`, `TOOL_CALL_START` / `ARGS` / `END` / `RESULT`,
`STEP_STARTED` / `STEP_FINISHED`, `RUN_FINISHED`, `RUN_ERROR`) are message-framing over the response, not events another bounded context
subscribes to.

## Invariants

- **`user_id` is trusted, not verified.** `open-bbcd` treats `user_id` as authoritative on
  every request; safe only when the operator's gateway has verified the caller and rewritten
  the field. Documented in [`../access-model.md`](../access-model.md).
- **Cross-user 404 policy.** `GET /deployed/{agent_id}/sessions/{id}?user_id=X` returns
  `404` (not `403`) when the id belongs to a different user, preventing session-id
  enumeration.
- **`DELETE` cascades messages.** `DELETE /deployed/{agent_id}/sessions/{id}?user_id=X`
  removes all associated `deployed_messages`.
- **One agent deployed per chain** (migration 011) — enforced by `agent-lifecycle`; this
  context sees only the currently-DEPLOYED version.
- **Child sessions are invisible to every per-session route.**
  `GET /deployed/{agent_id}/sessions` lists root sessions only. Every per-session route
  (`GET` / `DELETE …/sessions/{id}`, `PATCH …/title`, `POST …/turn`, and all artifact and
  pending-artifact routes) resolves root sessions only. A child id returns `404`,
  indistinguishable from an unknown id, with no side effect. Children are read only through
  `GET …/sessions/{root_id}/children/{child_id}?user_id=X`, which requires the root
  preamble (`user_id` and agent match on a root) and that `child_id` descends from
  `root_id`. `DELETE` of a root cascades its whole tree, including messages and
  session-artifact rows.
- **Sub-agent progress, not sub-agent tokens, on the wire.** While a sub-agent runs, the
  root stream emits `STEP_STARTED` / `STEP_FINISHED` with
  `stepName = "<binding name>:<parent agent toolCallId>"`, plus the sub-agent's
  `TOOL_CALL_*` and `TOOL_CALL_RESULT` events with a `"<childSessionId>:"`-prefixed
  `toolCallId` and `rawEvent.childSessionId`. Sub-agent `TEXT_MESSAGE_*`, run and error
  events are not forwarded. Neither is `ARTIFACT_REF` for the sub-agent's tool results,
  because the child's refs are not readable by the root's user. The caller's
  `TOOL_CALL_RESULT` for the `agent` tool carries the final result (text only). The stream
  has one `RUN_STARTED` and one `RUN_FINISHED`.
- **Depth and parallelism caps** (`AGENT_TOOL_MAX_DEPTH`, `AGENT_TOOL_MAX_PARALLEL`) apply
  on the deployed path exactly as on BO chat — see [`../../constraints.md`](../../constraints.md).
- **No per-session header overrides on outbound MCP calls today** — the deployed runtime
  uses whatever server-to-server credentials the MCP backend is registered with.
- **Artifact refs on `deployed_messages` are session-scoped by the same `user_id` trust
  boundary as the messages themselves.** Reads of
  `GET /deployed/{agent_id}/sessions/{session_id}/artifacts/{store_id}/{uri...}?user_id=X`
  require the `turn` preamble — the agent is DEPLOYED, the session resolves for the caller's
  `user_id`, and the session belongs to `agent_id` — plus a row in
  `deployed_session_artifacts` for `(session_id, store_id, uri)` (any origin, pending or
  consumed). Any failure returns `404` — see [`../access-model.md`](../access-model.md) and
  [`artifacts.md`](artifacts.md). Child sessions are not addressable by any artifact route
  (`404`).
- **User artifacts are staged, never client-asserted.** Upload records a pending
  `deployed_session_artifacts` row; the next turn claims every pending row for the session in
  the same transaction that persists the user message and appends them as `artifact_ref`
  blocks after its text. The turn reads only `text` blocks from the request body —
  client-sent `artifact_ref` blocks are ignored. Tool-result refs are recorded as
  `origin = tool_result` rows in the same transaction as their tool-role message, so a ref is
  never in history without a row.
- **AG-UI `CUSTOM` event for outbound artifact streaming.** Existing wire events
  (`RUN_STARTED`, `TEXT_MESSAGE_*`, `TOOL_CALL_START` / `ARGS` / `END` / `RESULT`,
  `STEP_STARTED` / `STEP_FINISHED`, `RUN_FINISHED`, `RUN_ERROR`) do not carry
  binary/file payloads; tool-produced artifacts (from `ImageContent` / `EmbeddedResource`
  normalisation on the tool-result path) stream as an AG-UI `CUSTOM` event with
  `name: "ARTIFACT_REF"` and `value {toolCallId, storeId, uri, mime, sizeBytes, sha256,
  filename}` — `CUSTOM` because official AG-UI SDKs validate `type` against a closed set, so a
  new top-level type would fail validation. It is emitted only for tool-result refs, after the
  round's tool-role message and rows commit (so a client can resolve it immediately); never for
  user uploads, child-session tool results, or `{store_id, uri}` pointers the LLM writes into
  tool input (those are opaque to the framework). `TOOL_CALL_RESULT` carries the normalised
  remainder. With the artifact registry enabled, bytes are never transported over SSE; with it
  disabled, raw tool output (possibly carrying base64 media) is streamed as before. The `value`
  payload changes only additively. The LLM itself never emits artifacts — only tools do; there
  is no assistant-emission leg. Upstream AG-UI spec is versioned separately — see
  [`../../c4/integrations.md § Contracts`](../../c4/integrations.md#contracts).

<!-- migrated from _migration-quarantine/PRODUCTION.md § 2 Integrating your frontend, § 4 Headers, § 5 Auth model, ARCHITECTURE.md § Agent Runtime on 2026-09-28. Updated 2026-09-30 for multiagent-tools — child sessions, STEP_* progress, caps. Updated 2026-10-01 for sync-deployed-runtime-artifacts — nested retrieval route + row allow-list, staged uploads, ARTIFACT_REF as AG-UI CUSTOM event, child refs not forwarded. Updated 2026-10-05 for sync-multiagent-feature — children invisible to every per-session route, stepName/prefixed toolCallId, RUN_FINISHED/RUN_ERROR. -->

## Published surface

- **REST (AG-UI-facing):** Every per-session route below resolves root sessions only (a
  child id returns `404`).
  - `POST /deployed/{agent_id}/sessions`
  - `GET /deployed/{agent_id}/sessions?user_id=X`
  - `GET /deployed/{agent_id}/sessions/{id}?user_id=X`
  - `PATCH /deployed/{agent_id}/sessions/{id}/title`
  - `DELETE /deployed/{agent_id}/sessions/{id}?user_id=X`
  - `GET /deployed/{agent_id}/sessions/{root_id}/children/{child_id}?user_id=X` — read-only
    child transcript. It returns the same JSON as the session read, plus
    `parent_session_id`, `parent_tool_call_id`, `depth` and `agent_version_id`. It returns
    `404` unless `root_id` passes the deployed preamble as a root and `child_id` descends
    from it. Child `artifact_ref` blocks are returned as data; no artifact route serves them.
  - `POST /deployed/{agent_id}/sessions/{session_id}/turn?user_id=X` (streams AG-UI SSE)
    — reads only `text` blocks (client `artifact_ref` blocks are ignored) and claims the
    session's pending artifacts; no non-empty text and nothing pending → `400` (`empty_turn`);
    a pending artifact removed concurrently so nothing is left to claim → in-band `RUN_ERROR`
    with code `empty_turn`
  - Artifact routes (registered only when the artifact registry is enabled; all run the
    `turn` preamble and return `404` on any preamble failure):
    - `POST /deployed/{agent_id}/sessions/{session_id}/artifacts?user_id=X` (multipart,
      field `file`) — stages a pending artifact; returns the pending-artifact object
      `{id, store_id, uri, filename, mime, size_bytes, sha256, status}` (`status: "pending"`).
      `409` when the session already holds `ARTIFACT_MAX_PENDING` pending artifacts; `413`
      past `ARTIFACT_MAX_UPLOAD_MB`.
    - `GET /deployed/{agent_id}/sessions/{session_id}/artifacts/{store_id}/{uri...}?user_id=X`
      — retrieval, authorised by a `deployed_session_artifacts` row (see
      [`artifacts.md § Published surface`](artifacts.md#published-surface)).
    - `GET /deployed/{agent_id}/sessions/{session_id}/pending-artifacts?user_id=X` — list the
      session's pending artifacts (pending-artifact objects, in the order the next turn adds
      them).
    - `DELETE /deployed/{agent_id}/sessions/{session_id}/pending-artifacts/{id}?user_id=X` —
      remove one pending artifact before the next turn (`204`; `404` if no such row in this
      session; `409` if the row is already consumed). Deletes the row only, not the blob.
- **Emitted contract:** AG-UI event stream (`RUN_STARTED`, `TEXT_MESSAGE_*`,
  `TOOL_CALL_START` / `ARGS` / `END` / `RESULT`, `STEP_STARTED` / `STEP_FINISHED` for
  sub-agent progress (see [`../../c4/integrations.md § Contracts`](../../c4/integrations.md#contracts)
  for `stepName`, the `rawEvent` fields and the prefixed child `toolCallId`), `RUN_FINISHED`,
  `RUN_ERROR`, and
  `CUSTOM` with `name: "ARTIFACT_REF"` for tool-result artifact refs) — any AG-UI SDK accepts
  the `CUSTOM` event; clients that do not handle `ARTIFACT_REF` can ignore it and fall back
  to text-only rendering, and those that do resolve the ref against the nested retrieval
  route above.
- **Downstream contract to external backend:** MCP tool calls over SSE / Streamable HTTP,
  dispatched by `internal/llm/tools.Builder` via `toolBackendStoreAdapter`
  (`internal/handler/api.go:355`).

<!-- migrated from _migration-quarantine/PRODUCTION.md § 2 Integrating your frontend, § 3 MCP layer, ARCHITECTURE.md § Agent Runtime, § Protocols on 2026-09-28. Updated 2026-10-01 for sync-deployed-runtime-artifacts — empty-turn rule, nested artifact + pending-artifacts routes, ARTIFACT_REF as CUSTOM event. Updated 2026-10-05 for sync-multiagent-feature — child-transcript route, root-only per-session routes, event list. -->
