---
id: deployed-runtime
level: 2
parent: open-bbcd
title: deployed-runtime
---

# deployed-runtime

## Purpose

Production runtime: deployed sessions + AG-UI turn streaming (Server-Sent Events) + the
`ARTIFACT_REF` event-type extension. End users hit this through the operator's auth
gateway which verifies the caller and rewrites `user_id` to the verified identity — the
daemon then trusts `user_id` as authoritative and returns `404` (not `403`) on
cross-user requests to block session-id enumeration.

Owns tables: `deployed_sessions` (scoped by `(agent_id, user_id)`, migration 011;
DB-enforced singleton "at most one DEPLOYED version per agent chain"), `deployed_messages`
(typed content-block JSONB — `text` + `artifact_ref` blocks post artifact-support;
streamed over AG-UI SSE).

Publishes: `POST /deployed/{agent_id}/sessions` (create),
`GET /deployed/{agent_id}/sessions?user_id=X` (list),
`GET /deployed/{agent_id}/sessions/{id}?user_id=X` (read, 404 on mismatch),
`PATCH .../title`, `DELETE .../sessions/{id}?user_id=X` (cascades messages),
`POST .../sessions/{session_id}/turn?user_id=X` (streams AG-UI event stream:
`RUN_STARTED`, `TEXT_MESSAGE_*`, `TOOL_CALL_*`, `ARTIFACT_REF`, `TURN_END`, `ERROR`), and
the session-scoped `POST .../sessions/{id}/artifacts?user_id=X` upload route (delegates to
the [`artifacts`](../artifacts/README.md) L2).

**No per-session header overrides on outbound MCP calls today** — the deployed runtime uses
whatever server-to-server credentials the MCP backend is registered with. Tracked as an
open question in `assumptions.md § Open questions`.

Implements the [`deployed-runtime`](../../../ddd/contexts/deployed-runtime.md) DDD context.
Consumes the shared LLM tool machinery from [`tool-runtime`](../tool-runtime/README.md).

## Scope (in / out)
