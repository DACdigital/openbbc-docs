# Assumptions

## Scope assumptions

- **Client backend can be plain HTTP/REST — no pre-existing MCP server required.** At runtime
  `open-bbcd` bridges each registered backend as an MCP tool via one of two `tool_backends`
  kinds:
  - `http_endpoint` — `open-bbcd` calls a REST endpoint directly, exposing it to the agent as
    an MCP tool (the built-in MCP-over-REST bridge).
  - `mcp_client` — `open-bbcd` proxies to an existing MCP server the operator points it at.
  Either kind is a first-class shipping mode.
- **Discovery is a proposer, not an MCP-server generator.** The `flow-map-compiler` Claude
  Code skill scans the client **frontend** repo, extracts every backend call site, and emits
  `endpoints/<id>.md` files carrying `proposed: true` — HTTP method, path, params, response
  shape, proposed MCP tool name. Its LOCKED anti-goals include "never generate MCP server
  code" and "never assume an MCP server exists". Wiring those proposed endpoints into a
  runnable MCP surface is downstream engineering (or a future generator skill).
- Client frontend uses the **AG-UI protocol** for the deployed-agent chat surface. The FE
  never talks to the client backend directly — it talks to `open-bbcd`, which dispatches tool
  calls to the MCP-bridged or MCP-proxied backend.
- **Single entrypoint agent; multi-agent via the agent tool.** Every BO chat session and
  every deployed session still talks to exactly one **root** agent version. Multi-agent
  topologies (planner → coordinator → workers, …) are composed by enabling the **agent
  tool** on a version: the root's LLM spawns other, pinned agent versions as sub-agents
  inside the same turn, Claude-Code-style (fresh context, prompt in, final answer out).
  Multiple concurrent deployments per agent chain remain unsupported.
- **`aikdm` (the Python CLI) is DB-unaware** — it talks REST to `open-bbcd` via `scripts/run_eval.sh`
  and `scripts/train_from_session.sh`. The alpha-drainer path is a scripted composition
  (`process_pending_alphas.sh` → `generate_alpha.sh` → `aikdm generate-agent` + `seed_bundle.py`)
  where `seed_bundle.py` — packaged into the `aikdm-runner` image alongside `aikdm` — writes
  the resulting bundle directly to Postgres. `aikdm` itself remains DB-unaware; the runner
  image is not.
- **Artifact-store credentials report a missing key as not-found.** The artifact framework
  distinguishes a blob that is gone (`410` on retrieval, text surrogate on render) from a
  transient store error by `Stat`, so the deployer's store credentials must let `Stat`
  report a missing key as not-found (for AWS S3 this needs `s3:ListBucket`). Each store's
  boot-time `probe()` checks it and fails boot otherwise.

<!-- migrated from _migration-quarantine/DESIGN.md § Assumptions, § Out of Scope, ARCHITECTURE.md § System Overview, § MCP wiring on 2026-09-28. Updated 2026-09-28 for OpenBBC PR #50 (mig 025 PENDING alpha + 026 discovery_zip + Helm chart + aikdm-runner image + published GHCR images) and flow-map-compiler skill LOCKED anti-goals. Updated 2026-10-01 for sync-deployed-runtime-artifacts — artifact-store missing-key assumption. -->

## Design decisions (locked)

- **Auth-agnostic ship** — no built-in auth on any route. Operators front `open-bbcd` with
  their own gateway. Rationale: auth policy varies wildly (SSO, mTLS, API gateway, tenant
  scoping) and baking one in would push assumptions onto every operator. Date: current shipping
  design as of migration 024. Trade-off documented in `nfrs.md § Security` and
  `ddd/access-model.md`.
- **Agent-level architecture vs version-level prompts (migration 017)** — endpoint→backend
  wiring is agent-keyed (`agent_endpoint_backend`) because endpoints are structural and frozen
  on first version; MCP attachments are version-keyed (`agent_version_mcp_backend`) with an
  editable `note` because prompt guidance varies per version. Date: migration 017.
