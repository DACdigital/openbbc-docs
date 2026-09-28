# chat-artifacts — file exchange in the backoffice test-chat surface

**Date:** 2026-09-28
**Status:** Draft — pending `/spec-review`
**Modularity attachment:** `docs/architecture/current/modularity/open-bbcd/feedback-datasets/` (L2), with sibling `docs/architecture/current/modularity/open-bbcd/artifacts/` (L2) in scope for the shared adapter substrate.
**Arch-of-record:** `chat-artifacts` L2 capability under `bizbok/capabilities.md § Feedback & dataset curation`, `artifact-store-adapter` L2 capability under `bizbok/capabilities.md § Artifact management`. Both merged in `docs/architecture/logs/2026-09-28-artifact-support/` (PR #5).

## Business value / Why

A downstream project consuming OpenBBC repeatedly asks for file exchange inside chat sessions — users attaching input material (documents, images, screenshots) and MCP tool results carrying files back. Today `chat_messages.content` is text-only.

Building this as **framework substrate** — one canonical content-block model + one pluggable storage adapter — prevents N bespoke per-deployer implementations diverging on data shape, wire semantics, and lifecycle behaviour. The merged `artifact-support` arch PR established the shape; this spec makes it real inside the backoffice test-chat surface, which is where new content shapes are exercised first before they graduate to the deployed runtime.

Grounds the value in three concrete outcomes for BO chat:

- **Admin attaches files to a test-chat turn** — the LLM reasons about them natively when the provider supports the MIME (Anthropic image/PDF today), falls back to a text-surrogate description when it doesn't.
- **MCP tool results carrying `ImageContent` or inline `EmbeddedResource`** are normalised into `artifact_ref` blocks on the tool-role message — same content-block shape as user uploads, regardless of upstream.
- **Dataset close-draft continues to snapshot sessions deterministically for eval replay**, now with `artifact_ref` blocks frozen alongside text. The arch invariant "refs stay resolvable while any locked session references them" is upheld at the adapter-wrapper layer.

## Change level

**C2.** The spec changes:

- The content-block shape of `chat_messages.content` JSONB (data-contract change; even with read-time normalisation for legacy rows, the write shape is new).
- The published REST surface of `open-bbcd` (new `POST /chat-sessions/{id}/artifacts`, new `GET /artifacts/{path}`).
- The `open-bbcd` boot-time contract (new env-var group + boot-fail semantics on missing default or failed probe).
- The LLM provider adapter interface (adds `RenderArtifactAsBlock` method with `ErrUnsupported` fallback).
- The framework's ACL to external infrastructure (new outbound calls to a deployer-provided Object store, new credential class in env).
- The session-scope authorisation surface (new session-scoped read route with its own 404 semantics).

C2 by definition per `docs/process/README.md`: "Never route through C0 if the change touches DB, API, events, security, tenant isolation, or an upgrade." This spec touches DB content shape, API, security, and session scoping. `/spec-review` runs as the unified gate (structure + naming + architecture + contracts).

## Scope

### In scope

- **Surface:** backoffice test-chat only — routes under `/agent_versions/{v}/chat[/{s}/*]` plus the two new artifact routes below.
- **Three legs of the artifact model:**
  - **User upload** — admin attaches a file with a user-role turn via a separate multipart POST; the returned ref is embedded in the outgoing turn body.
  - **MCP tool result → assistant** — tool result carrying `ImageContent` or inline (`blob`/`text`) `EmbeddedResource` is normalised into an `artifact_ref` content-block on the tool-role message.
  - **Agent-to-tool argument** — the LLM's tool-call arguments may reference an existing `artifact_ref` by including `{store_id, uri}` inside the tool_input payload.
- **`artifact-store-adapter` implementation** for the `s3_compatible` kind (covers AWS S3, MinIO, GCS with HMAC, R2, B2, and any other S3-API endpoint).
- **Env-driven registry hydration** at `open-bbcd` boot (`ARTIFACT_STORE_<ID>_*` groups + `ARTIFACT_STORE_DEFAULT`), boot fails if the default's `Probe()` fails, default is unset while artifact routes are enabled, or any group is malformed.
- **Read-time normalisation of legacy `chat_messages.content` rows** — no data migration. The repo layer detects the legacy shape on load and wraps into a synthetic `[{"type":"text","text":"..."}]` block list.
- **Session-scoped retrieval** — `GET /artifacts/{path}?session_id=<id>` with wildcard-path matching; framework verifies the ref is referenced by at least one message in the session via a JSONB scan, returns `404` on mismatch. The ref alone is not a bearer capability.
- **Adapter-picked delivery mode** — each adapter kind declares `PreferredDelivery() → Bytes | SignedURL`. `s3_compatible` returns `SignedURL` (offloads bandwidth to the store via presigned GETs). Retrieval route returns `200` with bytes or `302` with a `Location` header accordingly.
- **LLM adapter method for multimodal rendering** — `RenderArtifactAsBlock(ctx, ref, store) → ProviderContentBlock | ErrUnsupported`. Framework substitutes the text surrogate `[Attachment: <filename> (<mime>, <size>)]` on `ErrUnsupported`. Anthropic ships coverage for `image/{png,jpeg,gif,webp}` and `application/pdf`; other providers return `ErrUnsupported` until their own follow-on adapter work.
- **Dataset close-draft respects the "refs stay resolvable" invariant** — the framework's `adapter.Delete` wrapper refuses to remove any `(store_id, uri)` referenced by a message in a session whose `locked_at IS NOT NULL`.
- **`ARTIFACT_MAX_UPLOAD_MB` upload cap enforcement** at the multipart boundary — `413` before any store call is made.
- **Content-addressable URIs** — framework hashes the upload stream and constructs `uri = sha256/<hex>`; identical content deduplicates transparently (pre-check via `adapter.Stat`).
- **Optional `filename` metadata field on `artifact_ref` blocks** — user-facing label the UI renders; never used to construct storage URIs.

### Out of scope

- **Deployed runtime** — `deployed-runtime-artifacts` L2 is a separate follow-up spec. It needs the AG-UI `ARTIFACT_REF` event-type extension for outbound streaming and a different auth surface (session-scope via trusted-`user_id` from the operator gateway). Not shipped in this phase.
- **Leg 3 "agent emission"** — dropped. Only tools generate artifacts back to the assistant. The arch will be amended separately to remove leg 3 from the four-leg framing.
- **URI-only `EmbeddedResource` normalisation** — passes through as opaque data (not artifact-normalised); if a deployer needs URI ingestion later, that's a follow-on spec.
- **Blob lifecycle / garbage collection** — no framework-side cleanup on session or draft delete. Deployer handles orphan blobs via native store tools (S3 lifecycle rules, MinIO retention). `/new-spec chat-artifacts-gc` is the future extension if a real deployer complains.
- **Runtime CRUD over stores** — env-only per the merged arch amendment. No `POST /artifact-stores/*` routes, no BO UI for store management, no runtime reconfiguration. Store changes require a redeploy.
- **Adapter kinds beyond `s3_compatible`** — each additional kind (filesystem, GCS-native, Azure Blob) is code + tests, out of this spec.
- **Multi-provider LLM native multimodal support** — Anthropic ships in this spec; other providers (OpenAI, Gemini) return `ErrUnsupported` and use the text-surrogate fallback until per-provider adapter work.
- **Backoffice UI polish** — drag-and-drop targets, in-transcript thumbnails, previews, download buttons. The REST surface ships; UI rendering is a downstream concern outside the L2 scope.
- **Cross-user artifact reuse / discovery** — artifacts are session-scoped by construction; there is no "artifact library" browse UI or ref reuse across sessions.

## Contracts

### Env-var scheme

Registry hydrated once at `open-bbcd` boot. One group per store `<ID>` (uppercase alphanumeric + underscore, must be stable across deploys — the `<ID>` becomes the `store_id` value embedded in every `artifact_ref` block).

| Variable | Applies to | Meaning |
|---|---|---|
| `ARTIFACT_STORE_<ID>_KIND=s3_compatible` | always | Adapter kind. Phase 1 accepts only `s3_compatible`. |
| `ARTIFACT_STORE_<ID>_ENDPOINT` | `s3_compatible` | e.g. `https://s3.eu-west-1.amazonaws.com` |
| `ARTIFACT_STORE_<ID>_BUCKET` | `s3_compatible` | required |
| `ARTIFACT_STORE_<ID>_REGION` | `s3_compatible` | required |
| `ARTIFACT_STORE_<ID>_ACCESS_KEY` | `s3_compatible` | secret |
| `ARTIFACT_STORE_<ID>_SECRET_KEY` | `s3_compatible` | secret |
| `ARTIFACT_STORE_<ID>_PATH_STYLE=true` | `s3_compatible` | optional; default `false`; set `true` for MinIO |
| `ARTIFACT_STORE_DEFAULT=<ID>` | global | nominated write target |
| `ARTIFACT_MAX_UPLOAD_MB=<int>` | global | per-artifact upload cap; required if artifact routes are wired |
| `ARTIFACT_SIGNED_URL_TTL_SECONDS=<int>` | global | optional; default `300`; TTL for presigned URLs |

**Boot semantics:**
1. Parse env → collect one group per `<ID>` → construct one adapter per group.
2. Call `Probe()` on the default store (bounded retry: 3 attempts, exponential backoff).
3. Fail boot with a clear error if:
   - `ARTIFACT_STORE_DEFAULT` is unset when the chat-artifacts routes are compiled in.
   - The default store's `Probe()` fails after retries.
   - Any store group is malformed (`_KIND` set but a required sub-var is missing).
   - Two stores declare the same `<ID>` (env-var parsing catches this by de-dup).

### REST — upload

**`POST /chat-sessions/{session_id}/artifacts`** — `multipart/form-data`; single file field named `file`.

**Success (`201 Created`)** — response body:
```json
{
  "store_id":   "MAIN",
  "uri":        "sha256/9c1f2b7d...",
  "mime":       "application/pdf",
  "size_bytes": 245678,
  "sha256":     "9c1f2b7d..."
}
```

The client embeds this into the outgoing turn body as an `artifact_ref` content-block (optionally adding a per-ref `filename` for display).

**Error responses:**

| Status | Cause |
|---|---|
| `400` | No `file` field / malformed multipart |
| `404` | Session not found |
| `409` | Session is dataset-locked (`chat_sessions.locked_at IS NOT NULL`) — closed dataset versions may not accept new artifacts |
| `413` | Upload size exceeds `ARTIFACT_MAX_UPLOAD_MB` |
| `502` | Upstream store error — `adapter.Put` or `adapter.Stat` returned error |

**Dedup:** URI is content-addressable (`sha256/<hex>`). Framework hashes the incoming stream (buffered to memory, bounded by `ARTIFACT_MAX_UPLOAD_MB`), calls `adapter.Stat(uri)` first — on hit the upload is short-circuited and the existing ref is returned without re-upload. Same content uploaded twice by different admins produces one blob; the session-scope enforcement on reads prevents ref-guessing from succeeding cross-session.

### REST — retrieval

**`GET /artifacts/{path}?session_id=<id>`** — single wildcard path segment; the client constructs `{path}` by joining the ref JSON's `store_id` and `uri` fields with `/`:

```
GET /artifacts/MAIN/sha256/9c1f2b7d...?session_id=abc123
      ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
                {path}
```

**Server-side dispatch:** split `{path}` on the first `/` — prefix is `store_id`, remainder is store-local `uri`. Router matches with a wildcard capture on the segment after `/artifacts/`; concrete routing mechanism follows the existing HTTP mux used elsewhere in `open-bbcd` (verified at implementation time).

**Session-scope authorisation:** framework verifies the session exists **and** at least one `chat_messages` row for that `session_id` carries an `artifact_ref` block matching `(store_id, uri)`. Query is a JSONB scan over `chat_messages WHERE session_id = ?` — index-free is acceptable at BO chat scale. Mismatch → `404` (never `403`, to avoid leaking existence).

**Response** — depends on the adapter's `PreferredDelivery()`:

- **`Bytes`** → `200` with `Content-Type: <ref.mime>`, `Content-Length: <ref.size_bytes>`, body = streamed bytes from `adapter.Get()`.
- **`SignedURL`** → `302 Location: <url>`; framework calls `adapter.Sign(uri, ttl)` with `ttl = ARTIFACT_SIGNED_URL_TTL_SECONDS` (default 300s).

**Error responses:**

| Status | Cause |
|---|---|
| `404` | Session missing, ref not referenced by any message in that session, or `store_id` not registered in the boot-time registry |
| `410` | Blob was externally removed from the store — `adapter.Get`/`adapter.Sign` returned not-found |
| `502` | Adapter Get/Sign returned other error |

### Content-block JSONB schema on `chat_messages.content`

**New shape** (writes after ship) — typed content-block list:

```json
[
  { "type": "text", "text": "please summarise this report" },
  {
    "type":       "artifact_ref",
    "store_id":   "MAIN",
    "uri":        "sha256/9c1f2b7d...",
    "mime":       "application/pdf",
    "size_bytes": 245678,
    "sha256":     "9c1f2b7d...",
    "filename":   "Q3-report.pdf"
  }
]
```

- `type` — `"text"` or `"artifact_ref"` (extensible for future block types).
- `text` — required on `text` blocks.
- `store_id`, `uri`, `mime`, `size_bytes`, `sha256` — required on `artifact_ref` blocks.
- `filename` — optional per-ref display metadata; UI renders as the user-facing label; never used to construct storage URIs.

**Legacy shape** (existing rows — no migration): whatever the current code writes today (single opaque text JSONB). The repo layer's reader detects a non-array top level and wraps into a synthetic single-`text`-block array on load. All downstream code sees only the array shape. Writer always writes the new array shape after ship — no dual-write logic.

**Tool-call / tool-result content**: if the codebase surfaces tool interactions as their own content-block types (or fields), they may reference existing artifacts by embedding `{store_id, uri}` inside the tool_input / tool_result payload. The inner reference does not have to repeat the full `{mime, size_bytes, sha256, filename}` — it's a pointer to another message's `artifact_ref`.

### Artifact-store adapter interface (Go)

```go
type DeliveryMode int

const (
    DeliveryBytes DeliveryMode = iota
    DeliverySignedURL
)

type PutResult struct {
    URI       string // echo of the input uri (framework-computed sha256-based)
    SizeBytes int64
    Sha256    string // echo of the framework-computed hash
}

type StatResult struct {
    Exists    bool
    MIME      string
    SizeBytes int64
    Sha256    string
}

type ArtifactStore interface {
    Kind() string                                    // e.g. "s3_compatible"
    PreferredDelivery() DeliveryMode                 // declared at construction; framework caches

    Put(ctx context.Context, uri, mime string,
        r io.Reader, size int64) (PutResult, error)
    Get(ctx context.Context, uri string) (io.ReadCloser, error)                      // only called when Bytes
    Sign(ctx context.Context, uri string, ttl time.Duration) (string, error)         // only called when SignedURL
    Stat(ctx context.Context, uri string) (StatResult, error)                        // dedup pre-check + freshness

    Delete(ctx context.Context, uri string) error                                    // not called by any REST path in phase 1;
                                                                                     // provided for future GC + adapter-wrapper guard
    Probe(ctx context.Context) error                                                 // boot self-check
}
```

**`s3_compatible` implementation:**
- Uses AWS SDK v2 (`github.com/aws/aws-sdk-go-v2/service/s3`).
- Constructor reads its group's env vars; `Probe()` does a small write+read+delete round-trip against a sentinel key.
- `PreferredDelivery()` returns `SignedURL` — the S3 family offloads bandwidth to the store via presigned GETs; a bytes-proxy adapter is a future kind if a deployer needs it.
- `Put()` streams the reader directly to `PutObject` with `ContentLength: size`.
- `Sign()` calls `s3.PresignClient.PresignGetObject` with the configured TTL.

**Adapter-wrapper guard (framework layer, not adapter internals):** the framework wraps `Delete()` with a check "no locked session references `(store_id, uri)`" via a JSONB scan across `chat_messages` rows joined to `chat_sessions WHERE chat_sessions.locked_at IS NOT NULL`. Deletion is refused with an error the caller must handle. This upholds the "refs stay resolvable while any locked session references them" invariant regardless of who invokes `Delete()`.

### LLM provider adapter method

```go
type LLMProvider interface {
    // ... existing methods ...

    // RenderArtifactAsBlock converts a persisted artifact_ref into a provider-native
    // multimodal content block for inclusion in the completion request.
    // Return ErrUnsupported if the provider does not natively support this MIME —
    // framework substitutes a text surrogate ("[Attachment: <filename> (<mime>, <size>)]").
    RenderArtifactAsBlock(ctx context.Context, ref ArtifactRef,
        store ArtifactStore) (ProviderContentBlock, error)
}

var ErrUnsupported = errors.New("provider does not support this MIME natively")
```

**Anthropic implementation (first shipped):**
- `image/png`, `image/jpeg`, `image/gif`, `image/webp` → `image` block, base64-inlined bytes.
- `application/pdf` → `document` block, base64-inlined bytes.
- Everything else → `ErrUnsupported`.
- Bytes fetched via `store.Get()` for `Bytes` adapters, or via HTTPS from `store.Sign()` for `SignedURL` adapters.

**Other providers** (OpenAI, Gemini) implement the method returning `ErrUnsupported` for every MIME in phase 1. Native support per-provider is a follow-on spec.

**Text surrogate format** (framework fallback):
```
[Attachment: Q3-report.pdf (application/pdf, 240 KB)]
```
Size rendered in human-readable units. When `filename` is absent, the pattern becomes `[Attachment: <uri-tail> (<mime>, <size>)]`.

## Acceptance criteria

Grouped by contract. Every item is testable — integration-testable against a MinIO-in-compose store, or unit-testable against a mock adapter.

### Env registry + boot

- Given `ARTIFACT_STORE_MAIN_KIND=s3_compatible` and all required `_MAIN_*` vars set, `ARTIFACT_STORE_DEFAULT=MAIN`, and `ARTIFACT_MAX_UPLOAD_MB=25`, `open-bbcd` boots cleanly and the registry contains one entry keyed `MAIN`.
- Given `ARTIFACT_STORE_DEFAULT` unset while an artifact-carrying route is wired, boot fails with an error naming the missing var.
- Given a default whose `Probe()` returns error after retries, boot fails with the underlying error surfaced.
- Given a malformed group (e.g. `_KIND` set but `_BUCKET` missing), boot fails naming the specific missing sub-var.
- Given `ARTIFACT_MAX_UPLOAD_MB` unset while artifact routes are wired, boot fails naming the missing var.

### Upload — `POST /chat-sessions/{session_id}/artifacts`

- Multipart with a valid `file` field on an unlocked session returns `201` + ref JSON `{store_id, uri, mime, size_bytes, sha256}` matching the described shape exactly.
- The returned `uri` matches `sha256/<hex>` where `<hex>` equals the sha256 of the uploaded bytes.
- The returned `store_id` equals `ARTIFACT_STORE_DEFAULT` at boot time.
- Upload larger than `ARTIFACT_MAX_UPLOAD_MB` returns `413` before any store call is made.
- Upload with no `file` field returns `400`.
- Upload against a session whose `locked_at IS NOT NULL` returns `409`.
- Upload against a non-existent `session_id` returns `404`.
- Uploading identical content twice hits the dedup path: second call returns the same ref, `adapter.Put` is called at most once (verified via mock spy or store bucket count).

### Retrieval — `GET /artifacts/{path}?session_id=<id>`

- Given a session with a message content-block referencing `MAIN/sha256/<hex>` and an adapter reporting `PreferredDelivery = SignedURL`, the GET returns `302` with a `Location` header pointing at an HTTPS URL that resolves to the blob for the TTL duration (default 300s).
- Given the same ref and an adapter reporting `PreferredDelivery = Bytes`, the GET returns `200` with `Content-Type` matching the ref's `mime`, `Content-Length` matching `size_bytes`, and streamed body byte-identical to what was uploaded.
- Given a valid ref whose `(store_id, uri)` is not referenced by any message in the session, GET returns `404`.
- Given a valid ref in a session but the `store_id` prefix is not in the boot-time registry, GET returns `404`.
- Given a valid ref whose blob was externally removed from the store, GET returns `410`.
- Given a request missing the `session_id` query param, GET returns `400`.

### Content-block schema — read-time normalisation

- Reading a `chat_messages` row written before this feature (legacy JSONB shape) returns a content list of exactly one `text` block whose text equals the legacy body.
- Reading a new-shape row round-trips byte-identical.
- A tool-role message with a mixed content list of `text` + `artifact_ref` blocks is parsed and re-serialised without loss.
- Writer always writes the new array shape after ship (no dual-write); reader always accepts both.

### MCP tool-result normalisation

- Tool result with an `ImageContent` block causes an `artifact_ref` block to be appended to the tool-role message content; the stored ref's `mime` matches `ImageContent.mimeType` and `size_bytes` matches the decoded byte length.
- Tool result with an `EmbeddedResource` carrying inline `blob` produces an `artifact_ref` block (same treatment).
- Tool result with an `EmbeddedResource` carrying inline `text` produces an `artifact_ref` block whose stored blob is the UTF-8-encoded text; `mime` defaults to `resource.mimeType` or `text/plain`.
- Tool result with a URI-only `EmbeddedResource` (no `blob`/`text`) is passed through as opaque content — no `artifact_ref` is created and no store call is made.

### LLM adapter — multimodal rendering

- Anthropic adapter returns a native `image` block for `image/png` refs when building the completion request; the block's bytes match the stored blob.
- Anthropic adapter returns a native `document` block for `application/pdf` refs.
- Anthropic adapter returns `ErrUnsupported` for `application/zip`; the framework substitutes a text block matching the pattern `[Attachment: <filename> (<mime>, <size>)]`.
- The text-surrogate substitution is used verbatim for any provider that has not implemented `RenderArtifactAsBlock` (backward-compat with existing provider adapters).

### Dataset lifecycle

- Closing a draft transitions the session's `locked_at`; subsequent `POST /chat-sessions/{id}/artifacts` returns `409`.
- The framework's `Delete()` wrapper refuses to remove any `(store_id, uri)` referenced by a message in a locked session; the refusal returns an error the caller must handle (no route currently invokes this path — the guard is exercised via unit test on the wrapper).
- Deleting a chat session that is a member of a `CLOSED` dataset version continues to be refused (existing invariant, unchanged).

## Risks & assumptions

### Assumptions

- **Legacy content shape is one of a small known set.** The current write-path in the message repository layer writes exactly one JSONB shape (single opaque text blob). If a third shape is discovered during implementation, the normaliser must be extended before ship — verified by inspecting the write-path directly in the `open-bbcd` codebase.
- **BO chat is network-trusted.** The whole backoffice surface sits behind a deployer-operated VPC / VPN / SSO gateway; `open-bbcd` does not add per-user cryptographic authorisation for the artifact routes. `session_id` in the query string is authoritative when the caller is inside the trusted network — same trust model as every other BO route.
- **Buffered multipart streaming is acceptable at phase-1 scale.** Uploads stream to memory up to `ARTIFACT_MAX_UPLOAD_MB`. If deployers set the cap to hundreds of MB, a follow-up spec adds temp-file spilling; phase 1 stays memory-bounded.
- **One default store suffices operationally.** Multi-store registries work (reads route via `store_id`), but writes always go to `ARTIFACT_STORE_DEFAULT` — no per-agent, per-session, or per-MIME routing.
- **S3-API compatibility across MinIO / GCS-HMAC / R2 / B2 is sufficient at the AWS SDK v2 level.** All major implementations support `PutObject`, `GetObject`, `HeadObject`, and presigned-GET signing v4. Edge-case differences (path-style vs virtual-hosted-style, header canonicalisation) are covered by the `PATH_STYLE` env var and SDK defaults.
- **Anthropic remains the default LLM provider.** Other providers ship with `RenderArtifactAsBlock` returning `ErrUnsupported`; adding native support per-provider is out of this spec's scope.
- **Deployer configures at least one artifact store before enabling BO chat.** Fresh deploys with the chat-artifacts routes wired but no `ARTIFACT_STORE_DEFAULT` will boot-fail — deployer must configure or disable via build flag / feature gate (implementation detail; if disabled, chat-artifacts routes are not registered).

### Risks

- **Presigned URL TTL default (300s).** May be too short for a slow client downloading a large PDF, or long enough to be a concern if leaked in shared browsers. Mitigation: `ARTIFACT_SIGNED_URL_TTL_SECONDS` env var lets deployers tune per environment.
- **JSONB scan on retrieval for session-scope check.** O(N) over the session's messages per request. Fine at typical BO scale (dozens–hundreds of messages); a session with thousands of messages could see noticeable latency. Mitigation: no action for phase 1; if it bites, `/new-spec chat-artifacts-index` adds a materialised `chat_message_artifacts` reverse-index table.
- **S3-API compatibility across providers has edge cases.** Presigned URL signing (v4 vs v2) and header canonicalisation vary. Mitigation: integration tests run against MinIO in the compose profile; each additional store kind requires explicit validation at implementation time (not in phase 1 scope).
- **Content-addressable URIs give the same blob one identity across users.** Same content uploaded twice → same ref. Not a leak within the trusted BO context, but a documented design property. Session-scope on reads prevents ref-guessing from succeeding cross-session.
- **Legacy-shape detection at read time is a runtime hazard.** Malformed / half-written rows (e.g. a race during a rolling deploy) could confuse readers. Mitigation: writer always writes the new array shape after ship; reader accepts both; no dual-write logic that could introduce mixed intermediate states.
- **`filename` field is user-supplied and untrusted for storage.** Used only as a display label in ref metadata; never used to construct the storage URI (URI is always `sha256/<hex>`). Explicit invariant enforced at the multipart parsing boundary.
- **Boot failure on `Probe()` error is intentional but noisy.** A store credentials rotation or transient network blip at boot would keep the daemon down until recovery. Mitigation: `Probe()` has a bounded retry (3 attempts, exponential backoff to ~4s max) in the adapter implementation; deployers with unstable object stores can also lengthen the backoff via a follow-on env var if needed.
- **The write-path forever ties content identity to sha256.** If sha256 is ever deprecated for content-addressing (unlikely in the phase-1 horizon), historical refs would need a translation mechanism. Not a real risk at phase 1; noted for future consideration.

---

## Cross-references

- **Arch-of-record source (merged):** `docs/architecture/logs/2026-09-28-artifact-support/README.md` (PR #5) — the arch PR that established `chat-artifacts`, `deployed-runtime-artifacts`, `artifact-store-adapter` capabilities, and the `artifacts` DDD context.
- **Modularity node (merged):** `docs/architecture/current/modularity/open-bbcd/feedback-datasets/README.md` (primary attachment) and `docs/architecture/current/modularity/open-bbcd/artifacts/README.md` (sibling substrate) — both from PR #6.
- **Arch invariants referenced:** `docs/architecture/current/ddd/contexts/artifacts.md § Invariants`, `docs/architecture/current/constraints.md § Hard technical limits`, `docs/architecture/current/nfrs.md § Security`, `docs/architecture/current/assumptions.md § Design decisions (locked)`.
- **Follow-on specs anticipated:** `deployed-runtime-artifacts` (deployed surface), `chat-artifacts-gc` (blob lifecycle), `chat-artifacts-index` (retrieval performance), per-provider multimodal support (OpenAI, Gemini).
