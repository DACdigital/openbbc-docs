# deployed-runtime

## Purpose

Serve the production agent to end users over AG-UI. One DEPLOYED agent version per chain;
sessions are scoped by an opaque, gateway-verified `user_id`. Downstream of `agent-lifecycle`
(consumes the DEPLOYED version); upstream of the external client backend (over MCP).

## Aggregates & entities

- **Deployed session** (aggregate root) — `deployed_sessions` row. Scoped by `(agent_id,
  user_id)`. Fields: `id`, `agent_id`, `user_id`, `title`, timestamps. **Does not** carry
  `header_overrides` today.
- **Deployed message** (entity within Deployed session) — `deployed_messages` row. Turns;
  streamed over AG-UI Server-Sent Events. `content` is a typed content-block list (JSONB)
  matching `chat_messages.content`: `text` blocks + `artifact_ref` blocks. Refs point at
  the deployer's configured artifact store — see [`artifacts`](artifacts.md).

<!-- migrated from _migration-quarantine/ARCHITECTURE.md § Agent Runtime, PRODUCTION.md § 2 Integrating your frontend on 2026-09-28 -->

## Domain events

N/A because OpenBBC does not emit domain events on any transport. State transitions in
this context are Postgres-only — `INSERT` against `deployed_sessions` and
`deployed_messages`, `DELETE` cascades on session teardown — and downstream consumers do
not exist beyond the end user (who reads via the AG-UI SSE stream, a transport-layer
protocol, not a domain-event bus). The AG-UI wire chunks (`RUN_STARTED`,
`TEXT_MESSAGE_START` / `CONTENT` / `END`, `TOOL_CALL_START` / `ARGS` / `END`, `TURN_END`,
`ERROR`) are message-framing over the response, not events another bounded context
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
- **No per-session header overrides on outbound MCP calls today** — the deployed runtime
  uses whatever server-to-server credentials the MCP backend is registered with.
- **Artifact refs on `deployed_messages` are session-scoped by the same `user_id` trust
  boundary as the messages themselves.** Reads of `GET /artifacts/{store_id}/{uri}` require
  a session that resolves to the caller's `user_id` (via the same trusted-value model that
  scopes messages); mismatch returns `404` — see [`../access-model.md`](../access-model.md)
  and [`artifacts.md`](artifacts.md).
- **AG-UI event-type extension for outbound artifact streaming.** Existing wire events
  (`RUN_STARTED`, `TEXT_MESSAGE_*`, `TOOL_CALL_*`, `TURN_END`, `ERROR`) do not carry
  binary/file payloads; assistant-emitted artifacts stream as a new `ARTIFACT_REF` event
  type carrying the same content-block shape (`store_id`, `uri`, `mime`, `size_bytes`,
  `sha256`) rather than transporting bytes over SSE. Upstream AG-UI spec is versioned
  separately — see [`../../c4/integrations.md § Contracts`](../../c4/integrations.md#contracts).

<!-- migrated from _migration-quarantine/PRODUCTION.md § 2 Integrating your frontend, § 4 Headers, § 5 Auth model, ARCHITECTURE.md § Agent Runtime on 2026-09-28 -->

## Published surface

- **REST (AG-UI-facing):**
  - `POST /deployed/{agent_id}/sessions`
  - `GET /deployed/{agent_id}/sessions?user_id=X`
  - `GET /deployed/{agent_id}/sessions/{id}?user_id=X`
  - `PATCH /deployed/{agent_id}/sessions/{id}/title`
  - `DELETE /deployed/{agent_id}/sessions/{id}?user_id=X`
  - `POST /deployed/{agent_id}/sessions/{session_id}/turn?user_id=X` (streams AG-UI SSE)
  - `POST /deployed/{agent_id}/sessions/{session_id}/artifacts?user_id=X` (multipart
    upload; returns `{store_id, uri, mime, size_bytes, sha256}` for the caller to embed on
    the outgoing turn)
- **Emitted contract:** AG-UI event stream (`RUN_STARTED`, `TEXT_MESSAGE_*`, `TOOL_CALL_*`,
  `ARTIFACT_REF`, `TURN_END`, `ERROR`) — client integrates via any AG-UI SDK that
  understands the `ARTIFACT_REF` event-type extension; older SDKs see it as an unknown
  event and can fall back to text-only rendering.
- **Downstream contract to external backend:** MCP tool calls over SSE / Streamable HTTP,
  dispatched by `internal/llm/tools.Builder` via `toolBackendStoreAdapter`
  (`internal/handler/api.go:355`).

<!-- migrated from _migration-quarantine/PRODUCTION.md § 2 Integrating your frontend, § 3 MCP layer, ARCHITECTURE.md § Agent Runtime, § Protocols on 2026-09-28 -->
