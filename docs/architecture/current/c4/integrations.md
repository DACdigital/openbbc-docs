# Integrations

| System | Kind | Protocol | Auth | SLA/regulatory |
|--------|------|----------|------|----------------|
| Client frontend | Inbound consumer | AG-UI over Server-Sent Events (SSE), HTTPS | `user_id` in request body/query, trusted from the operator's gateway | ARCH_GAP — no SLA sourced |
| Client backend (REST OR MCP) | Outbound tool provider | Either plain HTTP/REST (bridged as MCP by OpenBBC's `http_endpoint` `tool_backends` kind) OR MCP over SSE / Streamable HTTP (`mcp_client` proxy). No pre-existing MCP server is required. | Server-to-server credentials in the `tool_backends.config` JSONB; per-session HTTP header overrides merged in the BO chat + eval paths (`chat_sessions.backend_header_overrides` migration 016; `evals.header_overrides` migration 023). No per-session overrides on the deployed path today. | ARCH_GAP — no SLA / regulatory constraints sourced |
| LLM providers — Anthropic (default), OpenAI, Gemini, plus, on `open-bbcd`'s `bifrost` adapter, the v1 key-only allow-list (anthropic, openai, gemini, mistral, groq, cohere, openrouter, deepseek, xai, cerebras) | Outbound completion | HTTPS via Google ADK + LiteLLM (aikdm, aikdm-runner). `open-bbcd` selects one LLM adapter at boot via `OPENBBC_LLM_ADAPTER`: `anthropic` (default) calls the Anthropic API directly through `anthropic-sdk-go`; `bifrost` calls the provider named in `OPENBBC_DEFAULT_MODEL` (`<provider>/<model>`) through the **Bifrost Go SDK** (`github.com/maximhq/bifrost/core`, Apache-2.0) embedded in-process. This is a library, not a gateway hop: no Bifrost container and no extra network leg. Optional `<PROVIDER>_BASE_URL` overrides the endpoint for `openai`/`anthropic`/`cohere`/`mistral` only (`https`, or `http` to a loopback host only). | `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, `GEMINI_API_KEY` from env (in k8s: Secrets mounted per drainer; in production compose: platform secret store). On the `bifrost` adapter, `open-bbcd` builds Bifrost's `Account` at boot from the selected provider's `<PROVIDER>_API_KEY` env var only (one provider, one key; other providers' vars are not read); no keys and no provider config live in Postgres or in a Bifrost config file. | Provider-defined SLAs; ARCH_GAP for internal-policy targets |
| Operator's auth gateway | Inbound trust mediator | HTTPS (whatever the gateway speaks upstream: session cookie, bearer, mTLS, SSO) | Gateway-owned; injects a verified `user_id` before forwarding to `open-bbcd` | Operator-owned; ARCH_GAP for internal policy |
| Claude Code (`flow-map-compiler`) | Discovery-side skill host | Local IPC (Claude Code plugin API) | Runs client-side on the discovery author's machine; the resulting zip is uploaded via wizard authenticated by the same operator gateway that fronts the backoffice | ARCH_GAP |
| GHCR (`ghcr.io/dacdigital/openbbc/*`) | Outbound (build/publish) + inbound (pull to k8s) | OCI registry API | GHCR PAT for publish (via `GITHUB_TOKEN` in `.github/workflows/publish-images.yml`); anonymous or `imagePullSecrets` for pull depending on package visibility | Images `open-bbcd`, `aikdm-runner`, `aikdm`; tags `pr-<num>`, `main`, `sha-<short>`, semver, `latest` |
| Object store (deployer-provided) | Outbound artifact-store backend | S3 API over HTTPS (first-shipped `s3_compatible` `artifact-store-adapter` kind — covers AWS S3, MinIO, GCS with HMAC, R2, B2, any S3-API endpoint). Framework-side call surface is uniform `put`/`get`/`sign`/`delete`/`stat`; wire is adapter-specific. | Credentials from **env vars only** — per-store `ARTIFACT_STORE_<ID>_ACCESS_KEY`, `_SECRET_KEY`, `_ENDPOINT`, `_BUCKET`, `_REGION`, optional path-style flag; `ARTIFACT_STORE_DEFAULT=<ID>` nominates the write target. Same secret class as LLM provider API keys (not persisted in Postgres). Store credentials must let `Stat` report a missing key as not-found (for AWS S3 this needs `s3:ListBucket`); `probe()` verifies it at boot. | Deployer-owned SLA; ARCH_GAP for internal-policy targets. Regulatory tag: user-content (deployer-classified) — bytes are chat / deployed artifacts. |

