---
id: artifacts
level: 2
parent: open-bbcd
title: artifacts
---

# artifacts

## Purpose

Artifact-store registry hydration (from env vars at boot), adapter dispatch,
MCP tool-result normalisation, and server-side MIME resolution. Handles both framework legs
of the artifact model: staged user upload and MCP tool output (inline media unpacked from
`ImageContent` / `EmbeddedResource` into `artifact_ref` blocks). A `{store_id, uri}` the LLM
writes into tool input is opaque JSON passed to the tool unchanged — never resolved,
rendered or authorised by the framework. (No assistant-emission leg: the LLM does not
generate binary content itself; only tools return artifacts to the assistant.) Writes route to the env-nominated
`ARTIFACT_STORE_DEFAULT`; reads route via the `store_id` embedded in each `artifact_ref`
(so old refs keep working after the default flips across redeploys).

**No Postgres tables owned by this L2.** The store registry lives in process memory
(hydrated once at boot from `ARTIFACT_STORE_<ID>_*` env vars); `artifact_ref` blocks live
embedded in `chat_messages.content` / `deployed_messages.content` JSONB, and session
artifacts (identity, pending → consumed lifecycle, read allow-list) live in
`chat_session_artifacts` / `deployed_session_artifacts` — owned by
[`feedback-datasets`](../feedback-datasets/README.md) and
[`deployed-runtime`](../deployed-runtime/README.md) respectively. This L2 is the *only*
code path that touches blob bytes on the deployer's Object store.

Publishes no routes of its own. The session-scoped artifact routes — upload, retrieval and
pending-artifact list/remove, nested under `/agent_versions/{v}/chat/{s}/…` (BO) and
`/deployed/{agent_id}/sessions/{sid}/…` (deployed) — are published by
[`feedback-datasets`](../feedback-datasets/README.md) and
[`deployed-runtime`](../deployed-runtime/README.md) and call into this L2 for store writes,
reads (proxied bytes or 302 to signed URL depending on adapter) and MIME resolution.
Session-scope is enforced by the owning context's session-artifact row: 404 on mismatch —
the ref alone is not a bearer capability.

Adapter contract (per kind, uniform framework-side surface):
`put(bytes, mime) → {uri, size_bytes, sha256}`; `get(uri) → bytes` or
`sign(uri, ttl, {content_type, content_disposition}) → https_url` (response overrides; kinds
that cannot set them ignore them); `stat(uri)` (also drives upload dedup and the
missing-blob render fallback); `delete(uri)`; `probe()` (boot-time self-check per registered
store, including that a missing key is reported as not-found). First shipped kind: `s3_compatible`. Uploads bounded by `ARTIFACT_MAX_UPLOAD_MB`.

Implements the [`artifacts`](../../../ddd/contexts/artifacts.md) DDD context.

## Scope (in / out)
