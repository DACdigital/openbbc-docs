# artifacts

## Purpose

Own the artifact lifecycle: identity, storage-backend selection, put/get/delete against the
configured artifact store, and normalisation of MCP tool-result file payloads
(`ImageContent`, `EmbeddedResource`) into a uniform `artifact_ref` shape. Sits sideways to
`feedback-datasets` and `deployed-runtime` — both contexts consume `artifact_ref` content
blocks on their message aggregates but do not own the artifact concept themselves.
Downstream: the deployer-provided external object store, reached through the artifact-store
adapter (see [`../../c4/integrations.md`](../../c4/integrations.md)).

## Aggregates & entities

- **Artifact store** (aggregate root) — `artifact_stores` row. Fields: `id`, `name`,
  `kind ∈ {s3_compatible, …}`, `config` JSONB (endpoint, bucket, region, credentials — treat
  as secrets), `is_default BOOLEAN` (partial unique index — exactly one true at a time),
  `created_at`, `updated_at`. Shape parallels `tool_backends`.
- **Artifact reference** (value object embedded in `chat_messages.content` /
  `deployed_messages.content` JSONB) — an `artifact_ref` content block: `{type:
  "artifact_ref", store_id, uri, mime, size_bytes, sha256}`. Not a standalone entity — has
  no independent identity outside the message it lives on; identity of the underlying blob
  is `(store_id, uri)`. Multiple messages may reference the same blob (e.g. tool result
  re-referenced in an assistant follow-up).

There is **no `artifacts` table**. The framework does not maintain a separate index of
uploaded blobs — the source of truth for "which artifacts exist" is the union of
`artifact_ref` blocks embedded in message content across all sessions. This keeps the
context stateless beyond the store-configuration row.

## Domain events

N/A because OpenBBC does not emit domain events on any transport. State transitions here are
Postgres-only (`INSERT` / `UPDATE` / `DELETE` on `artifact_stores`) plus HTTP calls to the
external object store via the adapter (which are not domain events — they are outbound
integration calls). Downstream consumers observe state changes by reading the REST /
adapter surface, not a bus.

## Invariants

- **Exactly one `is_default = true` artifact store at any time** — partial unique index on
  `artifact_stores (is_default) WHERE is_default = true`. Fresh deployments start with zero;
  admin picks one before the first artifact-carrying turn is accepted.
- **Bytes never live in Postgres.** `chat_messages.content` and `deployed_messages.content`
  hold only refs (`{store_id, uri, mime, size_bytes, sha256}`); the artifact-store adapter
  is the only surface that touches blob bytes.
- **Refs are immutable once written to a message.** A blob referenced from a locked chat
  session (dataset-closed) must remain resolvable — deletion of the underlying blob or its
  hosting `artifact_stores` row while any locked session references it is rejected at the
  repo layer (dataset-close-draft snapshots the ref, not the bytes; replay must resolve).
- **`ARTIFACT_MAX_UPLOAD_MB` gates ingest, not retrieval.** Uploads exceeding the env-var
  cap are rejected at the upload boundary; a ref already stored from a lower prior cap
  remains readable.
- **MCP tool-result normalisation.** `ImageContent` (`data` base64 + `mimeType`) and
  `EmbeddedResource` (uri or blob) are converted to `artifact_ref` blocks on ingest — bytes
  land in the default artifact store; the block carries the resulting `(store_id, uri)`
  pair.
- **Session-scoped read authorisation.** An `artifact_ref` embedded in a session's message
  is readable only in the context of that session's access policy (see
  [`../access-model.md`](../access-model.md)) — the ref alone is not a bearer capability.

## Published surface

- **REST (BO / admin):**
  - `GET/POST /artifact-stores`, `PATCH /artifact-stores/{id}`,
    `DELETE /artifact-stores/{id}` — CRUD.
  - `POST /artifact-stores/{id}/test-connection` — round-trip a small write+read+delete
    against the configured backend.
  - `POST /artifact-stores/{id}/set-default` — flip the `is_default` flag atomically.
- **REST (session-scoped):**
  - `POST /chat-sessions/{id}/artifacts` and
    `POST /deployed/{agent_id}/sessions/{sid}/artifacts?user_id=X` (multipart) — inbound
    upload from a user turn; returns `{store_id, uri, mime, size_bytes, sha256}` for the
    client to embed in the outgoing turn body.
  - `GET /artifacts/{store_id}/{uri}?session_id=…&user_id=…` — proxied read (or
    presigned-URL redirect, adapter-dependent). Enforces session-scope through the same
    trust boundary as messages: 404 on mismatch.
- **Emitted contract for `feedback-datasets` and `deployed-runtime`:** the `artifact_ref`
  content-block shape embedded in their `.content` JSONB. Both contexts round-trip these
  blocks verbatim; only this context calls the store adapter.
- **Consumed contract from external object store:** artifact-store adapter interface (see
  [`../../c4/integrations.md § Contracts`](../../c4/integrations.md#contracts)) — put, get
  or sign, delete, stat. Adapter-config schemas per kind live in
  [`../../c4/integrations.md`](../../c4/integrations.md).
