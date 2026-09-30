# deployed-runtime-artifacts — session-bound artifacts for deployed and BO chat

**Date:** 2026-09-30
**Status:** Approved (PR #10). Amended: upload locking, normalisation fallback, native-render budget, eval replay scope, event ordering.
**Modularity attachment:** `docs/architecture/current/modularity/open-bbcd/deployed-runtime/` (L2, primary), with `docs/architecture/current/modularity/open-bbcd/feedback-datasets/` (L2, BO surface) and `docs/architecture/current/modularity/open-bbcd/artifacts/` (L2, shared adapter substrate) in scope.
**Arch-of-record:** `deployed-runtime-artifacts` L2 capability (merged in `docs/architecture/logs/2026-09-28-artifact-support/`, PR #5). Builds on the shipped `chat-artifacts` feature (`docs/superpowers/specs/2026-09-28-chat-artifacts-design.md`, OpenBBC PR #53).

## Business value / Why

`chat-artifacts` shipped file exchange for the backoffice test chat only. The deployed runtime — the surface end users actually hit — has none of it: the deployed turn drops every non-text input block, the deployed orchestrator is not wired to the artifact registry (so MCP tool results carrying images/resources are never normalised and uploaded refs would never be rendered), there is no deployed upload or retrieval route, and the `ARTIFACT_REF` stream event promised by the arch does not exist on any surface. A downstream project needs end users to bring documents and screenshots into a conversation and to receive files produced by tools.

BO itself is only half-working: the shared message codec (`blocksToJSON` / `parseBlocks` in `internal/chat/orchestrator.go`) has no `artifact_ref` case, so refs reach the LLM for the one turn they arrive in and are then dropped from persisted history. Nothing is ever retrievable afterwards and artifacts do not carry into later turns. This spec fixes that for both surfaces.

This spec also changes **how** artifacts enter a conversation. OpenBBC is not a file store: uploading is the user's signal "the next message is about this file". So user-supplied artifacts are **session-bound and staged** — the server records each one as a *pending artifact* on the session and adds every pending artifact to the session's next turn automatically. The client never sends `artifact_ref` blocks. This closes two holes that the BO design leaves open and that matter once untrusted end users are the callers:

- **Cross-session ref borrowing** — a client embedding another session's `{store_id, uri}` into its own turn, which would make the session-scope read check pass for a blob it never uploaded.
- **Client-asserted metadata at turn time** — a client lying in the turn body about `mime`/`size_bytes` of a ref. With staging, `size_bytes` and `sha256` are measured by the server at upload, and `mime` is resolved server-side from the bytes for every type that is rendered natively to the LLM (see Contracts § MIME resolution). The multipart `Content-Type` header is still client input, so it is trusted only for types the framework passes to the model as a text surrogate, where a wrong label cannot change what the model is sent.

BO test chat adopts the same model so that interactive test sessions exercise exactly the artifact behaviour production users get. (Eval/training *replay* of sessions that carry artifacts is out of scope — see Scope.)

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
- **Deployed routes:** upload, pending-artifact list/remove, and session-scoped retrieval, nested under the deployed session and enforcing the same `(agent_id deployed, session_id, user_id)` checks as `turn`.
- **BO routes:** same paths as shipped in PR #53; upload gains the pending-row write and additive response fields (`id`, `status`), retrieval switches from a JSONB message scan to a table lookup.
- **Turn contract change on both surfaces:** client-sent `artifact_ref` input blocks are ignored; a turn is valid with non-empty text **or** at least one pending artifact; an empty turn is refused with `400` (new — today empty turns are not rejected).
- **Persist and rehydrate `artifact_ref` blocks** in the shared message codec, for user- and tool-role messages on both surfaces. Provider-rendered media (`InlineMediaBlock`) is never persisted.
- **Pending-artifact queue endpoints on both surfaces:** list the session's pending artifacts and remove one before the next turn. BO chat view also renders pending chips server-side.
- **Tool-result artifacts recorded:** refs produced by MCP tool-result normalisation get a row (`origin = 'tool_result'`), so the table is the single read allow-list.
- **`ARTIFACT_REF` stream event** on both surfaces and both transports (AG-UI as a `CUSTOM` event; JSONL as `artifact_ref`).
- **Deployed orchestrator wiring** to the artifact registry (renderer + tool-result uploader), parity with BO.
- **`ARTIFACT_MAX_PENDING`** env var (default `10`) capping pending artifacts per session.
- **Minimal BO UI:** add-file button on the chat form, pending chips, filename links for `artifact_ref` blocks in the transcript and for live `ARTIFACT_REF` events.
- **Deploy config:** Helm chart `artifacts:` values block (store groups, credentials from an existing Secret, caps); `docker-compose.yml` MinIO profile.

### Out of scope

- **Client-chosen reuse of past artifacts** (re-adding a file from an earlier turn). Artifacts are consumed once; they remain in history as refs and the LLM sees them there.
- **The text-resource regression** — MCP `EmbeddedResource` with inline `text` is normalised to a `text/plain` artifact that Anthropic renders as a `[Attachment: …]` surrogate, so the model loses text it previously saw. Separate spec (artifact handling strategy / text-like MIME rendering).
- **Native multimodal rendering for non-Anthropic providers.**
- **Blob GC / lifecycle** — unchanged from `chat-artifacts` (deployer uses native store lifecycle rules). Session delete cascades rows, not blobs.
- **Per-session header overrides on outbound MCP calls** from the deployed runtime (existing open question, unrelated).
- **BO UI polish** — thumbnails, previews, drag-and-drop.
- **Eval / training replay of artifacts.** `internal/eval/export.go` passes message content through to `aikdm`, whose simulator (`aikdm/eval/simulator.py`) produces text-only user turns and has no artifact handling. Datasets built from BO sessions that carry artifacts replay **without** the files. Follow-up spec: artifact-aware eval replay (resolve refs through the session-scoped read path and feed them to the simulated turn).
- **Passing artifacts across the agent-tool boundary**, in either direction, and any artifact-scope inheritance between a root session and its child sessions. See Contracts § Sub-agents. This narrows the merged `multiagent-tools` architecture (which lets refs flow both ways) and is listed as an arch delta.
- **Framework resolution of inner-ref pointers** (`{store_id, uri}` inside `tool_use` input). The framework never dereferences, renders or authorises them; they reach the tool as opaque JSON.
- **Backfill of PR #53 data.** None is needed: PR #53 never persisted `artifact_ref` blocks (see Business value), so no `chat_messages` row holds a ref. Blobs uploaded under PR #53 remain in the store unreferenced.

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
| `message_id` | `uuid NULL` | `NULL` = pending (origin `upload` only); otherwise the message carrying the ref — the claiming user message, or the tool-role message for `tool_result` rows |
| `created_at`, `updated_at` | `timestamptz NOT NULL DEFAULT now()` | |

Constraints / indexes:

- `UNIQUE (session_id, store_id, uri) WHERE origin = 'upload' AND message_id IS NULL` — at most one pending row per blob per session; an identical upload while pending returns the existing row. A consumed row and a new pending row for the same blob may coexist (re-uploading is a new intent to discuss the file). Tool-result rows are unconstrained — the same tool may return the same image twice; each is a separate record.
- `INDEX (session_id) WHERE message_id IS NULL` — claim path.
- `INDEX (session_id, store_id, uri)` — retrieval allow-list lookup.

`message_id` is deliberately not an FK: it references `chat_messages` or `deployed_messages` depending on the table, and both are removed together by the session cascade. On both write paths the message and its rows are written in one transaction (§ Store-layer writes).

Both migrations are additive (new tables only, no data backfill); the previous app version ignores them, so rollout and rollback are safe. Down migrations drop the tables. The numbers `027`/`028` are the next free ones today; if `multiagent-feature` migrations land first, these take the next free numbers at implementation.

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

### Data — message content

Shape as declared by `chat-artifacts`: `chat_messages.content` / `deployed_messages.content` hold typed block arrays with `text`, `tool_use`, `tool_result` and `artifact_ref` blocks. `artifact_ref` = `{type, store_id, uri, mime, size_bytes, sha256, filename?}`.

**Codec change (new).** `blocksToJSON` gains an `llm.ArtifactRefBlock` case writing exactly that shape; `parseBlocks` gains the matching `artifact_ref` case. `llm.InlineMediaBlock` (bytes produced by a renderer for one LLM request) is never persisted: `blocksToJSON` returns an error if it meets one, so a code path that tries to persist rendered bytes fails loudly in tests rather than writing bytes to Postgres.

**Provenance.** Every `artifact_ref` on a user-role message is built server-side from a claimed row; every `artifact_ref` on a tool-role message comes from MCP tool-result normalisation. Neither is ever taken from client input or from LLM output.

### REST conventions

New routes follow the daemon's existing route conventions rather than `docs/conventions/api.md`: no `/v1` prefix, nesting under the owning session, plain-text error bodies via the existing `Error`/`http.Error` helpers, no `Idempotency-Key`. Upload is naturally idempotent while a row is pending (content-hash dedup); `DELETE` is idempotent in effect (a repeat returns `404`). Aligning the daemon with `api.md` is a separate, daemon-wide change.

**Two collections, one concept.** `…/artifacts` addresses blobs by `(store_id, uri)` (upload, retrieval). `…/pending-artifacts` addresses queue entries by row `id` (list, remove). The `id` returned by upload is only addressable under `…/pending-artifacts/{id}`; there is no `DELETE …/artifacts/{id}`.

### REST — deployed (new)

All deployed artifact routes run the `turn` preamble: `requireDeployed(agent_id)`; `GetSession(session_id, user_id)`; `sess.AgentID == agent_id`. Any failure → `404` (no existence leak). `user_id` is a query parameter on every artifact route (upload body is multipart; `GET`/`DELETE` have no body). Routes are registered only when the artifact registry is enabled.

**`POST /deployed/{agent_id}/sessions/{session_id}/artifacts?user_id=X`** — `multipart/form-data`, single field `file`.

`201 Created`:

```json
{
  "id":         "5b0e…",
  "store_id":   "MAIN",
  "uri":        "sha256/9c1f2b7d…",
  "filename":   "Q3-report.pdf",
  "mime":       "application/pdf",
  "size_bytes": 245678,
  "sha256":     "9c1f2b7d…",
  "status":     "pending"
}
```

This is the **pending-artifact object**, used by every route below that returns a pending artifact. `store_id` + `uri` let the client build the retrieval URL for a pending artifact (e.g. to preview it before sending). They are informational only: the turn never reads refs from the client.

`status` is always `"pending"` in this response. Upload handling, in order. **No database transaction or lock is held while the client body is read or while the store is called**, so a slow or large upload never blocks turns or holds a pool connection:

1. **Read** — hash and buffer the body (`413` past `ARTIFACT_MAX_UPLOAD_MB`, before any store call); resolve `mime` (§ MIME resolution). No DB access.
2. **Pre-check (non-locking, fast fail)** — if a pending row for `(session_id, store_id, uri)` exists, return it (step 5 semantics); else if the session already has `ARTIFACT_MAX_PENDING` pending rows → `409`. Avoids a `Put` that would be refused anyway. Not authoritative.
3. **Store** — `adapter.Stat(uri)`; if it reports the blob exists, skip `Put`. If `Stat` errors, log and fall through to `Put` (current PR #53 behaviour, kept). `Put` failure → `502`. No DB transaction is open.
4. **Commit (short transaction)** — `BEGIN`; `SELECT pg_advisory_xact_lock(<table key>, hashtext(session_id::text))` (a per-table constant as the first key; serialises only uploads to the same session, never turns); **re-read the session row**: gone → `404`; BO `locked_at IS NOT NULL` → `409` (a dataset close may have landed during a slow upload); then steps 5–6; `COMMIT`. An FK violation on insert (session deleted concurrently) also maps to `404`.
5. **Dedup (authoritative)** — if a pending row for `(session_id, store_id, uri)` exists, return it (`201`, unchanged — its original `filename` and `mime` are kept even if this request's differ). The pending cap does not apply to a dedup hit. The partial unique index backstops this.
6. **Cap (authoritative) and insert** — if the session already has `ARTIFACT_MAX_PENDING` pending rows → `409`; otherwise insert the pending row and return it.

A `409` at step 6 after a successful `Put` leaves the blob in the store unreferenced — consistent with the no-GC stance and bounded by `ARTIFACT_MAX_UPLOAD_MB` per refused request.

**Lock compatibility.** The advisory lock is independent of row locks. The pending-row insert takes only the FK's `FOR KEY SHARE` on the session row, which is compatible with the `FOR KEY SHARE` taken by message inserts and with the BO `UPDATE chat_sessions SET updated_at` (`FOR NO KEY UPDATE`). Turns never take the advisory lock: a claim concurrent with an upload either sees the new row or leaves it pending for the next turn.

A re-upload of content already consumed in this session creates a new pending row.

| Status | Cause |
|---|---|
| `400` | No `file` field / malformed multipart / missing `user_id` |
| `404` | Agent not deployed, session missing, user mismatch, or agent mismatch |
| `409` | Session already has `ARTIFACT_MAX_PENDING` pending artifacts |
| `413` | Upload exceeds `ARTIFACT_MAX_UPLOAD_MB` (before any store call) |
| `502` | `adapter.Put` failed |

**`GET /deployed/{agent_id}/sessions/{session_id}/artifacts/{store_id}/{uri...}?user_id=X`** — retrieval. `{uri...}` is a Go 1.22 wildcard so `sha256/<hex>` passes intact.

Authorisation: preamble + a row exists in `deployed_session_artifacts` for `(session_id, store_id, uri)` (any origin, pending or consumed) + `store_id` is registered. Then `adapter.Stat(uri)`: not found → `410`. (`Sign` presigns offline and cannot detect a missing blob, so the `Stat` is required for `410` to be reachable.) Delivery per adapter `PreferredDelivery()`: `302 Location: <signed-url>` (TTL `ARTIFACT_SIGNED_URL_TTL_SECONDS`) or `200` with streamed body. Response headers — see § Retrieval response headers.

| Status | Cause |
|---|---|
| `400` | Missing `user_id`, or path not `<store_id>/<uri>` |
| `404` | Preamble failure, no matching row, or `store_id` not registered |
| `410` | `Stat` reports the blob missing (externally removed) |
| `502` | Other adapter error |

**Retrieval response headers (both surfaces).** Artifact bytes are user- or tool-supplied and served from an origin the client may render, so:

- **Bytes delivery (`200`):** `Content-Type: <row.mime>`, `Content-Length: <row.size_bytes>`, `X-Content-Type-Options: nosniff`, and `Content-Disposition: inline; filename*=UTF-8''<row.filename>` when `row.mime` is in the native-render set, `attachment; filename*=…` otherwise (filename omitted when the row has none).
- **Signed-URL delivery (`302`):** the signed URL carries the same `Content-Type` and `Content-Disposition` as response overrides (S3 `response-content-type` / `response-content-disposition` presign parameters), so the store serves them. This extends the adapter method to `Sign(ctx, uri, ttl, SignOptions{ContentType, ContentDisposition})`; kinds that cannot set response overrides ignore them. Per-request overrides are required because content-addressed blobs are shared by rows with different filenames.

**`GET /deployed/{agent_id}/sessions/{session_id}/pending-artifacts?user_id=X`** — list the session's queue.

`200` with `{ "pending_artifacts": [ <pending-artifact object>, … ] }`, ordered by `(created_at, id)` (the order they will be added to the next turn). `[]` when empty. Preamble failure → `404`; missing `user_id` → `400`.

**`DELETE /deployed/{agent_id}/sessions/{session_id}/pending-artifacts/{id}?user_id=X`** — remove one pending artifact so the next turn does not include it.

| Status | Cause |
|---|---|
| `204` | Row deleted |
| `400` | Missing `user_id` |
| `404` | Preamble failure, or no row with that `id` in this session |
| `409` | Row exists in this session but is already consumed (it is part of history and cannot be removed) |

Deletes the row only; the blob stays in the store (content-addressed blobs may be shared by other rows; blob GC is out of scope). After removal, retrieval of that ref in this session returns `404` unless another row for the same `(store_id, uri)` exists. Removing and re-uploading the same file creates a new pending row.

The existing `GET /deployed/{agent_id}/sessions/{session_id}?user_id=X` response is unchanged.

### REST — BO (existing paths, changed behaviour)

- **`POST /agent_versions/{version_id}/chat/{session_id}/artifacts`** — same preamble as today (`GetSession(session_id, version_id)`, `409` if `locked_at IS NOT NULL`). Now writes a pending `chat_session_artifacts` row and returns the pending-artifact object. This is additive over PR #53's `{store_id, uri, mime, size_bytes, sha256, filename}` — every existing field is kept with the same meaning; `id` and `status` are new. Adds the `409` pending-cap case and follows the same upload handling steps as deployed. Other statuses unchanged.
- **`GET /agent_versions/{version_id}/chat/{session_id}/artifacts/{store_id}/{uri...}`** — authorisation switches from the `chat_messages` JSONB scan (`SessionReferences`, which could never match because refs were not persisted) to a row lookup in `chat_session_artifacts`, followed by the same `Stat`-before-delivery step as deployed. Response headers per § Retrieval response headers (adds `Content-Length`, `nosniff`, `Content-Disposition`). Statuses unchanged.
- **`GET /agent_versions/{version_id}/chat/{session_id}/pending-artifacts`** and **`DELETE /agent_versions/{version_id}/chat/{session_id}/pending-artifacts/{id}`** — same contract as the deployed pair, with the BO preamble (`GetSession(session_id, version_id)`). `DELETE` on a session with `locked_at IS NOT NULL` → `409`, matching upload.
- **Chat view** (`GET /agent_versions/{v}/chat/{s}`) renders pending rows as chips, each with a remove control calling the `DELETE` route.

### REST — turn (both surfaces)

- `POST /deployed/{agent_id}/sessions/{session_id}/turn` and `POST /agent_versions/{v}/chat/{s}/turn`: input blocks of type `artifact_ref` are **ignored** (not rejected — older clients that still send them keep working; the blocks are simply dropped). Only `text` blocks are read from the body.
- **Validity (new rule — neither handler rejects empty input today).** After the existing session checks and before the SSE stream opens, the handler checks: no non-empty `text` block **and** no pending artifact for the session → `400` with the plain-text body `empty turn: no text and no pending artifacts` (error code `empty_turn`). A turn with no text but ≥1 pending artifact is accepted.
- **Race with `DELETE`.** A pending artifact removed between that check and the claim can leave a text-less turn with nothing to claim. `AppendUserTurn` then returns `ErrEmptyTurn` and rolls back (no message persisted); the orchestrator emits an in-band `RUN_ERROR` with code `empty_turn` and ends the stream.
- The claimed refs appear on the persisted user message after its text blocks, in `(created_at, id)` order.

### Store-layer writes

Both session stores gain:

```go
// AppendUserTurn persists the user message and claims every pending artifact on
// the session in one transaction. Claimed refs are appended to msg.Content after
// its existing blocks, in (created_at, id) order, and returned for the caller.
// Returns ErrEmptyTurn (and persists nothing) if msg has no non-empty text block
// and nothing was claimed.
AppendUserTurn(ctx context.Context, agentVersionID string, msg types.ChatMessage) ([]llm.ArtifactRefBlock, error)

// AppendToolMessage persists a tool-role message and one origin='tool_result' row
// per ref (message_id = msg.ID) in one transaction. refs may be empty.
AppendToolMessage(ctx context.Context, agentVersionID string, msg types.ChatMessage, refs []llm.ArtifactRefBlock) error
```

Claim SQL (inside the message-insert transaction):

```sql
UPDATE <table> SET message_id = $msg, updated_at = now()
WHERE session_id = $s AND origin = 'upload' AND message_id IS NULL
RETURNING id, store_id, uri, mime, size_bytes, sha256, filename, created_at
```

Rows are ordered by `(created_at, id)` in Go after `RETURNING`. Concurrent turns on one session: row locks make the second `UPDATE` wait and then match zero rows, so each pending artifact is added to exactly one message. If the turn fails after the transaction commits (LLM error, tool error), the refs are already in history and are seen on the next turn — nothing is lost or added twice.

The orchestrator calls `AppendUserTurn` in place of its current user-message `AppendMessages` call, and `AppendToolMessage` in place of its tool-role `AppendMessages` call. Because message and rows commit together, a ref is never in history without a row (which would make it unretrievable), and a row never exists without its message.

The orchestrator's user-input `artifact_ref` path is removed: `Orchestrator.Turn` input carries text blocks only, and user-role refs come solely from `AppendUserTurn`. (Verified: no internal caller — eval, training — drives `Orchestrator.Turn`.)

**Pending cap enforcement** is atomic via the per-session advisory lock in the upload's short commit transaction (§ REST — deployed, upload steps 4–6). The claim path does not take it.

### Rendering robustness

A consumed artifact stays in history and is re-rendered on every later turn, so a ref that makes rendering fail would fail every later turn of the session. Rules in `renderArtifactsForLLM` and the Anthropic renderer:

- **Any fetch error** from `RenderArtifactAsBlock` (other than `ErrUnsupported`) → the framework calls `adapter.Stat(uri)`:
  - `Stat` reports not found → text surrogate, `warn` log. Permanent condition; failing would brick the session. (Previously: turn failed.)
  - otherwise (blob exists, or `Stat` itself errors) → turn fails as today (`artifact_render`). Transient; the next turn retries.

  Using `Stat` rather than the signed fetch's HTTP status avoids misreading S3's `403 AccessDenied` for a missing key as a missing blob; § Assumptions requires store credentials under which a missing key is reported as not-found, and `Probe()` verifies it.
- **Provider limits, per block** — the renderer returns `ErrUnsupported` (→ text surrogate) when `size_bytes` exceeds the provider's documented per-block limit for that MIME. Anthropic limits are pinned in the renderer from the provider's documentation at implementation time.
- **Provider limits, per request (native-render budget)** — consumed refs are re-sent on every later turn, so their total grows with the session. `llm.MultimodalRenderer` gains `NativeRenderBudget() RenderBudget{MaxBytes int64, MaxBlocks int}`: the provider's documented whole-request media limits, pinned in the renderer at implementation time and set below the provider's hard request limit so text and tool payloads keep headroom (bytes are counted after base64 expansion, `ceil(size_bytes/3)*4`). `renderArtifactsForLLM` walks the request's refs **newest first** (by message order, then block order); each ref that would render natively is charged against the budget, and once the next ref would exceed `MaxBytes` or `MaxBlocks`, it and every older ref render as text surrogates. The result is deterministic per request, so accumulated history can never push a session into permanent failure. Refs are never dropped — over-budget refs still appear as surrogates.

  Walk order within one message is reversed too (last block first), so for uploads A then B on one user message, B is charged before A. Refs already downgraded to a text surrogate — by the per-block limit, by `ErrUnsupported`, or by a missing blob — are not charged.

  **Every LLM call renders a fresh copy.** Today the orchestrator writes the rendered list back into `req.Messages` (`orchestrator.go:252-256`), so from the second tool round on, earlier refs are already `InlineMediaBlock`s and escape the budget. New rule: `req.Messages` always holds the ref form (`artifact_ref` blocks, never `InlineMediaBlock`); each LLM call — every round of the tool loop — renders a fresh copy of that list, applies the budget to the whole copy, and sends it. The rendered copy is discarded after the call. This keeps the budget per request across tool rounds and also keeps rendered bytes out of anything that could be persisted.
- **Stored MIME is trusted by the renderer** because § MIME resolution guarantees native-render labels only on bytes that sniff (and, for images, decode) as that type.

### Sub-agents

Applies once `multiagent-feature` (child sessions + the `agent` tool) is implemented; whichever of the two specs lands second implements and tests these rules. It matches `multiagent-feature`'s own text-only decision.

- **Artifact scope is per session.** A child session has its own rows in the same table (`chat_session_artifacts` / `deployed_session_artifacts` — a child is a row in `chat_sessions` / `deployed_sessions`). There is no inheritance between root and child in either direction.
- **The `agent` tool carries text only.** No `artifacts` argument, no refs in its result; the root's `agent` tool-result message never contains `artifact_ref` blocks.
- **Child tool results** are normalised and recorded as `tool_result` rows on the child session; the child's LLM sees them. The root's LLM and the end user do not.
- **No `ARTIFACT_REF` for child tool results** is forwarded to the root's stream (unlike the child's `TOOL_CALL_*` events, which `multiagent-feature` forwards tagged with `child_session_id`). A forwarded event would point at a ref the user cannot read.
- **Child sessions are not addressable by artifact routes.** Upload, pending list/remove and retrieval on both surfaces return `404` for a session with `parent_session_id IS NOT NULL`. Children never have pending artifacts (they have no user turns). The BO read-only child transcript renders child `artifact_ref` blocks as filename labels without a link.
- **LLM-written refs are never trusted.** `artifact_ref` blocks enter a message only via `AppendUserTurn` (claimed rows) or `AppendToolMessage` (normalised tool results). A `{store_id, uri}` the LLM writes into `tool_use` input is opaque JSON passed to the tool unchanged; the framework never resolves, renders or authorises it, so prompt injection cannot pull another session's blob into a message.

### Stream event — `ARTIFACT_REF`

Internal: `transport.ArtifactRefEvent{ToolCallID, StoreID, URI, MIME, SizeBytes, Sha256, Filename}`. Not emitted for user uploads (the uploading client already knows them) or for child-session tool results (§ Sub-agents).

**Ordering and no bytes on the stream.** Today the orchestrator sends `ToolResultEvent` with the raw tool output (including base64 `ImageContent`) *before* normalisation runs. This spec reorders it:

1. per tool call: normalise, then send `ToolResultEvent` carrying the **normalised remainder** (inline media removed);
2. once every tool call of the round has finished, `AppendToolMessage` commits the tool-role message and its rows;
3. **after that commit**, send one `ArtifactRefEvent` per ref, in tool-call order then item order.

So a client receiving `ARTIFACT_REF` can always resolve it immediately. If the commit fails, no `ARTIFACT_REF` is sent for that round and the turn fails in-band as today for a persistence error.

**Normalisation failure never falls back to raw output.** Today an upload failure inside `normaliseToolResult` makes the orchestrator log and pass the raw output through, which would put base64 on SSE and in Postgres. New rule, per inline-media item (`ImageContent`, `EmbeddedResource` with `blob`/`text`):

- upload succeeds → the item becomes an `artifact_ref` block (as above);
- upload fails (store error, or the item's base64 does not decode) → the item is **replaced in the remainder** by a text item `{"type":"text","text":"[artifact unavailable: <mime>, <size>]"}`, a `warn` is logged with the tool name and error, and no row is written. `<mime>` is the resolved MIME (§ MIME resolution) when bytes were decoded, otherwise the declared MIME; `<size>` is the human-readable decoded size, or `unknown` when decoding failed. Other items in the same result are processed independently. A failed `EmbeddedResource` with inline `text` loses that text for the model — accepted; the `warn` log records it.
- **Tool results flagged as errors are normalised too.** Today normalisation only runs when `!res.IsError` (`orchestrator.go:373`), and MCP `isError: true` results (e.g. a screenshot of a failed state) would pass through raw. The guard is removed: every tool result is normalised; `is_error` is preserved as reported by the tool.
- **No raw fallback on re-serialisation.** If re-marshalling the normalised remainder fails (`artifacts.go` currently returns the raw payload), the tool result becomes `{"content":[{"type":"text","text":"[tool result unavailable]"}]}` with `is_error: true`, and a `warn` is logged. Refs already uploaded for that result are still recorded and emitted.

`is_error` is otherwise unchanged by normalisation (the tool's own verdict stands). With the artifact registry enabled, artifact bytes therefore never travel over SSE and never reach Postgres, upholding `ddd/contexts/deployed-runtime.md § Invariants` and `ddd/contexts/artifacts.md § Invariants`.

**Registry disabled.** Normalisation does not run; `ToolResultEvent` and the persisted `tool_result` carry the raw output, including any base64, exactly as today. The "no bytes over SSE / in Postgres" invariants hold only when the registry is enabled — declared as an arch delta, not changed by this spec.

**Payload evolution.** This is stream framing, not a domain event (the arch declares none), so `events.md` `schemaVersion`/`eventId` rules do not apply. The `value` payload only ever changes additively; fields are never removed or repurposed.

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

**Registry disabled** (no store groups): no artifact or pending-artifact routes on either surface, turns behave as today except that the empty-turn `400` rule still applies, no `ARTIFACT_REF` events.

### Deploy config

- Helm chart: `artifacts.enabled`, `artifacts.stores.<ID>.{kind, endpoint, bucket, region, pathStyle}`, `artifacts.stores.<ID>.existingSecret` (keys `accessKey`, `secretKey`), `artifacts.default`, `artifacts.maxUploadMB`, `artifacts.maxPending`, `artifacts.signedUrlTTLSeconds` → rendered as the env vars above. Disabled by default.
- `docker-compose.yml`: `artifacts` profile adding a MinIO service + bucket init and the env for a `MAIN` store.

### Arch deltas for `/arch-review` (post-approval sync — not edited by this spec)

- `ddd/contexts/artifacts.md § Published surface` and `modularity/open-bbcd/artifacts/README.md`: replace stale `/chat-sessions/{id}/artifacts` and top-level `/artifacts/{store_id}/{uri}` paths with the nested per-surface routes; replace "client embeds the returned ref in the outgoing turn body" with the staged pending → consumed-on-next-turn model; session-scope enforced via the per-context artifact table, not a JSONB scan.
- `modularity/open-bbcd/deployed-runtime/README.md` and `ddd/contexts/deployed-runtime.md`: own `deployed_session_artifacts`; publish the nested retrieval route and the `pending-artifacts` list/remove routes.
- `modularity/open-bbcd/feedback-datasets/README.md` and `ddd/contexts/feedback-datasets.md`: own `chat_session_artifacts`; publish the BO `pending-artifacts` list/remove routes.
- `c4/integrations.md § AG-UI ARTIFACT_REF`: carried as AG-UI `CUSTOM` event `name: "ARTIFACT_REF"`; correct the "SDKs ignore unknown events" assumption.
- `ddd/access-model.md`: deployed retrieval path; end user may list and remove own session's pending artifacts; BO and deployed read allow-list = own session's artifact rows.
- `constraints.md` / `c4/deployment.md`: `ARTIFACT_MAX_PENDING`.
- `ddd/contexts/artifacts.md § Aggregates & entities`: the "no artifacts table", "not a standalone entity" and "source of truth is the union of `artifact_ref` blocks" statements no longer hold — session artifacts have identity and a pending → consumed lifecycle (stored in the owning contexts' tables, not in `artifacts`). Also the § Invariants MCP normalisation line (URI-only `EmbeddedResource` is not normalised) and the MIME-resolution rule.
- `c4/containers.md` (Postgres data-ownership line) and `modularity/open-bbcd/README.md` (L1 owned tables): add both tables.
- `bizbok/information-map.md`: Artifact row + the new *Session artifact* / *Pending artifact* concepts. `glossary.md`: add those two terms; fix *Content block* ("pointer to an `artifact_stores` row" — there is no such table); fix *Artifact* ("four legs" — agent emission was dropped).
- `c4/data-flows.md § Multimodal chat with artifacts` and `bizbok/value-streams.md § Multimodal chat with artifacts`: turn content is text only; pending artifacts are added server-side.
- Stale route paths: `nfrs.md § Security`, `c4/deployment.md` threat-model row "Artifact ref leak", `constraints.md` `ARTIFACT_MAX_UPLOAD_MB` bullet, `ddd/contexts/feedback-datasets.md` invariant naming `/chat-sessions/{id}/artifacts`.
- `ddd/contexts/deployed-runtime.md § Published surface` ("older SDKs see it as an unknown event") and `modularity/open-bbcd/deployed-runtime/README.md` (upload response, `ARTIFACT_REF` wording).
- `c4/integrations.md` artifact-store adapter contract: `Sign` gains `SignOptions` (response content-type / disposition overrides); `Probe()` checks missing-key detection.
- **Sub-agent artifact scope (narrows `multiagent-tools`):** `nfrs.md § Security › Sub-agent trust model` ("refs may flow between parent and child in both directions" → per-session scope, no flow); `ddd/contexts/feedback-datasets.md` child session "inherits the root's … artifact scope" → does not; `c4/integrations.md § Agent tool` signature loses `artifacts?` / `artifacts`; `glossary.md` *Agent tool* and *Sub-agent* rows; `assumptions.md` agent-tool bullet ("+ optional `artifact_ref`s", "any `artifact_ref`s as the tool result"); `ddd/contexts/deployed-runtime.md` — child `ARTIFACT_REF` events are not forwarded.
- **Bytes invariants scoped to an enabled registry:** `ddd/contexts/artifacts.md § Invariants` ("Bytes never live in Postgres") and `ddd/contexts/deployed-runtime.md § Invariants` (no bytes over SSE) hold when the artifact registry is enabled; with it disabled, raw tool output (possibly carrying base64 media) is streamed and persisted as today.
- `ddd/contexts/feedback-datasets.md` / `bizbok/value-streams.md` (eval replay): replay of artifact-bearing sessions is text-only until an artifact-aware replay follow-up.
- **Inner-ref pointers:** `ddd/contexts/artifacts.md § Published surface` (the `{store_id, uri}` pointer inside `tool_input` / `tool_result`, "leg 4") and the *agent-to-tool argument* leg in `modularity/open-bbcd/artifacts/README.md`: the framework does not resolve them; tool inputs are opaque.
- `bizbok/capabilities.md § deployed-runtime-artifacts` ("End user uploads artifacts alongside a user turn") → staged pending artifacts consumed by the next turn; `c4/containers.md` ("store-config CRUD" — there is none; registry is env-only).

## Acceptance criteria

Integration tests run against MinIO (compose `artifacts` profile) unless marked unit.

### Upload (both surfaces)

- Deployed upload with valid `file`, matching `user_id` returns `201` with `{id, store_id, uri, filename, mime, size_bytes, sha256, status:"pending"}`; `store_id` equals `ARTIFACT_STORE_DEFAULT`, `uri` equals `sha256/<sha256>`; `sha256` equals the sha256 of the bytes; a `deployed_session_artifacts` row exists with `origin='upload'`, `message_id IS NULL`.
- BO upload returns the same shape and writes a `chat_session_artifacts` row; every field of PR #53's response is present with the same value it had before.
- Deployed and BO GET of `/artifacts/{store_id}/{uri}` built from an upload response succeeds while the artifact is still pending.
- Deployed upload with a `user_id` that does not own the session → `404`; with the session under a different `agent_id` → `404`; for an agent with no DEPLOYED version → `404`; missing `user_id` → `400`.
- BO upload on a session with `locked_at IS NOT NULL` → `409`.
- The `ARTIFACT_MAX_PENDING + 1`-th pending artifact on a session → `409`; after a turn consumes them, uploads succeed again.
- With the session at the cap, uploading bytes identical to an existing pending row returns that row (`201`), not `409`.
- `ARTIFACT_MAX_PENDING` concurrent uploads of distinct files plus one more, all in parallel: exactly `ARTIFACT_MAX_PENDING` rows exist afterwards and one request got `409` (integration, Postgres).
- **Uploads never block turns:** with an upload of a large file mid-body (client stalled before finishing the request body) on a session, a turn on the same session runs to completion and persists its user, assistant and tool messages without waiting; no DB connection is held by the stalled upload (integration, pool-usage assertion).
- With the session at the cap, an upload of new content is refused with `409` at the pre-check without calling `adapter.Put` (mock spy, unit).
- With `adapter.Put` stubbed to block, a concurrent turn on the same session completes and the blocked upload holds no DB pool connection (integration).
- A BO session locked (dataset close) while an upload is between its pre-check and its commit: the upload gets `409` and no row is inserted (integration, blocking `Put` stub).
- A session deleted while an upload is in flight: the upload gets `404` (integration).
- A dedup hit with a different multipart filename returns the existing row's `filename`.
- When `adapter.Stat` errors during upload, `Put` is still attempted and a successful `Put` yields `201`.
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

- A consumed ref whose blob was removed from the bucket: the next turn succeeds, the LLM request carries the text surrogate for it, and a `warn` is logged (MinIO, signed-URL adapter).
- A store network error during rendering fails the turn with `artifact_render`; the next turn with the store healthy renders natively.
- An `image/png` ref above the pinned Anthropic image limit is rendered as a text surrogate (unit).
- With a stub budget `{MaxBlocks: 2}` and three consumed image refs across history, the LLM request carries the two newest as native image blocks and the oldest as a text surrogate (unit).
- A turn with three tool rounds each returning an image, stub budget `{MaxBlocks: 2}`: the final LLM call carries exactly two native image blocks (the two newest) and `req.Messages` holds no `InlineMediaBlock` between rounds (unit).
- Two uploads A then B on one user message with `{MaxBlocks: 1}`: B is native, A is a surrogate (unit).
- With a stub budget `{MaxBytes: N}`, refs are charged at base64-expanded size and the first ref (newest first) that would exceed `N` and every older ref render as surrogates (unit).
- A session whose consumed refs total more than the provider's request limit still completes turns (integration with a stub provider enforcing the limit).

### Turn / claim

- After two uploads (A then B) and a turn with text "summarise", the persisted user message content is `[text, artifact_ref(A), artifact_ref(B)]`; both rows have `message_id` = that message's id.
- Reloading the session history (`LoadMessages` → `parseBlocks`) returns the `artifact_ref` blocks with all fields intact, on both surfaces; the next turn's LLM request includes them.
- `blocksToJSON` given an `InlineMediaBlock` returns an error (unit).
- The `artifact_ref` blocks' `mime`/`size_bytes`/`sha256`/`filename` equal the row values.
- A turn body containing an `artifact_ref` block (any `store_id`/`uri`) does not add that block to the persisted message (both surfaces).
- A turn with empty text and one pending artifact is accepted and the message contains only the `artifact_ref` block.
- A turn with empty text and no pending artifacts → `400` with body `empty turn: no text and no pending artifacts`, before any SSE bytes; nothing persisted (both surfaces).
- A text-less turn whose only pending artifact is deleted between the validity check and the claim: the stream carries `RUN_ERROR` with code `empty_turn`, and no user message is persisted (unit, with a store stub that deletes between check and claim).
- A turn with no pending artifacts persists exactly as today (no behaviour change).
- Two concurrent turns on one session with one pending artifact: exactly one persisted user message carries the ref (integration, Postgres, parallel transactions).
- An LLM failure after the user message is persisted leaves the rows consumed and the refs in history; the next turn's LLM request includes them.
- With the Anthropic provider, a claimed `image/png` ref is sent as a native image block and `application/pdf` as a document block on the deployed surface (parity with BO).

### Tool-result artifacts + event

- A deployed turn whose MCP tool returns `ImageContent` produces: an `artifact_ref` on the tool-role message, a `deployed_session_artifacts` row with `origin='tool_result'` and `message_id` = that message, and an AG-UI `CUSTOM` event with `name:"ARTIFACT_REF"` emitted after the corresponding tool-result event, carrying the matching `toolCallId`, `storeId`, `uri`, `mime`, `sizeBytes`, `sha256`.
- The `TOOL_CALL_RESULT` event for that call contains no base64 image data (the `ImageContent` item is absent from its `content`).
- If the row insert fails, the tool-role message is not persisted either (single transaction; unit with a failing insert).
- When `adapter.Put` fails while normalising an `ImageContent` item: the persisted `tool_result` and the `TOOL_CALL_RESULT` event contain `[artifact unavailable: image/png, <size>]` in place of the item and no base64; no row is written; no `ARTIFACT_REF` is emitted; the turn continues (unit).
- A tool result with two images where only the second upload fails yields one `artifact_ref` and one `[artifact unavailable: …]` text item (unit).
- With the registry disabled, a tool result carrying `ImageContent` is streamed and persisted unchanged (unit — existing behaviour pinned).
- An MCP result with `isError: true` carrying `ImageContent` is normalised: no base64 in `TOOL_CALL_RESULT` or the persisted `tool_result`, `is_error` stays `true`, a row is written (unit).
- `ARTIFACT_REF` events for a round are sent only after its `AppendToolMessage` commits; with the commit failing, none is sent (unit, event-order assertion).
- A GET issued for the ref immediately on receiving `ARTIFACT_REF` succeeds (integration).
- Same on BO.
- JSONL transport emits `{"type":"artifact_ref", …}` with the same fields (unit).
- No `ARTIFACT_REF` event is emitted for claimed user uploads.

### Retrieval

- Deployed GET for a ref uploaded to the session (pending or consumed) returns `302` to a URL that serves the bytes; BO GET likewise.
- GET for a tool-result ref recorded on the session succeeds.
- GET for a `(store_id, uri)` that exists in the store but has no row for this session → `404`, including when the same blob has a row in another session.
- GET with wrong `user_id`, wrong `agent_id`, or unregistered `store_id` → `404`; missing `user_id` → `400`.
- GET after the blob is removed from the bucket → `410` (MinIO, signed-URL adapter — via the `Stat` step).
- `Probe()` against credentials without list permission fails boot naming the permission (unit, stubbed `403` on `Stat` of a missing key).
- Bytes-mode adapter (mock, unit): `200` with `Content-Type` = row `mime` and `Content-Length` = row `size_bytes`.

### Pending-artifact queue

- `GET …/pending-artifacts` lists unconsumed uploads in `(created_at, id)` order as pending-artifact objects; `[]` after a turn consumes them (both surfaces).
- `DELETE …/pending-artifacts/{id}` on a pending row → `204`; the next turn does not include it; the list no longer shows it; the blob is still in the bucket.
- `DELETE` on a consumed row → `409`; on an `id` from another session, a wrong `user_id`, or a wrong `agent_id` → `404`.
- `DELETE` then re-upload of the same file creates a new pending row with a new `id`.
- A removed pending artifact no longer counts toward `ARTIFACT_MAX_PENDING`.
- BO `DELETE` on a locked session → `409`.
- BO chat view renders a chip per pending artifact on page load; its remove control deletes it.

### Sub-agents (once `multiagent-feature` is implemented)

- A child session whose MCP tool returns `ImageContent`: the row is written on the child session; the root's stream carries the child's tagged `TOOL_CALL_*` events but no `ARTIFACT_REF`; the root's `agent` tool-result message contains no `artifact_ref`.
- Upload, pending list/remove and retrieval addressed at a child session id → `404` (both surfaces).
- An LLM `tool_use` whose input contains `{"store_id":"MAIN","uri":"sha256/<hex of a blob in another session>"}` results in no row and no `artifact_ref` in any message of this session; retrieval of that ref here → `404`.

### Eval export (out-of-scope replay pinned)

- Exporting a CLOSED dataset that contains a session with `artifact_ref` blocks succeeds, and the blocks appear unchanged in the export (unit, `internal/eval/export.go`).

### Lifecycle, config, feature-off

- Deleting a deployed session removes its `deployed_session_artifacts` rows (cascade); blobs remain in the store.
- `ARTIFACT_MAX_PENDING=0` or non-integer fails boot naming the var; unset defaults to `10`.
- With no `ARTIFACT_STORE_*` groups, deployed artifact and pending-artifact routes return `404` (unregistered), turns are unchanged, and no `ARTIFACT_REF` events are emitted.
- Helm chart with `artifacts.enabled=true` renders all artifact env vars with credentials sourced from the named Secret (`helm template` snapshot test).

## Risks & assumptions

### Assumptions

- **The operator gateway rewrites `user_id` to the verified identity** on every deployed route, including the new upload and retrieval routes. Same trust model as existing deployed routes; the daemon does no authentication.
- **All pending artifacts going with the next turn is the right UX.** Users upload, then send; everything pending goes with the next message. No per-file choice at send time.
- **`ARTIFACT_MAX_PENDING=10` is a sensible default** for LLM request size alongside `ARTIFACT_MAX_UPLOAD_MB`; deployers tune it.
- **The AG-UI encoder can emit `CUSTOM` events** without a library change: `internal/transport/agui` uses the community Go SDK (`pkg/core/events`, `pkg/encoding/sse`), which provides `events.NewCustomEvent`.
- **No internal caller other than the HTTP turn handlers drives `Orchestrator.Turn`** (verified for eval/training), so removing its user-input ref path breaks nothing.
- **Store credentials report a missing key as not-found.** For `s3_compatible`, the credentials include `s3:ListBucket` on the bucket, without which S3 returns `403` for a missing key. `Probe()` gains a step: `Stat` on a random non-existent key must return not-found; otherwise boot fails with an error naming the permission. MinIO and AWS S3 both behave this way with list permission.

### Risks

- **Turn-body contract change on BO.** Refs sent in the turn body are now ignored. A client built against PR #53's "upload, then embed the returned ref" flow keeps working unchanged: its upload creates a pending artifact, the next turn adds it server-side, and the embedded ref it also sends is dropped — same persisted result. The upload response is additive. What such a client cannot do any more is attach a ref it did not upload to this session, which is the intended restriction. Mitigation: release notes for the downstream project.
- **Blobs uploaded under PR #53 become permanently unreferenced.** PR #53 never persisted refs, so those blobs were already unreachable after their turn; no row or message will ever point at them. Deployer removes them with store lifecycle rules if needed. Datasets closed before this ships contain no refs to preserve.
- **`filename` may carry PII** (user-chosen file names). Classified as user-provided free text; not logged by the upload handler; removed with the session cascade. It also appears inside `artifact_ref` blocks in dataset exports (`export.yaml`) and so reaches the aikdm simulator/judge LLM with the rest of the transcript — same exposure as the message text itself.
- **Artifact-only user turns in eval replay.** User turns may now carry no text (only `artifact_ref`s). In replay the simulator sees those turns as JSON; `aikdm/eval/runner.py` ends a simulated conversation when the simulator returns empty content, so such datasets may replay shorter than the reference. Accepted until the artifact-aware replay follow-up.
- **Signed URLs in the `302` are bearer URLs for their TTL.** Unchanged from `chat-artifacts`; tune `ARTIFACT_SIGNED_URL_TTL_SECONDS`.
- **Clients unaware of `CUSTOM ARTIFACT_REF`** see tool results as text only. Acceptable degradation; the ref is still in persisted history and visible via `GET session`.
- **Empty turns are newly refused.** A client that today posts a turn with no text (e.g. a bare "continue") now gets `400`. None is known; noted in release notes.
- **`Probe()` is stricter.** Deployments whose store credentials lack list permission fail boot after upgrade. Intended — without it, missing blobs are indistinguishable from auth failures. Noted in release notes.
- **Sub-agent artifact scope diverges from the merged arch** until `/arch-review` syncs it (listed in § Arch deltas).
- **Removing a pending artifact leaves its blob** in the store. Consistent with the no-GC stance; bounded by the upload cap.
- **Provider rejects a request despite the checks** (e.g. a PDF over the provider's page limit, an encrypted PDF, an image the provider decodes differently). Because consumed refs are re-sent every turn, every later turn of that session fails. Mitigated by MIME resolution, the image structural check, per-block limits and the per-request native-render budget; residual risk accepted for this phase.
- **Older artifacts degrade to surrogates in long sessions.** Once the native-render budget is spent on newer refs, the model sees older files only as `[Attachment: …]` labels. Intended trade-off; the budget values are tuned at implementation. Follow-up: record a per-ref render-failure marker so a ref that caused a provider `4xx` falls back to a text surrogate.
- **MIME resolution changes labels on write.** A file declared `image/png` that doesn't decode is stored and served as `application/octet-stream`; the user gets it as a download rather than inline. Intended.
- **Pending rows on sessions that never get another turn** accumulate as orphan rows and blobs. Bounded by `ARTIFACT_MAX_PENDING` per session; removed on session delete.

## Cross-references

- Parent feature: `docs/superpowers/specs/2026-09-28-chat-artifacts-design.md` (BO, OpenBBC PR #53).
- Arch-of-record: `docs/architecture/logs/2026-09-28-artifact-support/README.md`; `docs/architecture/current/ddd/contexts/artifacts.md`; `docs/architecture/current/ddd/access-model.md`; `docs/architecture/current/c4/integrations.md § AG-UI ARTIFACT_REF`.
- Modularity: `docs/architecture/current/modularity/open-bbcd/{deployed-runtime,feedback-datasets,artifacts}/README.md`.
- Conventions: `docs/conventions/persistence.md` (per-context ownership, additive migrations, PII); `docs/conventions/api.md` (deviations stated in § REST conventions).
- Related spec: `spec/multiagent-feature` branch, `docs/superpowers/specs/2026-09-30-multiagent-feature-design.md` (text-only agent tool).
- Follow-ups: artifact handling strategy (text-like MIME rendering, pluggable fallback); per-provider native multimodal; `chat-artifacts-gc`.
