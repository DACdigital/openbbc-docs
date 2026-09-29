---
id: artifacts
level: 2
parent: open-bbcd
title: artifacts
---

# artifacts

## Purpose

Artifact-store registry hydration (from env vars at boot), adapter dispatch, and
MCP tool-result normalisation. Handles all three legs of the artifact model: user upload,
MCP tool output (unpacked from `ImageContent` / `EmbeddedResource` into `artifact_ref`
blocks), agent-to-tool argument. (No assistant-emission leg: the LLM does not generate
binary content itself; only tools return artifacts to the assistant.) Writes route to the env-nominated
`ARTIFACT_STORE_DEFAULT`; reads route via the `store_id` embedded in each `artifact_ref`
(so old refs keep working after the default flips across redeploys).

**No Postgres tables owned by this L2.** The store registry lives in process memory
(hydrated once at boot from `ARTIFACT_STORE_<ID>_*` env vars); `artifact_ref` blocks live
embedded in `chat_messages.content` / `deployed_messages.content` JSONB — owned by
[`feedback-datasets`](../feedback-datasets/README.md) and
[`deployed-runtime`](../deployed-runtime/README.md) respectively. This L2 is the *only*
code path that touches blob bytes on the deployer's Object store.

Publishes: `POST /chat-sessions/{id}/artifacts` (BO multipart upload),
`POST /deployed/{agent_id}/sessions/{sid}/artifacts?user_id=X` (deployed multipart upload),
`GET /artifacts/{store_id}/{uri}?session_id=…&user_id=…` (session-scoped read, proxied
bytes or 302 to signed URL depending on adapter). Session-scope enforced the same way as
messages: 404 on mismatch — the ref alone is not a bearer capability.

Adapter contract (per kind, uniform framework-side surface):
`put(bytes, mime) → {uri, size_bytes, sha256}`; `get(uri) → bytes` or `sign(uri, ttl) →
https_url`; `stat(uri)`; `delete(uri)`; `probe()` (boot-time self-check per registered
store). First shipped kind: `s3_compatible`. Uploads bounded by `ARTIFACT_MAX_UPLOAD_MB`.

Implements the [`artifacts`](../../../ddd/contexts/artifacts.md) DDD context.

## Scope (in / out)