- **At most one DEPLOYED per agent chain (migration 011).** Deploying a new version implicitly
  rotates the previous one. Rationale: keep the deployed runtime unambiguous. Date: migration
  011.
- **At most one DRAFT per dataset (migration 019).** Closing a DRAFT seeds the next DRAFT with
  the CLOSED version's sessions (migration 020) so users see cumulative content, not an empty
  next version. Date: migrations 019–020.
- **Score formula is global pass-rate** — `sum(passed_criteria) / sum(total_criteria)` across
  all sessions in the eval (migration 022). Every criterion counts equally; longer sessions
  weigh proportionally more. Date: migration 022.
- **Distroless runtime image, CGO off.** `open-bbcd/Dockerfile` runtime on
  `gcr.io/distroless/static-debian12:nonroot`; the container `HEALTHCHECK` uses the
  `open-bbcd healthcheck` subcommand (no `curl` in distroless). Date: current shipping design.
- **Discovery zip lives on the `agents` row, not on local disk.** `agents.discovery_zip BYTEA`
  (migration 026) inlines the uploaded `.flow-map/` archive; `internal/storage/storage.go`
  has been removed; the deprecated `discovery_file_path` column is retained ignored to keep
  the migration reversible. The `open-bbcd` pod therefore needs no persistent volume — all
  state lives in Postgres. Date: migration 026 (OpenBBC PR #50).
- **PENDING is the alpha-generation state.** `agent_versions.status` now includes `PENDING`
  between `INITIALIZING` and `READY` (migration 025). Wizard Finalize transitions a root
  version `INITIALIZING → PENDING`; the async drainer (`scripts/process_pending_alphas.sh`
  → `generate_alpha.sh`) generates the bundle and lands it via `seed_bundle.py`, then
  transitions `PENDING → READY`. Date: migration 025 (OpenBBC PR #50).
- **Contract between `aikdm` and `open-bbcd` is REST + a versioned YAML schema.** No shared
  library. Section structure declared in `aikdm/schemas/prompt-v1.yaml`.
- **Helm chart is the shipping k8s deployment path** (`deploy/helm/openbbc/`): open-bbcd
  Deployment + Service + optional Ingress, optional in-cluster Postgres StatefulSet, and three
  CronJobs (alphas / evals / trainings) running the `aikdm-runner` image. Date: OpenBBC PR
  #50.
- **Artifacts live in a pluggable artifact store; `open-bbcd` holds only refs and
  per-session artifact metadata.** Chat and deployed-runtime file exchanges (any MIME, both
  directions, both framework legs — staged user upload and MCP tool output) flow through an
  `artifact-store-adapter` to a deployer-configured Object store. A `{store_id, uri}` the
  LLM writes into tool input is opaque and never resolved by the framework. **There is no
  assistant-emission leg** — the LLM does not itself generate binary content (images, files);
  only tools return artifacts back to the assistant. With the artifact registry enabled,
  bytes never enter Postgres; `chat_messages.content` and `deployed_messages.content` JSONB
  carry typed content blocks including `artifact_ref` pointers only, and the per-session
  read scope lives in `chat_session_artifacts` / `deployed_session_artifacts`. First shipped kind:
  `s3_compatible` (covers AWS S3, MinIO, GCS-HMAC, R2, B2, any S3-API endpoint).
  **Store registry is env-driven, not REST-driven** — deployers declare stores at boot via
  `ARTIFACT_STORE_<ID>_*` env vars and nominate a write target via
  `ARTIFACT_STORE_DEFAULT=<ID>`. There is no REST / BO surface for adding, removing, or
  reconfiguring stores at runtime; credentials handling matches the LLM-API-key pattern
  (env / operator secret store), not the `tool_backends` DB-config pattern. Reads route
  via the `store_id` on each ref; writes route to the env-nominated default. Rationale:
  keeps `open-bbcd`'s "no local disk state" invariant intact (Postgres bloat and media
  workloads don't mix); keeps blob-store credentials out of Postgres entirely (a stronger
  security posture than `tool_backends.config` — one fewer secret class in the DB); makes
  install reproducible from a single env manifest (CI/CD / Helm-friendly); keeps locked
  sessions stable via the invariant "refs stay resolvable while any locked session
  references them" (see [`ddd/contexts/artifacts.md`](ddd/contexts/artifacts.md)) — refs
  are exported unchanged, while eval replay of artifact-bearing sessions is text-only until
  an artifact-aware replay follow-up.
  Trade-off: adding or reconfiguring a store requires a redeploy — accepted because store
  changes are rare and refs remain resolvable across default flips. Date: 2026-09-28.
- **Multi-agent = an in-process agent tool over pinned versions; no inter-agent protocol.**
  A version with `agent_versions.agent_tool_enabled = true` gets one `agent` tool whose
  `subagent` enum is its allow-list (`agent_version_subagent` rows: pinned
  `target_version_id`, tool-facing `name`, prompt `note`). Calling it runs the target
  version's full turn loop — its own prompts, its own tool wiring, its own agent tool if
  enabled — inside `open-bbcd`, with a fresh context holding only the caller's `prompt`
  (text only). The caller gets back the sub-agent's final text as the tool result; no
  artifacts cross the agent-tool boundary in either direction and artifact scope is per
  session. Each spawn persists as a **child session** linked to
  its parent (`parent_session_id`, `parent_tool_call_id`). Config is **version-scoped on
  the caller** (the caller's prompt decides how the tool is used, so it varies per version
  like MCP attachments do) and **targets are pinned versions**, not agents, so BO chat,
  evals, and training replay the exact same topology. Guardrails: DAG-only topology
  (cycle rejection at save), `AGENT_TOOL_MAX_DEPTH`, `AGENT_TOOL_MAX_PARALLEL`. Rationale:
  the Claude Code sub-agent pattern is proven in practice, needs no new wire protocol, and
  reuses the existing session / message / artifact / tool-dispatch machinery; A2A or an
  MCP-per-agent layer would add a network hop, a second auth surface, and a protocol
  dependency for what is an in-process call. Trade-off: upgrading a worker requires a new
  caller version (pin bump) — accepted for reproducibility. Date: 2026-09-30.
- **Multi-provider LLM access in `open-bbcd` goes through the embedded Bifrost Go SDK.**
  `open-bbcd` keeps its provider-agnostic `llm.LLM` interface. A second adapter wraps the
  Bifrost Go SDK (`github.com/maximhq/bifrost/core`, Apache-2.0) in-process.
  `OPENBBC_LLM_ADAPTER` selects `anthropic` (direct, default) or `bifrost` at boot.
  On `bifrost`:
  - `OPENBBC_DEFAULT_MODEL` is `<provider>/<model>` and stays deployment-global, shared by
    BO chat, deployed runtime and sub-agents.
  - Provider keys come from env only.
  - Bifrost fallbacks and load-balancing are not used yet.
  - v1 accepts only key-only providers on a fixed allow-list (anthropic, openai, gemini,
    mistral, groq, cohere, openrouter, deepseek, xai, cerebras). Cloud-credential
    (Azure OpenAI, Bedrock, Vertex) and keyless self-hosted providers are refused at
    boot.

  New providers are added through Bifrost (by extending the allow-list), not by writing
  new `llm.LLM` implementations. The direct Anthropic adapter is kept as an env-selectable alternative
  and stays the default. `aikdm` is unchanged and keeps Google ADK + LiteLLM.

  Rationale: one adapter gives access to 20+ providers through a single normalised
  request/stream surface, so `open-bbcd` no longer grows one adapter per provider. As an
  in-process Go library it keeps the single-binary, no-sidecar deployment, adds no
  network hop and no second secret surface, and fits the env-only credentials pattern.

  Trade-offs:
  - Bifrost becomes a core dependency whose request/stream schema `open-bbcd` must track.
  - Provider-specific features (e.g. Anthropic PDF document blocks) are only as rich as
    Bifrost's normalisation. MIMEs the Bifrost adapter cannot render natively fall back
    to the existing `TextSurrogate`.

  Date: 2026-10-07.

<!-- migrated from _migration-quarantine/ARCHITECTURE.md § MCP wiring, § Feedback + datasets, § Evals, § Docker deployment, DESIGN.md, PRODUCTION.md § 1a Docker Compose, § 1b Standalone containers, § 6 Batch operations on 2026-09-28. Updated 2026-09-28 for OpenBBC PR #50. Updated 2026-09-28 for artifact-support — added locked decision for pluggable artifact-store adapter. Updated 2026-10-01 for sync-deployed-runtime-artifacts — two artifact legs, per-session artifact tables, bytes rule scoped to an enabled registry, text-only eval replay, text-only agent tool. Updated 2026-10-07 for bifrost — added Bifrost LLM-adapter locked decision. Updated 2026-10-07 for sync-bifrost — v1 key-only provider allow-list. -->

## Open questions

- **Auth middleware bundle.** Ship an optional JWT verifier / mTLS shim so operators aren't
  forced to run a gateway. Blocked on picking a first-supported scheme.
- **Header pass-through on deployed sessions.** Backoffice chat has per-session, per-backend
  `header_overrides`; deployed runtime does not. Extension needs a new column on
  `deployed_sessions` and a `POST /deployed/{agent_id}/sessions/{id}/headers` route.
- **Multi-replica-safe migrations.** Helm chart ships open-bbcd as a `Deployment` and each pod
  runs migrations on boot via embedded `goose` — replicas racing on migrations is still open.
  Fix: switch to goose's `Provider` API with `SessionLocker`, or move migrations to a `Job`
  hook the chart runs pre-install/pre-upgrade.
- **Timeout-based reset of stuck IN_PROGRESS items.** If a batch script dies mid-run, evals /
  training sessions stay IN_PROGRESS forever; today the fix is manual DB update.
- **Agent operator / multi-tenant runtime.** Roadmap mentions an operator pattern for
  multi-agent deployments; unscoped. (In-process multi-agent topologies are now covered
  by the agent tool — this item is about running agents as separately-scaled workloads.)
- **Agent-tool notes in the training patch surface.** Training copies the parent's
  agent-tool config (`agent_tool_enabled` + `agent_version_subagent` rows, including
  `note`) verbatim. Whether the teacher LLM may also patch `note` (it is prompt guidance)
  is undecided.
- **Pinned-target bump ergonomics.** Upgrading a worker means creating a new caller
  version whose binding points at the new target. A "bump pinned targets to latest
  READY" action (and a view of which callers pin a given version) is unscoped.
- **Per-turn LLM budget for multi-agent turns.** Depth and parallelism are capped; total
  token / cost spend per root turn is not. A budget knob is unscoped.
- **Per-version model selection.** The model is deployment-global (`OPENBBC_DEFAULT_MODEL`).
  Bifrost makes per-version (or per-sub-agent) provider/model choice cheap to wire, but
  that needs an `agent_versions` column and eval/training parity rules. Unscoped.
- **Bifrost fallbacks / load-balancing.** Not enabled. Whether to expose Bifrost's
  fallback chain and multi-key weighting via env is unscoped.
- **Retiring the direct Anthropic adapter.** It is kept as the default and as a fallback
  path. Once the Bifrost adapter reaches parity (streaming tool use, native image/PDF
  rendering), whether to drop it and make `bifrost` the only adapter is undecided.
- **Runtime vs eval model parity.** `aikdm` still uses LiteLLM, so with a non-Anthropic
  Bifrost model the BO / deployed runtime and `aikdm evaluate` may run through different
  client stacks. Moving `aikdm` onto the same model routing is unscoped.

<!-- migrated from _migration-quarantine/PRODUCTION.md § 8 Known gaps, ARCHITECTURE.md § Docker deployment future, DESIGN.md § Out of Scope on 2026-09-28. Updated 2026-09-28 for OpenBBC PR #50 (Helm chart + GHCR publish removed from roadmap). Updated 2026-09-30 for multiagent-tools — replaced single-agent scope assumption, added agent-tool locked decision and three open questions. Updated 2026-10-07 for bifrost — added four LLM-adapter open questions. -->
