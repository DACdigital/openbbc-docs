---
id: deployed-runtime
level: 2
parent: open-bbcd
title: deployed-runtime
---

# deployed-runtime

## Purpose

Production runtime: deployed sessions + AG-UI turn streaming (Server-Sent Events) + the
`ARTIFACT_REF` stream event (carried as an AG-UI `CUSTOM` event, `name: "ARTIFACT_REF"`). End users hit this through the operator's auth
gateway which verifies the caller and rewrites `user_id` to the verified identity — the
daemon then trusts `user_id` as authoritative and returns `404` (not `403`) on
cross-user requests to block session-id enumeration.

Owns tables: `deployed_sessions` (scoped by `(agent_id, user_id)`, migration 011;
DB-enforced singleton "at most one DEPLOYED version per agent chain"), `deployed_messages`
(typed content-block JSONB — `text` + `artifact_ref` blocks post artifact-support;
streamed over AG-UI SSE), `deployed_session_artifacts` (migration 028;
`session_id → deployed_sessions(id) ON DELETE CASCADE`; one row per artifact in a session's
read scope, origin `upload` | `tool_result`, `message_id NULL` = pending).

Publishes: `POST /deployed/{agent_id}/sessions` (create),
`GET /deployed/{agent_id}/sessions?user_id=X` (list),
`GET /deployed/{agent_id}/sessions/{id}?user_id=X` (read, 404 on mismatch),
`PATCH .../title`, `DELETE .../sessions/{id}?user_id=X` (cascades messages),
`POST .../sessions/{session_id}/turn?user_id=X` (reads only `text` blocks — client
`artifact_ref` blocks are ignored — and claims the session's pending artifacts; no text and
nothing pending → `400` `empty_turn`, a concurrent removal leaving nothing to claim →
in-band `RUN_ERROR` `empty_turn`; streams AG-UI event stream: `RUN_STARTED`,
`TEXT_MESSAGE_*`, `TOOL_CALL_*`, `CUSTOM` `ARTIFACT_REF` for tool-result refs after the
round's tool message commits, `TURN_END`, `ERROR`), and the session-scoped artifact routes
(store and normalisation via the [`artifacts`](../artifacts/README.md) L2):
`POST .../sessions/{session_id}/artifacts?user_id=X` (upload — returns the pending-artifact
object `{id, store_id, uri, filename, mime, size_bytes, sha256, status}`; `409` at the
`ARTIFACT_MAX_PENDING` cap), `GET .../sessions/{session_id}/artifacts/{store_id}/{uri...}?user_id=X`
(retrieval, authorised by a `deployed_session_artifacts` row),
`GET .../sessions/{session_id}/pending-artifacts?user_id=X` (list pending) and
`DELETE .../sessions/{session_id}/pending-artifacts/{id}?user_id=X` (remove pending; `409` on
a consumed row). All artifact routes return `404` on any preamble failure.

**No per-session header overrides on outbound MCP calls today** — the deployed runtime uses
whatever server-to-server credentials the MCP backend is registered with. Tracked as an
open question in `assumptions.md § Open questions`.

Implements the [`deployed-runtime`](../../../ddd/contexts/deployed-runtime.md) DDD context.
Consumes the shared LLM tool machinery from [`tool-runtime`](../tool-runtime/README.md).

## Scope (in / out)
