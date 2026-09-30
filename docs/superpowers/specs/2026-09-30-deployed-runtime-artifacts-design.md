# deployed-runtime-artifacts — session-bound artifacts for deployed and BO chat

**Date:** 2026-09-30
**Status:** Draft — pending `/spec-review`
**Modularity attachment:** `docs/architecture/current/modularity/open-bbcd/deployed-runtime/` (L2, primary), with `docs/architecture/current/modularity/open-bbcd/feedback-datasets/` (L2, BO surface) and `docs/architecture/current/modularity/open-bbcd/artifacts/` (L2, shared adapter substrate) in scope.
**Arch-of-record:** `deployed-runtime-artifacts` L2 capability (merged in `docs/architecture/logs/2026-09-28-artifact-support/`, PR #5). Builds on the shipped `chat-artifacts` feature (`docs/superpowers/specs/2026-09-28-chat-artifacts-design.md`, OpenBBC PR #53).

## Business value / Why

`chat-artifacts` shipped file exchange for the backoffice test chat only. The deployed runtime — the surface end users actually hit — has none of it: the deployed turn drops every non-text input block, the deployed orchestrator is not wired to the artifact registry (so MCP tool results carrying images/resources are never normalised and uploaded refs would never be rendered), there is no deployed upload or retrieval route, and the `ARTIFACT_REF` stream event promised by the arch does not exist on any surface. A downstream project needs end users to bring documents and screenshots into a conversation and to receive files produced by tools.

This spec also changes **how** artifacts enter a conversation. OpenBBC is not a file store: uploading is the user's signal "the next message is about this file". So user-supplied artifacts are **session-bound and staged** — the server records each one as a *pending artifact* on the session and adds every pending artifact to the session's next turn automatically. The client never sends `artifact_ref` blocks. This closes two holes that the BO design leaves open and that matter once untrusted end users are the callers:

- **Cross-session ref borrowing** — a client embedding another session's `{store_id, uri}` into its own turn, which would make the session-scope read check pass for a blob it never uploaded.
- **Client-asserted metadata at turn time** — a client lying in the turn body about `mime`/`size_bytes` of a ref. With staging, `size_bytes` and `sha256` are measured by the server at upload, and `mime` is resolved server-side from the bytes for every type that is rendered natively to the LLM (see Contracts § MIME resolution). The multipart `Content-Type` header is still client input, so it is trusted only for types the framework passes to the model as a text surrogate, where a wrong label cannot change what the model is sent.

BO test chat adopts the same model so that eval/dataset sessions exercise exactly the artifact behaviour production users get.

Success is judged by: an end user of a deployed agent can upload a PDF or image, send a message, and get an answer that reasons over the file; a tool returning an image produces an `ARTIFACT_REF` event the client can resolve; no request shape lets a caller read or attach a blob outside its own session.

**Terminology.** This spec uses the glossary's *Artifact* for the file object and avoids its listed aliases (*attachment*, *upload* as a noun). Two new terms, proposed for `glossary.md`:

- **Session artifact** — a row recording that an artifact belongs to a session's read scope (origin `upload` or `tool_result`).
- **Pending artifact** — a session artifact of origin `upload` not yet consumed by a turn.

"Upload" is used only as a verb and as the `origin` value naming how a session artifact arrived. The `[Attachment: …]` text-surrogate literal from `chat-artifacts` is unchanged.

## Change level

**C2.** The spec introduces two tables (migrations), new published REST routes on the deployed surface, a changed contract on the existing BO upload/retrieve/turn routes, a new stream event on both transports, a new env var, and new session-scope authorisation rules for untrusted callers. Touches DB, API, events, and security — C2 by `docs/process/README.md`.

## Scope

### In scope

- **Staged, session-bound artifacts on both surfaces.** Uploading records a pending artifact on the session; the next turn claims all pending artifacts atomically with persisting the user message and appends them as `artifact_ref` blocks.
- **Deployed routes:** upload and session-scoped retrieval, nested under the deployed session and enforcing the same `(agent_id deployed, session_id, user_id)` checks as `turn`.
- **BO routes:** same paths as shipped in PR #53; upload gains the pending-row write and the new response shape, retrieval switches from a JSONB message scan to a table lookup.
- **Turn contract change on both surfaces:** client-sent `artifact_ref` input blocks are ignored; a turn is valid with non-empty text **or** at least one pending artifact.
- **BO backfill** of `chat_session_artifacts` from refs already embedded in `chat_messages` (migration `027`).
- **Pending visibility:** deployed `GET session` response gains `pending_artifacts`; BO chat view renders pending chips server-side.
- **Tool-result artifacts recorded:** refs produced by MCP tool-result normalisation get a row (`origin = 'tool_result'`), so the table is the single read allow-list.
- **`ARTIFACT_REF` stream event** on both surfaces and both transports (AG-UI as a `CUSTOM` event; JSONL as `artifact_ref`).
- **Deployed orchestrator wiring** to the artifact registry (renderer + tool-result uploader), parity with BO.
- **`ARTIFACT_MAX_PENDING`** env var (default `10`) capping pending artifacts per session.
- **Minimal BO UI:** attach button on the chat form, pending chips, filename links for `artifact_ref` blocks in the transcript and for live `ARTIFACT_REF` events.
- **Deploy config:** Helm chart `artifacts:` values block (store groups, credentials from an existing Secret, caps); `docker-compose.yml` MinIO profile.

### Out of scope

- **Removing a pending artifact before the next turn.** No `DELETE` on pending rows; a mistakenly uploaded file is added to the next turn. Follow-up if users need it.
- **Client-chosen reuse of past artifacts** (re-adding a file from an earlier turn). Artifacts are consumed once; they remain in history as refs and the LLM sees them there.
- **The text-resource regression** — MCP `EmbeddedResource` with inline `text` is normalised to a `text/plain` artifact that Anthropic renders as a `[Attachment: …]` surrogate, so the model loses text it previously saw. Separate spec (artifact handling strategy / text-like MIME rendering).
- **Native multimodal rendering for non-Anthropic providers.**
- **Blob GC / lifecycle** — unchanged from `chat-artifacts` (deployer uses native store lifecycle rules). Session delete cascades rows, not blobs.
- **Per-session header overrides on outbound MCP calls** from the deployed runtime (existing open question, unrelated).
- **BO UI polish** — thumbnails, previews, drag-and-drop.

## Contracts

### Data — new tables

Two tables with identical shape, one per session-owning context (`docs/conventions/persistence.md` forbids cross-context shared tables; the `artifacts` L2 continues to own no tables).

| Table | Owning context / L2 | Migration | FK |
|---|---|---|---|
| `chat_session_artifacts` | `feedback-datasets` | `027_chat_session_artifacts.sql` | `session_id → chat_sessions(id) ON DELETE CASCADE` |
| `deployed_session_artifacts` | `deployed-runtime` | `028_deployed_session_artifacts.sql` | `session_id → deployed_sessions(id) ON DELETE CASCADE` |

Columns (both tables):

| Column | Type | Notes |
|---|---|---|
| `id` | `uuid` PK, default `gen_random_uuid()` | returned to the client as the pending-artifact id |
| `session_id` | `uuid NOT NULL` | FK as above |
| `origin` | `text NOT NULL CHECK (origin IN ('upload','tool_result'))` | |
| `store_id` | `text NOT NULL` | registry slug the blob was written to |
| `uri` | `text NOT NULL` | `sha256/<hex>` |
| `mime` | `text NOT NULL` | resolved server-side per § MIME resolution; never taken from the turn body |
| `size_bytes` | `bigint NOT NULL` | server-measured |
| `sha256` | `text NOT NULL` | server-computed |
| `filename` | `text NULL` | display label only; never used to build a storage URI. May contain user-provided PII — see Risks |
| `consumed_by_message_id` | `uuid NULL` | `NULL` = pending; otherwise the message the ref was added to |
| `created_at`, `updated_at` | `timestamptz NOT NULL DEFAULT now()` | |

Constraints / indexes:

- `UNIQUE (session_id, store_id, uri) WHERE origin = 'upload' AND consumed_by_message_id IS NULL` — at most one pending row per blob per session; an identical upload while pending returns the existing row. A consumed row and a new pending row for the same blob may coexist (re-uploading is a new intent to discuss the file). Tool-result rows are unconstrained — the same tool may return the same image twice; each is a separate record.
- `INDEX (session_id) WHERE consumed_by_message_id IS NULL` — claim path.
- `INDEX (session_id, store_id, uri)` — retrieval allow-list lookup.

`consumed_by_message_id` is deliberately not an FK: messages and artifact rows are written in the same transaction on the claim path, and the tool-result path records rows after the tool message is appended; a plain column avoids ordering coupling. Both are removed together by the session cascade.

Both migrations are additive (new tables only); the previous app version ignores them, so rollout and rollback are safe.

### MIME resolution (both origins)

The server has the bytes in hand on both write paths (multipart upload and MCP tool-result normalisation), so it resolves `mime` before writing the row:

1. `declared` = multipart part `Content-Type` (upload) or MCP `mimeType` (tool_result), parameters stripped, lower-cased; `application/octet-stream` if absent.
2. `sniffed` = `http.DetectContentType` over the first 512 bytes, parameters stripped.
3. **Native-render set** = `image/png`, `image/jpeg`, `image/gif`, `image/webp`, `application/pdf` (the MIMEs any shipped `MultimodalRenderer` inlines as bytes).
4. If `sniffed` is in the native-render set → `mime = sniffed` (whatever was declared).
5. Else if `declared` is in the native-render set (sniff disagrees) → `mime = application/octet-stream`. A native-render label is never stored for bytes that don't look like it.
6. Else → `mime = declared`. Types outside the native-render set only ever reach the model as a text surrogate, so a wrong label is cosmetic.
7. **Structural check for images** in the native-render set: `image.DecodeConfig` must succeed (stdlib `image/png`, `image/jpeg`, `image/gif`; `golang.org/x/image/webp`). On failure → `mime = application/octet-stream`. The file is kept and retrievable; it is never sent to the model as an image.

Nothing is rejected by this step — resolution only decides the stored label. `size_bytes` and `sha256` are always measured over the received bytes.

**BO backfill (in `027`).** Refs written to `chat_messages` under PR #53 have no row, so they would become unreadable. `027` inserts one `chat_session_artifacts` row per `artifact_ref` block found in existing `chat_messages.content` arrays: `origin = 'upload'` for user-role messages, `'tool_result'` for tool-role, `consumed_by_message_id` = the message id, `created_at` = the message's `created_at`. Non-array (legacy text) content is skipped. `deployed_messages` needs no backfill — the deployed turn never accepted refs.

### Data — message content

Unchanged shape from `chat-artifacts`: `chat_messages.content` / `deployed_messages.content` hold typed block arrays with `text` and `artifact_ref` blocks (`{type, store_id, uri, mime, size_bytes, sha256, filename?}`). What changes is provenance: every `artifact_ref` on a user-role message is now built server-side from a claimed row, never from client input.

### REST — deployed (new)

All deployed artifact routes run the `turn` preamble: `requireDeployed(agent_id)`; `GetSession(session_id, user_id)`; `sess.AgentID == agent_id`. Any failure → `404` (no existence leak). `user_id` is a query parameter on both routes (upload body is multipart). Routes are registered only when the artifact registry is enabled.

**`POST /deployed/{agent_id}/sessions/{session_id}/artifacts?user_id=X`** — `multipart/form-data`, single field `file`.

`201 Created`:

```json
{
  "id":         "5b0e…",
  "filename":   "Q3-report.pdf",
  "mime":       "application/pdf",
  "size_bytes": 245678,
  "sha256":     "9c1f2b7d…",
  "status":     "pending"
}
```

`status` is always `"pending"` in this response (a re-upload of content already pending returns the existing row; a re-upload of content already consumed in this session creates a new pending row).

| Status | Cause |
|---|---|
| `400` | No `file` field / malformed multipart / missing `user_id` |
| `404` | Agent not deployed, session missing, user mismatch, or agent mismatch |
| `409` | Session already has `ARTIFACT_MAX_PENDING` pending artifacts |
| `413` | Upload exceeds `ARTIFACT_MAX_UPLOAD_MB` (before any store call) |
| `502` | `adapter.Stat` / `adapter.Put` failed |

**`GET /deployed/{agent_id}/sessions/{session_id}/artifacts/{store_id}/{uri...}?user_id=X`** — retrieval. `{uri...}` is a Go 1.22 wildcard so `sha256/<hex>` passes intact.

Authorisation: preamble + a row exists in `deployed_session_artifacts` for `(session_id, store_id, uri)` (any origin, pending or consumed) + `store_id` is registered. Delivery per adapter `PreferredDelivery()`: `302 Location: <signed-url>` (TTL `ARTIFACT_SIGNED_URL_TTL_SECONDS`) or `200` with streamed body. Response headers — see § Retrieval response headers.

| Status | Cause |
|---|---|
| `400` | Missing `user_id`, or path not `<store_id>/<uri>` |
| `404` | Preamble failure, no matching row, or `store_id` not registered |
| `410` | Blob externally removed (`ErrBlobMissing`) |
| `502` | Other adapter error |

**Retrieval response headers (both surfaces).** Artifact bytes are user- or tool-supplied and served from an origin the client may render, so:

- **Bytes delivery (`200`):** `Content-Type: <row.mime>`, `Content-Length: <row.size_bytes>`, `X-Content-Type-Options: nosniff`, and `Content-Disposition: inline; filename*=UTF-8''<row.filename>` when `row.mime` is in the native-render set, `attachment; filename*=…` otherwise (filename omitted when the row has none).
- **Signed-URL delivery (`302`):** the signed URL carries the same `Content-Type` and `Content-Disposition` as response overrides (S3 `response-content-type` / `response-content-disposition` presign parameters), so the store serves them. This extends the adapter method to `Sign(ctx, uri, ttl, SignOptions{ContentType, ContentDisposition})`; kinds that cannot set response overrides ignore them. Per-request overrides are required because content-addressed blobs are shared by rows with different filenames.

**`GET /deployed/{agent_id}/sessions/{session_id}?user_id=X`** — existing route; response gains an additive field:

```json
{ "session": {…}, "messages": […], "pending_artifacts": [ { "id", "filename", "mime", "size_bytes", "sha256", "status": "pending" } ] }
```

Ordered by `created_at`. Always present; `[]` when none or when the registry is disabled.

### REST — BO (existing paths, changed behaviour)

- **`POST /agent_versions/{version_id}/chat/{session_id}/artifacts`** — same preamble as today (`GetSession(session_id, version_id)`, `409` if `locked_at IS NOT NULL`). Now writes a pending `chat_session_artifacts` row and returns the same response shape as the deployed upload (the previous `{store_id, uri, mime, size_bytes, sha256, filename}` shape is replaced). Adds the `409` pending-cap case. Other statuses unchanged.
- **`GET /agent_versions/{version_id}/chat/{session_id}/artifacts/{store_id}/{uri...}`** — authorisation switches from the `chat_messages` JSONB scan (`SessionReferences`) to a row lookup in `chat_session_artifacts`. Response headers per § Retrieval response headers (adds `Content-Length`, `nosniff`, `Content-Disposition`). Statuses unchanged.
- **Chat view** (`GET /agent_versions/{v}/chat/{s}`) renders pending rows as chips.

### REST — turn (both surfaces)

- `POST /deployed/{agent_id}/sessions/{session_id}/turn` and `POST /agent_versions/{v}/chat/{s}/turn`: input blocks of type `artifact_ref` are **ignored** (not rejected — older clients that still send them keep working; the blocks are simply dropped). Only `text` blocks are read from the body.
- Validity: a turn with no non-empty `text` block and no pending artifacts is rejected exactly as an empty turn is today. A turn with no text but ≥1 pending artifact is accepted.
- The claimed refs appear on the persisted user message after its text blocks, in `created_at` order.

### Claim semantics (store layer)

Both session stores gain:

```go
// AppendUserTurn persists the user message and claims every pending artifact on
// the session in one transaction. Claimed refs are appended to msg.Content after
// its existing blocks, in created_at order, and returned for the caller.
AppendUserTurn(ctx context.Context, agentVersionID string, msg types.ChatMessage) ([]llm.ArtifactRefBlock, error)

// RecordToolArtifacts inserts origin='tool_result' rows, already consumed by messageID.
RecordToolArtifacts(ctx context.Context, sessionID, messageID string, refs []llm.ArtifactRefBlock) error
```

Claim SQL (inside the message-insert transaction):

```sql
UPDATE <table> SET consumed_by_message_id = $msg, updated_at = now()
WHERE session_id = $s AND origin = 'upload' AND consumed_by_message_id IS NULL
RETURNING store_id, uri, mime, size_bytes, sha256, filename, created_at
```

Rows are ordered by `created_at` in Go after `RETURNING`. Concurrent turns on one session: row locks make the second `UPDATE` wait and then match zero rows, so each pending artifact is added to exactly one message. If the turn fails after the transaction commits (LLM error, tool error), the refs are already in history and are seen on the next turn — nothing is lost or added twice.

The orchestrator calls `AppendUserTurn` in place of its current user-message `AppendMessages` call, and `RecordToolArtifacts` after appending a tool-role message that carries normalised refs. Internal callers of `Orchestrator.Turn` may still pass `artifact_ref` input blocks directly; only the HTTP parsers drop them.

### Rendering robustness

A consumed artifact stays in history and is re-rendered on every later turn, so a ref that makes rendering fail would fail every later turn of the session. Rules in `renderArtifactsForLLM` and the Anthropic renderer:

- **Blob missing** (`ErrBlobMissing` from `Get`/`Sign`/signed fetch) → text surrogate, `warn` log. Permanent condition; failing would brick the session. (Previously: turn failed.)
- **Other store errors** (network, 5xx, auth) → turn fails as today (`artifact_render`). Transient; the next turn retries.
- **Provider limits** — the renderer returns `ErrUnsupported` (→ text surrogate) when `size_bytes` exceeds the provider's documented per-block limit for that MIME. Anthropic limits are pinned in the renderer from the provider's documentation at implementation time.
- **Stored MIME is trusted by the renderer** because § MIME resolution guarantees native-render labels only on bytes that sniff (and, for images, decode) as that type.

### Stream event — `ARTIFACT_REF`

Internal: `transport.ArtifactRefEvent{ToolCallID, StoreID, URI, MIME, SizeBytes, Sha256, Filename}`. Emitted by the orchestrator immediately after the `ToolResultEvent` for a tool call whose result produced refs, one event per ref. Not emitted for user uploads (the uploading client already knows them).

**AG-UI encoding** — AG-UI `CUSTOM` event (the protocol's extension mechanism; official SDKs validate `type` against a closed set, so a new top-level type would fail validation instead of being ignored):

```json
{ "type": "CUSTOM", "name": "ARTIFACT_REF",
  "value": { "toolCallId": "…", "storeId": "MAIN", "uri": "sha256/…", "mime": "image/png",
             "sizeBytes": 1234, "sha256": "…", "filename": null } }
```

Field casing follows the existing AG-UI encoder's camelCase.

**JSONL encoding:**

```json
{ "type": "artifact_ref", "tool_call_id": "…", "store_id": "MAIN", "uri": "sha256/…",
  "mime": "image/png", "size_bytes": 1234, "sha256": "…", "filename": null }
```

Clients build the retrieval URL from `store_id` + `uri` against their own surface's route. Emitted on both BO and deployed (shared orchestrator).

### Env

| Variable | Meaning |
|---|---|
| `ARTIFACT_MAX_PENDING=<int>` | New. Optional, default `10`, must be ≥1. Max pending artifacts per session; exceeding → `409` on upload. Parsed with the rest of `config.ArtifactsConfig`; invalid value fails boot. |

Existing `ARTIFACT_STORE_<ID>_*`, `ARTIFACT_STORE_DEFAULT`, `ARTIFACT_MAX_UPLOAD_MB`, `ARTIFACT_SIGNED_URL_TTL_SECONDS` unchanged.

**Registry disabled** (no store groups): no artifact routes on either surface, turns behave as today, `pending_artifacts` is `[]`, no `ARTIFACT_REF` events.

### Deploy config

- Helm chart: `artifacts.enabled`, `artifacts.stores.<ID>.{kind, endpoint, bucket, region, pathStyle}`, `artifacts.stores.<ID>.existingSecret` (keys `accessKey`, `secretKey`), `artifacts.default`, `artifacts.maxUploadMB`, `artifacts.maxPending`, `artifacts.signedUrlTTLSeconds` → rendered as the env vars above. Disabled by default.
- `docker-compose.yml`: `artifacts` profile adding a MinIO service + bucket init and the env for a `MAIN` store.

### Arch deltas for `/arch-review` (post-approval sync — not edited by this spec)

- `ddd/contexts/artifacts.md § Published surface` and `modularity/open-bbcd/artifacts/README.md`: replace stale `/chat-sessions/{id}/artifacts` and top-level `/artifacts/{store_id}/{uri}` paths with the nested per-surface routes; replace "client embeds the returned ref in the outgoing turn body" with the staged pending → consumed-on-next-turn model; session-scope enforced via the per-context artifact table, not a JSONB scan.
- `modularity/open-bbcd/deployed-runtime/README.md` and `ddd/contexts/deployed-runtime.md`: own `deployed_session_artifacts`; publish the nested retrieval route and `pending_artifacts` field.
- `modularity/open-bbcd/feedback-datasets/README.md` and `ddd/contexts/feedback-datasets.md`: own `chat_session_artifacts`.
- `c4/integrations.md § AG-UI ARTIFACT_REF`: carried as AG-UI `CUSTOM` event `name: "ARTIFACT_REF"`; correct the "SDKs ignore unknown events" assumption.
- `ddd/access-model.md`: deployed retrieval path; BO and deployed read allow-list = own session's artifact rows.
- `constraints.md` / `c4/deployment.md`: `ARTIFACT_MAX_PENDING`.
- `ddd/contexts/artifacts.md § Aggregates & entities`: the "no artifacts table", "not a standalone entity" and "source of truth is the union of `artifact_ref` blocks" statements no longer hold — session artifacts have identity and a pending → consumed lifecycle (stored in the owning contexts' tables, not in `artifacts`). Also the § Invariants MCP normalisation line (URI-only `EmbeddedResource` is not normalised) and the MIME-resolution rule.
- `c4/containers.md` (Postgres data-ownership line) and `modularity/open-bbcd/README.md` (L1 owned tables): add both tables.
- `bizbok/information-map.md`: Artifact row + the new *Session artifact* / *Pending artifact* concepts. `glossary.md`: add those two terms; fix *Content block* ("pointer to an `artifact_stores` row" — there is no such table); fix *Artifact* ("four legs" — agent emission was dropped).
- `c4/data-flows.md § Multimodal chat with artifacts` and `bizbok/value-streams.md § Multimodal chat with artifacts`: turn content is text only; pending artifacts are added server-side.
- Stale route paths: `nfrs.md § Security`, `c4/deployment.md` threat-model row "Artifact ref leak", `constraints.md` `ARTIFACT_MAX_UPLOAD_MB` bullet, `ddd/contexts/feedback-datasets.md` invariant naming `/chat-sessions/{id}/artifacts`.
- `ddd/contexts/deployed-runtime.md § Published surface` ("older SDKs see it as an unknown event") and `modularity/open-bbcd/deployed-runtime/README.md` (upload response, `ARTIFACT_REF` wording).
- `c4/integrations.md` artifact-store adapter contract: `Sign` gains `SignOptions` (response content-type / disposition overrides).

## Acceptance criteria

Integration tests run against MinIO (compose `artifacts` profile) unless marked unit.

### Upload (both surfaces)

- Deployed upload with valid `file`, matching `user_id` returns `201` with `{id, filename, mime, size_bytes, sha256, status:"pending"}`; `sha256` equals the sha256 of the bytes; a `deployed_session_artifacts` row exists with `origin='upload'`, `consumed_by_message_id IS NULL`.
- BO upload returns the same shape and writes a `chat_session_artifacts` row.
- Deployed upload with a `user_id` that does not own the session → `404`; with the session under a different `agent_id` → `404`; for an agent with no DEPLOYED version → `404`; missing `user_id` → `400`.
- BO upload on a session with `locked_at IS NOT NULL` → `409`.
- The `ARTIFACT_MAX_PENDING + 1`-th pending artifact on a session → `409`; after a turn consumes them, uploads succeed again.
- Upload over `ARTIFACT_MAX_UPLOAD_MB` → `413` and no store call (mock spy, unit).
- Uploading identical bytes twice while the first is pending returns the same `id` and creates no second row; `adapter.Put` is called at most once.
- Uploading bytes identical to an already-consumed upload in the same session creates a new pending row.

### MIME resolution + retrieval headers

- Upload of real PNG bytes declared `application/octet-stream` → row `mime = image/png` (unit).
- Upload of non-image bytes declared `image/png` → row `mime = application/octet-stream`; a later turn renders it as a text surrogate, never as an image block (unit).
- Upload of a truncated PNG (sniffs as PNG, `image.DecodeConfig` fails) → row `mime = application/octet-stream` (unit).
- Upload of a `.docx` declared with its Office MIME keeps the declared MIME (unit).
- MCP `ImageContent` with `mimeType: image/jpeg` whose bytes are PNG → row `mime = image/png` (unit).
- Bytes-mode retrieval sets `X-Content-Type-Options: nosniff`; `Content-Disposition` is `inline` for `image/png` and `attachment` for `application/zip`, with the row's filename (unit).
- Signed-URL retrieval: the signed URL carries `response-content-type` and `response-content-disposition` matching the row (integration, MinIO).

### Rendering robustness

- A consumed ref whose blob was removed from the bucket: the next turn succeeds, the LLM request carries the text surrogate for it, and a `warn` is logged.
- A store network error during rendering fails the turn with `artifact_render`; the next turn with the store healthy renders natively.
- An `image/png` ref above the pinned Anthropic image limit is rendered as a text surrogate (unit).

### Turn / claim

- After two uploads (A then B) and a turn with text "summarise", the persisted user message content is `[text, artifact_ref(A), artifact_ref(B)]`; both rows have `consumed_by_message_id` = that message's id.
- The `artifact_ref` blocks' `mime`/`size_bytes`/`sha256`/`filename` equal the row values.
- A turn body containing an `artifact_ref` block (any `store_id`/`uri`) does not add that block to the persisted message (both surfaces).
- A turn with empty text and one pending artifact is accepted and the message contains only the `artifact_ref` block; a turn with empty text and no pending artifacts is rejected as today.
- A turn with no pending artifacts persists exactly as today (no behaviour change).
- Two concurrent turns on one session with one pending artifact: exactly one persisted user message carries the ref (unit test against Postgres with parallel transactions).
- An LLM failure after the user message is persisted leaves the rows consumed and the refs in history; the next turn's LLM request includes them.
- With the Anthropic provider, a claimed `image/png` ref is sent as a native image block and `application/pdf` as a document block on the deployed surface (parity with BO).

### Tool-result artifacts + event

- A deployed turn whose MCP tool returns `ImageContent` produces: an `artifact_ref` on the tool-role message, a `deployed_session_artifacts` row with `origin='tool_result'` consumed by that message, and an AG-UI `CUSTOM` event with `name:"ARTIFACT_REF"` emitted after the corresponding tool-result event, carrying the matching `toolCallId`, `storeId`, `uri`, `mime`, `sizeBytes`, `sha256`.
- Same on BO.
- JSONL transport emits `{"type":"artifact_ref", …}` with the same fields (unit).
- No `ARTIFACT_REF` event is emitted for claimed user uploads.

### Retrieval

- Deployed GET for a ref uploaded to the session (pending or consumed) returns `302` to a URL that serves the bytes; BO GET likewise.
- GET for a tool-result ref recorded on the session succeeds.
- GET for a `(store_id, uri)` that exists in the store but has no row for this session → `404`, including when the same blob has a row in another session.
- GET with wrong `user_id`, wrong `agent_id`, or unregistered `store_id` → `404`; missing `user_id` → `400`.
- GET after the blob is removed from the bucket → `410`.
- Bytes-mode adapter (mock, unit): `200` with `Content-Type` = row `mime` and `Content-Length` = row `size_bytes`.

### Pending visibility

- Deployed `GET session` returns `pending_artifacts` listing unconsumed uploads in `created_at` order; `[]` after a turn consumes them; `[]` when the registry is disabled.
- BO chat view renders a chip per pending artifact on page load.

### BO backfill

- Given a `chat_messages` row written under PR #53 with one user-role `artifact_ref` block, after migration `027` a `chat_session_artifacts` row exists with `origin='upload'` and `consumed_by_message_id` = that message's id, and BO GET for the ref succeeds.
- A tool-role message's `artifact_ref` backfills with `origin='tool_result'`; legacy non-array content rows are skipped without error.

### Lifecycle, config, feature-off

- Deleting a deployed session removes its `deployed_session_artifacts` rows (cascade); blobs remain in the store.
- `ARTIFACT_MAX_PENDING=0` or non-integer fails boot naming the var; unset defaults to `10`.
- With no `ARTIFACT_STORE_*` groups, deployed artifact routes return `404` (unregistered), turns are unchanged, and no `ARTIFACT_REF` events are emitted.
- Helm chart with `artifacts.enabled=true` renders all artifact env vars with credentials sourced from the named Secret (`helm template` snapshot test).

## Risks & assumptions

### Assumptions

- **The operator gateway rewrites `user_id` to the verified identity** on every deployed route, including the new upload and retrieval routes. Same trust model as existing deployed routes; the daemon does no authentication.
- **All pending artifacts going with the next turn is the right UX.** Users upload, then send; everything pending goes with the next message. No per-file choice at send time.
- **`ARTIFACT_MAX_PENDING=10` is a sensible default** for LLM request size alongside `ARTIFACT_MAX_UPLOAD_MB`; deployers tune it.
- **The existing AG-UI encoder can emit `CUSTOM` events** without a library change (hand-rolled SSE encoder, verified at implementation time).
- **No internal caller other than the HTTP turn handlers depends on client-embedded refs.** Verified for eval/training (neither drives `Orchestrator.Turn`); re-checked at implementation.

### Risks

- **Breaking change to the BO upload response and turn contract.** Any client built against PR #53's "embed the returned ref" flow stops attaching files (its refs are now ignored). Mitigation: PR #53 shipped days ago with no BO UI; the only known consumer is the downstream project, which is informed via the release notes. Refs are ignored rather than rejected so such clients degrade to text-only instead of erroring.
- **BO refs written under PR #53 have no table row**, so their retrieval now returns `404`. Mitigation: migration `027` backfills them (see Contracts § BO backfill), so datasets closed on such sessions keep resolvable refs. Residual risk: a ref embedded in a shape the backfill query does not recognise stays unreadable; the backfill counts inserted rows in the migration log.
- **A mistaken upload cannot be withdrawn** and will be sent to the LLM on the next turn. Accepted for this phase; follow-up adds `DELETE` for pending rows.
- **`filename` may carry PII** (user-chosen file names). Classified as user-provided free text; not logged by the upload handler; removed with the session cascade.
- **Signed URLs in the `302` are bearer URLs for their TTL.** Unchanged from `chat-artifacts`; tune `ARTIFACT_SIGNED_URL_TTL_SECONDS`.
- **Clients unaware of `CUSTOM ARTIFACT_REF`** see tool results as text only. Acceptable degradation; the ref is still in persisted history and visible via `GET session`.
- **Provider rejects a request despite the checks** (e.g. a PDF over the provider's page limit, an encrypted PDF, an image the provider decodes differently). Because consumed refs are re-sent every turn, every later turn of that session fails. Mitigated by MIME resolution, the image structural check, and provider size limits; residual risk accepted for this phase. Follow-up: record a per-ref render-failure marker so a ref that caused a provider `4xx` falls back to a text surrogate.
- **MIME resolution changes labels on write.** A file declared `image/png` that doesn't decode is stored and served as `application/octet-stream`; the user gets it as a download rather than inline. Intended.
- **Pending rows on sessions that never get another turn** accumulate as orphan rows and blobs. Bounded by `ARTIFACT_MAX_PENDING` per session; removed on session delete.

## Cross-references

- Parent feature: `docs/superpowers/specs/2026-09-28-chat-artifacts-design.md` (BO, OpenBBC PR #53).
- Arch-of-record: `docs/architecture/logs/2026-09-28-artifact-support/README.md`; `docs/architecture/current/ddd/contexts/artifacts.md`; `docs/architecture/current/ddd/access-model.md`; `docs/architecture/current/c4/integrations.md § AG-UI ARTIFACT_REF`.
- Modularity: `docs/architecture/current/modularity/open-bbcd/{deployed-runtime,feedback-datasets,artifacts}/README.md`.
- Conventions: `docs/conventions/persistence.md` (per-context ownership, additive migrations, PII).
- Follow-ups: pending-artifact removal; artifact handling strategy (text-like MIME rendering, pluggable fallback); per-provider native multimodal; `chat-artifacts-gc`.
