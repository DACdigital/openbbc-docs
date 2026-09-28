# Integrations

| System | Kind | Protocol | Auth | SLA/regulatory |
|--------|------|----------|------|----------------|
| Client frontend | Inbound consumer | AG-UI over Server-Sent Events (SSE), HTTPS | `user_id` in request body/query, trusted from the operator's gateway | ARCH_GAP — no SLA sourced |
| Client backend (REST OR MCP) | Outbound tool provider | Either plain HTTP/REST (bridged as MCP by OpenBBC's `http_endpoint` `tool_backends` kind) OR MCP over SSE / Streamable HTTP (`mcp_client` proxy). No pre-existing MCP server is required. | Server-to-server credentials in the `tool_backends.config` JSONB; per-session HTTP header overrides merged in the BO chat + eval paths (`chat_sessions.backend_header_overrides` migration 016; `evals.header_overrides` migration 023). No per-session overrides on the deployed path today. | ARCH_GAP — no SLA / regulatory constraints sourced |
| LLM providers — Anthropic (default), OpenAI, Gemini | Outbound completion | HTTPS via Google ADK + LiteLLM (aikdm, aikdm-runner); direct Anthropic API (open-bbcd) | `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, `GEMINI_API_KEY` from env (in k8s: Secrets mounted per drainer; in production compose: platform secret store) | Provider-defined SLAs; ARCH_GAP for internal-policy targets |
| Operator's auth gateway | Inbound trust mediator | HTTPS (whatever the gateway speaks upstream: session cookie, bearer, mTLS, SSO) | Gateway-owned; injects a verified `user_id` before forwarding to `open-bbcd` | Operator-owned; ARCH_GAP for internal policy |
| Claude Code (`flow-map-compiler`) | Discovery-side skill host | Local IPC (Claude Code plugin API) | Runs client-side on the discovery author's machine; the resulting zip is uploaded via wizard authenticated by the same operator gateway that fronts the backoffice | ARCH_GAP |
| GHCR (`ghcr.io/dacdigital/openbbc/*`) | Outbound (build/publish) + inbound (pull to k8s) | OCI registry API | GHCR PAT for publish (via `GITHUB_TOKEN` in `.github/workflows/publish-images.yml`); anonymous or `imagePullSecrets` for pull depending on package visibility | Images `open-bbcd`, `aikdm-runner`, `aikdm`; tags `pr-<num>`, `main`, `sha-<short>`, semver, `latest` |
| Object store (deployer-provided) | Outbound artifact-store backend | S3 API over HTTPS (first-shipped `s3_compatible` `artifact-store-adapter` kind — covers AWS S3, MinIO, GCS with HMAC, R2, B2, any S3-API endpoint). Framework-side call surface is uniform `put`/`get`/`sign`/`delete`/`stat`; wire is adapter-specific. | Credentials in `artifact_stores.config` JSONB (access key + secret; endpoint URL; bucket; region; optional path-style flag). Treat as secrets — same handling class as `tool_backends.config`. | Deployer-owned SLA; ARCH_GAP for internal-policy targets. Regulatory tag: user-content (deployer-classified) — bytes are chat / deployed artifacts. |

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
  framework-side contract, adapter-per-kind wire translation:
  - `put(bytes, mime) → {uri, size_bytes, sha256}`
  - `get(uri) → bytes` OR `sign(uri, ttl) → https_url`
  - `stat(uri) → {mime, size_bytes, sha256, exists}`
  - `delete(uri)`
  - `test() → ok | error` (round-trip small write+read+delete under the configured
    credentials; drives the BO "Test connection" button)
  Kinds are versioned via the `artifact_stores.kind` string. First-shipped kind:
  `s3_compatible`. Adapter config schema per kind is declared alongside the kind
  registration (analogous to `tool_backends.kind` + `config` JSONB schemas).
- **AG-UI `ARTIFACT_REF` event-type extension (open-bbcd → client frontend).**
  Complements the base AG-UI events (`RUN_STARTED`, `TEXT_MESSAGE_*`, `TOOL_CALL_*`,
  `TURN_END`, `ERROR`) with `ARTIFACT_REF` carrying `{store_id, uri, mime, size_bytes,
  sha256}` inside the assistant turn. SDKs that don't understand the event ignore it (SSE
  unknown-event semantics) and render text-only; SDKs that do understand it resolve refs
  via `GET /artifacts/{store_id}/{uri}` scoped by the session's `user_id`. Upstream
  AG-UI spec: [ag-ui-protocol/ag-ui](https://github.com/ag-ui-protocol/ag-ui).

<!-- migrated from _migration-quarantine/ARCHITECTURE.md § flow-map-compiler, § aikdm, § MCP wiring, § Protocols, PRODUCTION.md § 2, § 3 MCP layer, § 6 Batch operations on 2026-09-28. Updated 2026-09-28 for OpenBBC PR #50 — explicit bridge-vs-proxy contract, flow-map schema v2, drainer JSON surfaces. Updated 2026-09-28 for artifact-support — artifact-store adapter interface + AG-UI ARTIFACT_REF event extension. -->
