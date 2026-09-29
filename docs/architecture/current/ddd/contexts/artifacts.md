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

- **Artifact-store registry** (aggregate root — in-memory, not persisted) — populated at
  `open-bbcd` boot from env vars: for each `<ID>`, `ARTIFACT_STORE_<ID>_KIND ∈
  {s3_compatible, …}` plus kind-specific config (endpoint, bucket, region, credentials).
  `ARTIFACT_STORE_DEFAULT=<ID>` nominates the write target. The registry is redeploy-
  immutable; there is no runtime CRUD. `<ID>` is the `store_id` embedded in every
  `artifact_ref` block.
- **Artifact reference** (value object embedded in `chat_messages.content` /
  `deployed_messages.content` JSONB) — an `artifact_ref` content block: `{type:
  "artifact_ref", store_id, uri, mime, size_bytes, sha256, filename?}`. `filename` is
  optional display metadata used by the UI as the user-facing label; it is **never** used
  to construct storage URIs (the `(store_id, uri)` pair alone identifies the underlying
  blob). Not a standalone entity — has no independent identity outside the message it lives
  on. Multiple messages may reference the same blob (e.g. tool result re-referenced in a
  follow-up tool call via a `{store_id, uri}` inner-ref pointer inside `tool_input` /
  `tool_result` — see § Published surface).

There is **no `artifacts` table** and **no `artifact_stores` table**. Postgres holds only
`artifact_ref` blocks embedded in message content; the store registry lives in process
memory, hydrated from env at boot. The source of truth for "which artifacts exist" is the
union of `artifact_ref` blocks across all sessions; the source of truth for "which stores
exist" is the deploy-time env manifest. This keeps the context stateless in the DB.

## Domain events

N/A because OpenBBC does not emit domain events on any transport. The only state this
context owns is the in-memory store registry (loaded once at boot from env, immutable at
runtime) plus outbound HTTP calls to the external object store via the adapter — neither
is a domain event. Downstream consumers observe artifact activity by reading `artifact_ref`
blocks off message content or by calling the session-scoped read surface, not through a
bus.

## Invariants

- **Store registry is boot-time env-driven, not runtime-mutable.** Stores are declared via
  `ARTIFACT_STORE_<ID>_*` env vars; the registry is hydrated once at `open-bbcd` boot and
  is immutable thereafter. Adding, removing, or reconfiguring a store requires a redeploy.
  There is no REST / BO surface for store CRUD — this is intentional (parity with
  LLM-API-key handling, not with `tool_backends`).
- **Exactly one `ARTIFACT_STORE_DEFAULT` at boot.** Nominates which store new writes go
  to. If unset and any capability that emits `artifact_ref` blocks is exercised, the
  request is rejected with a clear error. Refs already stored under a previous default
  remain resolvable via their embedded `store_id` — flipping the default (across
  redeploys) does not break history.
- **Read/write asymmetry.** *Writes* always target the env-nominated
  `ARTIFACT_STORE_DEFAULT`. *Reads* route via the `store_id` embedded in the ref — every
  blob knows its store, so old refs keep working after the default flips. Consequence:
  multiple stores can coexist in the registry; only one drives fresh writes at a time.
- **Bytes never live in Postgres.** `chat_messages.content` and `deployed_messages.content`
  hold only refs (`{store_id, uri, mime, size_bytes, sha256}`); the artifact-store adapter
  is the only surface that touches blob bytes.
- **Refs are immutable once written to a message.** A blob referenced from a locked chat
  session (dataset-closed) must remain resolvable — deletion of the underlying blob while
  any locked session references it is rejected at the repo layer; removing the referenced
  store id from the deploy env would likewise strand refs and is the deployer's
  responsibility to avoid (dataset-close-draft snapshots the ref, not the bytes; replay
  must resolve).
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