<!-- migrated from _migration-quarantine/PRODUCTION.md § 2 Integrating your frontend, § 3 MCP layer, § 4 Headers, § 5 Auth model, § 7 Provider LLM keys, ARCHITECTURE.md § Protocols, § flow-map-compiler on 2026-09-28. Updated 2026-09-28 for OpenBBC PR #50 — client-backend row split into REST-bridge/MCP-proxy alternatives; GHCR row added. Updated 2026-09-28 for artifact-support — added Object store row. Updated 2026-10-01 for sync-deployed-runtime-artifacts — Object store missing-key-as-not-found credential requirement. Updated 2026-10-07 for bifrost — open-bbcd LLM adapter selectable between direct Anthropic and the embedded Bifrost Go SDK. Updated 2026-10-07 for sync-bifrost — v1 provider allow-list, single selected key, <PROVIDER>_BASE_URL. -->

## Contracts

- **AG-UI event stream (client frontend ↔ open-bbcd).** Event types: `RUN_STARTED`,
  `TEXT_MESSAGE_START/CONTENT/END`, `TOOL_CALL_START/ARGS/END`, `TOOL_CALL_RESULT`,
  `STEP_STARTED/FINISHED`, `RUN_FINISHED`, `RUN_ERROR`, and `CUSTOM` (`ARTIFACT_REF`).
  Upstream spec: [ag-ui-protocol/ag-ui](https://github.com/ag-ui-protocol/ag-ui).
  **Sub-agent progress** uses the standard AG-UI step events. `STEP_STARTED` /
  `STEP_FINISHED` bracket each sub-agent run with
  `stepName = "<binding name>:<toolCallId of the parent's agent call>"`, which is unique
  across parallel calls to the same binding. Their `rawEvent` is
  `{childSessionId, parentToolCallId, description}` on start and `{childSessionId, isError}`
  on finish; nested steps attach to the step whose `toolCallId` equals
  `rawEvent.parentToolCallId`. The sub-agent's own `TOOL_CALL_START/ARGS/END/RESULT` events
  are forwarded with `toolCallId = "<childSessionId>:<tool_use id>"` (prefixed exactly once
  at any depth) and `rawEvent = {childSessionId}`. Sub-agent run, `TEXT_MESSAGE_*`, error
  and `ARTIFACT_REF` events are not forwarded. For each `agent` call the root stream
  carries, in order: the root's `TOOL_CALL_START/ARGS/END`, `STEP_STARTED`, the child's
  tagged events, `STEP_FINISHED`, then the root's `TOOL_CALL_RESULT` carrying the
  sub-agent's text. The stream has exactly one `RUN_STARTED` and one `RUN_FINISHED`.
  Streams of versions without the agent tool are unchanged. SDKs that ignore steps and
  `rawEvent` still render the root's text and tool calls correctly, but also show the
  worker's tool calls as extra top-level tool calls (each id is uniquely prefixed, so
  nothing collides). Tool-result artifact refs are
  carried as the `CUSTOM` `ARTIFACT_REF` event (below); the BO chat stream emits them with
  the same semantics on both its transports (an `artifact_ref` frame on JSONL, the `CUSTOM`
  event on AG-UI).
- **Tool-backend wire protocol (open-bbcd ↔ client backend).** Two `tool_backends` kinds:
  - `http_endpoint` — OpenBBC's **built-in MCP-over-REST bridge**. `open-bbcd` calls the
    registered REST endpoint directly and exposes it to the agent as an MCP tool. No client
    MCP server required.
  - `mcp_client` — MCP over SSE or Streamable HTTP. `open-bbcd` acts as an MCP client to an
    existing MCP server the operator points it at.
