# artifacts

## Purpose

Own the artifact substrate: the env-hydrated store registry, storage-backend selection and
adapter dispatch (put/get/sign/stat/delete against the configured artifact store),
normalisation of MCP tool-result file payloads (`ImageContent`, `EmbeddedResource`) into a
uniform `artifact_ref` shape, and server-side MIME resolution. Owns no tables. Sits sideways
to `feedback-datasets` and `deployed-runtime` — both contexts consume `artifact_ref` content
blocks on their message aggregates and own the *Session artifact* identity, its
pending → consumed lifecycle and the per-session read allow-list in their own tables
(`chat_session_artifacts` / `deployed_session_artifacts`), and publish the session-scoped
artifact routes that call into this context.
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
  blob). The block itself is a value object; the identity of an artifact *within a session*
  is the owning context's *Session artifact* row (see below). Blobs are content-addressed
  (`uri = sha256/<hex>`), so several rows and messages — across sessions — may reference the
  same blob. `artifact_ref` blocks enter a message only from a claimed upload (user-role) or
  from MCP tool-result normalisation (tool-role); never from client input or LLM output.
- **Session artifact** (entity — owned by `feedback-datasets` as `chat_session_artifacts`
  and by `deployed-runtime` as `deployed_session_artifacts`, not by this context) — records
  that an artifact belongs to a session's read scope (origin `upload` or `tool_result`). A
  **pending artifact** is a session artifact of origin `upload` not yet consumed by a turn;
  the next turn on the session claims every pending artifact in the same transaction that
  persists the user message. See
  [`feedback-datasets.md § Aggregates & entities`](feedback-datasets.md#aggregates--entities)
  and [`deployed-runtime.md § Aggregates & entities`](deployed-runtime.md#aggregates--entities).

The **`artifacts` context owns no tables** — there is no `artifacts` table and no
`artifact_stores` table. The store registry lives in process memory, hydrated from env at
boot; the source of truth for "which stores exist" is the deploy-time env manifest. Which
artifacts a session may read is the owning context's session-artifact rows.

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
- **Bytes never live in Postgres when the artifact registry is enabled.**
  `chat_messages.content` and `deployed_messages.content` hold only refs
  (`{store_id, uri, mime, size_bytes, sha256}`); the artifact-store adapter is the only
  surface that touches blob bytes. With the registry disabled, normalisation does not run and
  raw tool output (possibly carrying base64 media) is streamed and persisted as before.
  Rendered media produced for one LLM request (inline bytes) is never persisted.
- **Refs are immutable once written to a message.** A blob referenced from a locked chat
  session (dataset-closed) must remain resolvable — deletion of the underlying blob while
  any locked session references it is rejected at the repo layer; removing the referenced
  store id from the deploy env would likewise strand refs and is the deployer's
  responsibility to avoid (dataset-close-draft snapshots the ref, not the bytes; the ref
  must stay resolvable).
- **`ARTIFACT_MAX_UPLOAD_MB` gates ingest, not retrieval.** Uploads exceeding the env-var
  cap are rejected at the upload boundary; a ref already stored from a lower prior cap
  remains readable.
- **MCP tool-result normalisation.** Inline media — `ImageContent` / audio (`data` base64 +
  `mimeType`) and `EmbeddedResource` with inline `blob` or `text` — is converted to
  `artifact_ref` blocks on ingest; bytes land in the default artifact store and the block
  carries the resulting `(store_id, uri)` pair. URI-only `EmbeddedResource`s are not
  normalised. Results flagged `isError` are normalised too (`is_error` is preserved). An item
  whose upload fails is replaced in the result by a text item
  `[artifact unavailable: <mime>, <size>]`; normalisation never falls back to the raw payload.
- **Server-side MIME resolution and measurement (both origins).** The stored `mime` is
  resolved from the bytes for uploads and tool results alike: for the native-render set
  (`image/png`, `image/jpeg`, `image/gif`, `image/webp`, `application/pdf`) the sniffed type
  wins, and images must also pass a structural decode check; a native-render label on bytes
  that do not match → `application/octet-stream`; otherwise the declared label is kept.
  `size_bytes` and `sha256` are always measured by the server. A native-render label is
  therefore never stored for bytes that do not look like it.
- **`{store_id, uri}` written by the LLM is opaque.** A pointer the LLM writes into
  `tool_use` input is passed to the tool unchanged — the framework never resolves, renders or
  authorises it, so prompt injection cannot pull another session's blob into a message.
- **Session-scoped read authorisation.** An artifact is readable only through its session's
  surface and only when the owning context holds a session-artifact row for
  `(session_id, store_id, uri)` — upload or tool_result origin, pending or consumed — under
  that session's access policy (see [`../access-model.md`](../access-model.md)). Message
  content is not consulted, and the ref alone is not a bearer capability.

## Published surface

- **Read-only admin visibility (optional, not required by this context):** an operator or
  admin can observe the loaded registry through `GET /health` or a diagnostic endpoint —
  the shape is not part of the published contract. **No** REST CRUD, test-connection, or
  set-default routes exist; the registry cannot be mutated at runtime.
- **REST (session-scoped, nested under the owning session; published by the owning
  context, backed by this context's registry and adapters; registered only when the
  registry is enabled):**
  - Upload — `POST /agent_versions/{v}/chat/{s}/artifacts` (BO, multipart; published by
    `feedback-datasets`; `409` when `chat_sessions.locked_at IS NOT NULL` — see
    [`feedback-datasets.md § Invariants`](feedback-datasets.md#invariants)) and
    `POST /deployed/{agent_id}/sessions/{sid}/artifacts?user_id=X` (deployed, multipart;
    published by `deployed-runtime`, additionally scoped by the gateway-verified `user_id`).
    Stages a **pending artifact** on the session and returns the pending-artifact object
    `{id, store_id, uri, filename, mime, size_bytes, sha256, status}`; the next turn claims it
    server-side — the client never embeds refs in the turn body (client-sent `artifact_ref`
    blocks are ignored). `409` at the `ARTIFACT_MAX_PENDING` cap; `413` past
    `ARTIFACT_MAX_UPLOAD_MB`.
  - Retrieval — `GET /agent_versions/{v}/chat/{s}/artifacts/{store_id}/{uri...}` (BO) and
    `GET /deployed/{agent_id}/sessions/{sid}/artifacts/{store_id}/{uri...}?user_id=X`
    (deployed). Authorised by a row in the owning context's session-artifact table for
    `(session_id, store_id, uri)` (any origin, pending or consumed) plus a registered
    `store_id`; then `adapter.Stat` — missing blob → `410`; then `302 Location <signed_url>`
    or `200` bytes per the adapter's declared `PreferredDelivery()` (see
    [`../../c4/integrations.md § Contracts`](../../c4/integrations.md#contracts)). Bytes
    responses carry `X-Content-Type-Options: nosniff` and `Content-Disposition` `inline` for
    native-render types, `attachment` otherwise; signed URLs carry the equivalent response
    overrides. Returns `404` on session-scope mismatch, no matching row, or unknown
    `store_id`.
  - Pending-artifact queue — `GET …/pending-artifacts` (list) and
    `DELETE …/pending-artifacts/{id}` (remove before the next turn) under each surface's
    session path; see the owning contexts' § Published surface.
- **Emitted contract for `feedback-datasets` and `deployed-runtime`:** the `artifact_ref`
  content-block shape embedded in their `.content` JSONB. Both contexts round-trip these
  blocks verbatim; only this context calls the store adapter.
- **Tool-call payloads are opaque.** The framework does not resolve, render or authorise any
  `{store_id, uri}` pointer inside `tool_use` input; it reaches the tool as opaque JSON.
  There is no agent-to-tool artifact leg.
- **Consumed contract from external object store:** artifact-store adapter interface (see
  [`../../c4/integrations.md § Contracts`](../../c4/integrations.md#contracts)) — `PreferredDelivery`
  (adapter-declared at construction, `Bytes | SignedURL`), `put`, `get` (called when
  `Bytes`) or `sign` (called when `SignedURL`; takes response content-type / disposition
  overrides), `delete`, `stat` (returns `{exists, mime, size_bytes, sha256}`; also drives
  upload dedup and the render fallback for a missing blob), `probe` (boot-time self-check,
  including that a missing key is reported as not-found). Adapter-config env-var schemas
  per kind live alongside the kind registration in
  [`../../c4/integrations.md`](../../c4/integrations.md).
- **Consumed contract from deploy env:** `ARTIFACT_STORE_<ID>_*` variables (one set per
  registered store) plus `ARTIFACT_STORE_DEFAULT=<ID>` (nominated write target) plus
  `ARTIFACT_MAX_UPLOAD_MB` (per-upload cap; required whenever artifact routes are wired)
  plus `ARTIFACT_SIGNED_URL_TTL_SECONDS` (optional, default `300`; passed as `ttl` to
  `adapter.Sign` when the adapter's `PreferredDelivery == SignedURL`). Parsed once at
  boot; failure to parse — or a duplicate `<ID>` across store groups — is a boot-time
  error.
