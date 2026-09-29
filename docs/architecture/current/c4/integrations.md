# Integrations

| System | Kind | Protocol | Auth | SLA/regulatory |
|--------|------|----------|------|----------------|
| Client frontend | Inbound consumer | AG-UI over Server-Sent Events (SSE), HTTPS | `user_id` in request body/query, trusted from the operator's gateway | ARCH_GAP — no SLA sourced |
| Client backend (REST OR MCP) | Outbound tool provider | Either plain HTTP/REST (bridged as MCP by OpenBBC's `http_endpoint` `tool_backends` kind) OR MCP over SSE / Streamable HTTP (`mcp_client` proxy). No pre-existing MCP server is required. | Server-to-server credentials in the `tool_backends.config` JSONB; per-session HTTP header overrides merged in the BO chat + eval paths (`chat_sessions.backend_header_overrides` migration 016; `evals.header_overrides` migration 023). No per-session overrides on the deployed path today. | ARCH_GAP — no SLA / regulatory constraints sourced |
| LLM providers — Anthropic (default), OpenAI, Gemini | Outbound completion | HTTPS via Google ADK + LiteLLM (aikdm, aikdm-runner); direct Anthropic API (open-bbcd) | `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, `GEMINI_API_KEY` from env (in k8s: Secrets mounted per drainer; in production compose: platform secret store) | Provider-defined SLAs; ARCH_GAP for internal-policy targets |
| Operator's auth gateway | Inbound trust mediator | HTTPS (whatever the gateway speaks upstream: session cookie, bearer, mTLS, SSO) | Gateway-owned; injects a verified `user_id` before forwarding to `open-bbcd` | Operator-owned; ARCH_GAP for internal policy |
| Claude Code (`flow-map-compiler`) | Discovery-side skill host | Local IPC (Claude Code plugin API) | Runs client-side on the discovery author's machine; the resulting zip is uploaded via wizard authenticated by the same operator gateway that fronts the backoffice | ARCH_GAP |
| GHCR (`ghcr.io/dacdigital/openbbc/*`) | Outbound (build/publish) + inbound (pull to k8s) | OCI registry API | GHCR PAT for publish (via `GITHUB_TOKEN` in `.github/workflows/publish-images.yml`); anonymous or `imagePullSecrets` for pull depending on package visibility | Images `open-bbcd`, `aikdm-runner`, `aikdm`; tags `pr-<num>`, `main`, `sha-<short>`, semver, `latest` |
| Object store (deployer-provided) | Outbound artifact-store backend | S3 API over HTTPS (first-shipped `s3_compatible` `artifact-store-adapter` kind — covers AWS S3, MinIO, GCS with HMAC, R2, B2, any S3-API endpoint). Framework-side call surface is uniform `put`/`get`/`sign`/`delete`/`stat`; wire is adapter-specific. | Credentials from **env vars only** — per-store `ARTIFACT_STORE_<ID>_ACCESS_KEY`, `_SECRET_KEY`, `_ENDPOINT`, `_BUCKET`, `_REGION`, optional path-style flag; `ARTIFACT_STORE_DEFAULT=<ID>` nominates the write target. Same secret class as LLM provider API keys (not persisted in Postgres). | Deployer-owned SLA; ARCH_GAP for internal-policy targets. Regulatory tag: user-content (deployer-classified) — bytes are chat / deployed artifacts. |

<!-- migrated from _migration-quarantine/PRODUCTION.md § 2 Integrating your frontend, § 3 MCP layer, § 4 Headers, § 5 Auth model, § 7 Provider LLM keys, ARCHITECTURE.md § Protocols, § flow-map-compiler on 2026-09-28. Updated 2026-09-28 for OpenBBC PR #50 — client-backend row split into REST-bridge/MCP-proxy alternatives; GHCR row added. Updated 2026-09-28 for artifact-support — added Object store row. -->

## Contracts

- **AG-UI event stream (client frontend ↔ open-bbcd).** Event types: `RUN_STARTED`,
  `TEXT_MESSAGE_START/CONTENT/END`, `TOOL_CALL_START/ARGS/END`, `TURN_END`, `ERROR`.
  Upstream spec: [ag-ui-protocol/ag-ui](https://github.com/ag-ui-protocol/ag-ui).
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
  `open-bbcd` at `GET /evals/{id}/export.yaml`.
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
  - `sign(uri, ttl) → https_url` — called when `PreferredDelivery() == SignedURL`; the
    framework passes `ttl = ARTIFACT_SIGNED_URL_TTL_SECONDS` (default `300s`). Routes to
    the store named by the ref's `store_id`.
  - `stat(uri) → {exists, mime, size_bytes, sha256}` — `exists = false` signals the blob
    has been externally removed and drives a `410 Gone` on the retrieval route (distinct
    from `404` for session-scope mismatch or unknown `store_id`).
  - `delete(uri)`
  - `probe() → ok | error` — boot-time self-check per registered store. Boot **fails** with
    a clear error if any of: `ARTIFACT_STORE_DEFAULT` is unset while artifact routes are
    compiled in; the default store's `probe()` fails after bounded retries; any store group
    is malformed (missing kind-specific vars); two stores declare the same `<ID>`; or
    `ARTIFACT_MAX_UPLOAD_MB` is unset (required whenever artifact routes are wired).
  Kinds are versioned via the env-var `KIND` value. First-shipped kind: `s3_compatible`.
  Adapter config schema per kind is a set of `ARTIFACT_STORE_<ID>_*` env-var names declared
  alongside the kind registration in code (not a runtime plug-in surface). No REST CRUD,
  test-connection button, or BO UI exists for stores — reconfiguration is a redeploy.
- **AG-UI `ARTIFACT_REF` event-type extension (open-bbcd → client frontend).**
  Complements the base AG-UI events (`RUN_STARTED`, `TEXT_MESSAGE_*`, `TOOL_CALL_*`,
  `TURN_END`, `ERROR`) with `ARTIFACT_REF` carrying `{store_id, uri, mime, size_bytes,
  sha256}` inside the assistant turn. SDKs that don't understand the event ignore it (SSE
  unknown-event semantics) and render text-only; SDKs that do understand it resolve refs
  via `GET /artifacts/{store_id}/{uri}` scoped by the session's `user_id`. Upstream
  AG-UI spec: [ag-ui-protocol/ag-ui](https://github.com/ag-ui-protocol/ag-ui).

<!-- migrated from _migration-quarantine/ARCHITECTURE.md § flow-map-compiler, § aikdm, § MCP wiring, § Protocols, PRODUCTION.md § 2, § 3 MCP layer, § 6 Batch operations on 2026-09-28. Updated 2026-09-28 for OpenBBC PR #50 — explicit bridge-vs-proxy contract, flow-map schema v2, drainer JSON surfaces. Updated 2026-09-28 for artifact-support — artifact-store adapter interface + AG-UI ARTIFACT_REF event extension. -->