- **Read-only admin visibility (optional, not required by this context):** an operator or
  admin can observe the loaded registry through `GET /health` or a diagnostic endpoint —
  the shape is not part of the published contract. **No** REST CRUD, test-connection, or
  set-default routes exist; the registry cannot be mutated at runtime.
- **REST (session-scoped):**
  - `POST /chat-sessions/{id}/artifacts` (BO, multipart) — inbound upload from a user turn;
    returns `{store_id, uri, mime, size_bytes, sha256, filename?}` for the client to embed
    in the outgoing turn body. Returns `409` when `chat_sessions.locked_at IS NOT NULL`
    (closed dataset versions may not accept new artifacts — see
    [`feedback-datasets.md § Invariants`](feedback-datasets.md#invariants)).
  - `POST /deployed/{agent_id}/sessions/{sid}/artifacts?user_id=X` (deployed, multipart) —
    same shape as above, additionally scoped by the gateway-verified `user_id`.
  - `GET /artifacts/{store_id}/{uri}?session_id=…` (BO) and
    `GET /artifacts/{store_id}/{uri}?session_id=…&user_id=…` (deployed) — retrieval. In BO
    the `session_id` in the query string is authoritative because the whole BO surface
    sits behind the operator gateway (see [`../../nfrs.md § Security`](../../nfrs.md#security));
    the deployed variant additionally requires `user_id` to enforce the same trust
    boundary as deployed messages. Session-scope is enforced by scanning the session's
    `chat_messages` / `deployed_messages` for an `artifact_ref` content block matching the
    requested `(store_id, uri)`; response is `200` proxied bytes or `302 Location <signed_url>`
    per the adapter's declared `PreferredDelivery()` (see
    [`../../c4/integrations.md § Contracts`](../../c4/integrations.md#contracts)). Returns
    `404` on session-scope mismatch or unknown `store_id`; `410` when the underlying blob
    has been externally removed (`adapter.stat().exists == false`).
- **Emitted contract for `feedback-datasets` and `deployed-runtime`:** the `artifact_ref`
  content-block shape embedded in their `.content` JSONB. Both contexts round-trip these
  blocks verbatim; only this context calls the store adapter.
- **Emitted contract for tool-call / tool-result payloads.** Inside the assistant's turn,
  `tool_input` and `tool_result` blocks may embed a `{store_id, uri}` inner-ref pointer
  when referring to an artifact that already has a full `artifact_ref` block on some
  message in the session. The inner ref does **not** repeat `{mime, size_bytes, sha256,
  filename}` — it's a pointer to the canonical ref. This is how the assistant references
  an existing artifact when calling a tool (leg 4), and how a tool result carrying an
  already-known artifact avoids duplication.
- **Consumed contract from external object store:** artifact-store adapter interface (see
  [`../../c4/integrations.md § Contracts`](../../c4/integrations.md#contracts)) — `PreferredDelivery`
  (adapter-declared at construction, `Bytes | SignedURL`), `put`, `get` (called when
  `Bytes`) or `sign` (called when `SignedURL`), `delete`, `stat` (returns `{exists, mime,
  size_bytes, sha256}`), `probe` (boot-time self-check). Adapter-config env-var schemas
  per kind live alongside the kind registration in
  [`../../c4/integrations.md`](../../c4/integrations.md).
- **Consumed contract from deploy env:** `ARTIFACT_STORE_<ID>_*` variables (one set per
  registered store) plus `ARTIFACT_STORE_DEFAULT=<ID>` (nominated write target) plus
  `ARTIFACT_MAX_UPLOAD_MB` (per-upload cap; required whenever artifact routes are wired)
  plus `ARTIFACT_SIGNED_URL_TTL_SECONDS` (optional, default `300`; passed as `ttl` to
  `adapter.Sign` when the adapter's `PreferredDelivery == SignedURL`). Parsed once at
  boot; failure to parse — or a duplicate `<ID>` across store groups — is a boot-time
  error.