- **`.flow-map/` (discovery → wizard).** Schema version 2. Layout: `AGENTS.md`, `APP.md`,
  `glossary.md`, `skills/<id>.md`, `flows/<id>.md`, `endpoints/<id>.md` (every endpoint
  frontmatter carries `proposed: true`). Full schema in
  `bbc-discovery/flow-map-compiler/references/output-schemas.md` (versioned); the 15-rule
  contract-triple lint lives in `references/lint-contract.md`. LOCKED anti-goals: never
  generate MCP server code; never call any registry API; never run target-repo code.
- **aikdm bundle format.** `aikdm/schemas/prompt-v1.yaml` — sections `metadata`,
  `main_prompt`, `capabilities[]`, `skills[]`, `external_actions[]`. Versioned; schema
  changes bump the file.
- **Aikdm eval input.** `eval-input.yaml` — agent version + dataset version pair, consumed by
  `aikdm evaluate` and `aikdm train-agent`. Structural shape defined by aikdm; served by
  `open-bbcd` at `GET /evals/{id}/export.yaml`. For versions with the agent tool it also
  carries an `agent_tool` block (`max_depth`, `max_parallel`, the root's bindings) and a
  `subagents` map keyed by pinned `agent_version_id` holding each reachable version's
  bundle, tool wiring, and own bindings — transitive, so aikdm needs no further lookups.
  The bundle schema (`prompt-v1.yaml`) itself is unchanged.
- **Agent tool (open-bbcd / aikdm ↔ LLM).** Built-in tool definition presented to the LLM
  when the version has `agent_tool_enabled` **and at least one binding**, placed
  immediately after `Skill`: `agent(subagent: enum<binding name>, description: string,
  prompt: string)` → `text`. All three properties are required and no others are allowed.
  The tool carries text only: no artifacts argument and no refs in its result; artifact
  scope is per session. The tool description renders each binding's `name` + `note`. On
  success, `tool_result` content is the concatenated `text` blocks of the sub-agent's last
  assistant message, with `is_error: false` and no ids or envelope. On error it is
  `is_error: true` with content `"<code>: <details>"`, where code is one of
  `max_depth_exceeded`, `unknown_subagent`, `invalid_input`, `session_locked`,
  `subagent_max_tool_rounds`, `subagent_failed` or `cancelled`. Errors go back to the
  calling LLM and never fail the turn. Endpoint tools are not prefixed, so enabling the tool
  or adding a binding is refused with `409` when the agent has an endpoint whose sanitised
  tool name is `agent`. The same definition is used in `open-bbcd` and `aikdm`, so evals
  exercise the production contract.
- **Drainer discovery.** `GET /agent_versions.json?status=PENDING`,
  `GET /evals.json?status=PENDING`, `GET /training-sessions.json?status=PENDING` — the three
  JSON list surfaces the k8s CronJobs and one-shot scripts use to enumerate PENDING work.
- **Artifact-store adapter interface (open-bbcd internal → object store).** Uniform
  framework-side contract, adapter-per-kind wire translation. Each adapter declares its
  delivery mode at construction; the framework calls exactly one of `Get` or `Sign` per
  retrieval based on that declaration.
  - `PreferredDelivery() → Bytes | SignedURL` — adapter-declared at construction and
    cached. Drives which retrieval path the framework calls, and therefore which HTTP
    response the retrieval REST route returns: `200` proxied bytes when `Bytes`, `302
    Location <signed_url>` when `SignedURL`.
  - `put(bytes, mime) → {uri, size_bytes, sha256}` — routes to the env-nominated
    `ARTIFACT_STORE_DEFAULT`.
  - `get(uri) → bytes` — called when `PreferredDelivery() == Bytes`. Routes to the store
    named by the `store_id` embedded in the calling `artifact_ref`.
  - `sign(uri, ttl, SignOptions{ContentType, ContentDisposition}) → https_url` — called
    when `PreferredDelivery() == SignedURL`; the framework passes
    `ttl = ARTIFACT_SIGNED_URL_TTL_SECONDS` (default `300s`) and per-request response
    overrides (content-addressed blobs are shared by rows with different filenames); kinds
    that cannot set response overrides ignore them. Routes to the store named by the ref's
    `store_id`.
  - `stat(uri) → {exists, mime, size_bytes, sha256}` — `exists = false` signals the blob
    has been externally removed and drives a `410 Gone` on the retrieval route (distinct
    from `404` for session-scope mismatch or unknown `store_id`). Also drives the render
    fallback (a missing blob renders as a text surrogate instead of failing the turn) and
    the upload `put` skip when the content-addressed blob already exists.
  - `delete(uri)`
  - `probe() → ok | error` — boot-time self-check per registered store. Boot **fails** with
    a clear error if any of: `ARTIFACT_STORE_DEFAULT` is unset while artifact routes are
    compiled in; the default store's `probe()` fails after bounded retries; any store group
    is malformed (missing kind-specific vars); two stores declare the same `<ID>`;
    `ARTIFACT_MAX_UPLOAD_MB` is unset (required whenever artifact routes are wired); or a
    `stat` on a random missing key is not reported as not-found.
  Kinds are versioned via the env-var `KIND` value. First-shipped kind: `s3_compatible`.
  Adapter config schema per kind is a set of `ARTIFACT_STORE_<ID>_*` env-var names declared
  alongside the kind registration in code (not a runtime plug-in surface). No REST CRUD,
  test-connection button, or BO UI exists for stores — reconfiguration is a redeploy.
- **AG-UI `ARTIFACT_REF` (open-bbcd → client frontend).** Carried as an AG-UI `CUSTOM`
  event — `{type: "CUSTOM", name: "ARTIFACT_REF", value: {toolCallId, storeId, uri, mime,
  sizeBytes, sha256, filename}}` — because official AG-UI SDKs validate `type` against a
  closed set, so a new top-level event type would fail validation rather than be ignored.
  Emitted only for tool-result refs, once per ref after the round's tool-role message and its
  session-artifact rows commit (so the ref is immediately resolvable); never for user
  uploads, child-session tool results, or `{store_id, uri}` pointers in tool input.
  `TOOL_CALL_RESULT` carries the normalised remainder (no base64 when the artifact registry
  is enabled). Clients that do not handle the `CUSTOM` name ignore it and render text-only;
  clients that do resolve the ref against their own surface's nested retrieval route
  (`GET /deployed/{agent_id}/sessions/{sid}/artifacts/{store_id}/{uri...}?user_id=X` on the
  deployed surface). The `value` payload changes only additively. Upstream AG-UI spec:
  [ag-ui-protocol/ag-ui](https://github.com/ag-ui-protocol/ag-ui).

<!-- migrated from _migration-quarantine/ARCHITECTURE.md § flow-map-compiler, § aikdm, § MCP wiring, § Protocols, PRODUCTION.md § 2, § 3 MCP layer, § 6 Batch operations on 2026-09-28. Updated 2026-09-28 for OpenBBC PR #50 — explicit bridge-vs-proxy contract, flow-map schema v2, drainer JSON surfaces. Updated 2026-09-28 for artifact-support — artifact-store adapter interface + AG-UI ARTIFACT_REF event extension. Updated 2026-09-30 for multiagent-tools — AG-UI step events for sub-agent progress, eval-input subagents section, agent tool contract. Updated 2026-10-01 for sync-deployed-runtime-artifacts — ARTIFACT_REF as AG-UI CUSTOM event, BO stream artifact refs, text-only agent tool, adapter SignOptions + missing-key probe. Updated 2026-10-05 for sync-multiagent-feature — AG-UI stepName/rawEvent/prefixed child toolCallId, RUN_FINISHED/RUN_ERROR, agent tool result and error contract. -->
