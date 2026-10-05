# Constraints

## Regulatory regimes

N/A because OpenBBC ships as self-hosted open-source middleware and DAC does not operate
any deployment; regulatory scope is a deployer concern (see
[`nfrs.md § Compliance`](nfrs.md#compliance)).

## Compliance obligations

N/A because OpenBBC ships as self-hosted open-source middleware and DAC does not operate
any deployment; audit logging, data residency, retention, right-to-erasure, and breach
notification obligations flow to whoever operates a deployment against their own data
classes (see [`nfrs.md § Compliance`](nfrs.md#compliance) and
[`bizbok/information-map.md`](bizbok/information-map.md) for the concept inventory).

## Hard technical limits

- **Postgres 15+ required.** `goose` migrations embedded (`//go:embed`), currently at
  `029_agent_tool`.
- **Go 1.22+, `database/sql` + `lib/pq`.** Runtime image is
  `gcr.io/distroless/static-debian12:nonroot`, CGO off.
- **Python 3.12+ for `aikdm`, managed with `uv`.** Multi-provider LLM via Google ADK +
  LiteLLM (Anthropic, OpenAI, Gemini).
- **`aikdm-runner` image is the k8s drainer runtime** (`Dockerfile.aikdm-runner`): bash +
  curl + tini + python 3.12 + uv + `aikdm/` source + `scripts/`. Runs as uid 65532
  (`runner`). Consumed by Helm chart CronJobs (alphas / evals / trainings).
- **`agent_versions.status` state machine:** `INITIALIZING → PENDING → READY` via the alpha
  drainer (migration 025); `READY → TRAINING → READY` on hill-climb; `READY → DEPLOYED` on
  deploy. `INITIALIZING,PENDING,DRAFT,TRAINING,READY,DEPLOYED` are the enforced
  `agent_versions_status_check` values.
- **At most one `DEPLOYED` version per agent chain** (migration 011, DB-enforced singleton).
  Deploying a new version implicitly rotates the previous one.
- **At most one `DRAFT` per dataset** (migration 019, partial unique index).
- **At most one `PENDING` or `IN_PROGRESS` training session per eval** (migration 024, partial
  unique index `idx_ts_one_active_per_eval`).
- **A session belongs to at most one dataset** — enforced at the repo layer, not the schema
  (migration 020 dropped schema uniqueness to allow cross-version reuse within one dataset).
- **`chat_message_feedback.judge_criteria` must be non-empty on every session's feedback rows
  before dataset close-draft succeeds.**
- **`open-bbcd` needs no local disk state.** Discovery zip lives on `agents.discovery_zip
  BYTEA` (migration 026); the deprecated `discovery_file_path` column is retained ignored
  for reversibility. `DISCOVERY_STORAGE_DIR` is **not** required; `internal/storage/storage.go`
  has been removed. All persisted state lives in Postgres.
- **Multi-replica deployments are not migration-safe.** Each open-bbcd pod runs migrations
  on boot via embedded `goose`; concurrent boots race. The Helm chart ships a single
  `Deployment` replica of `open-bbcd` by default.
- **`open-bbcd` ships auth-agnostic** — no route enforces authentication or authorization.
  Every operator must front it with a gateway; direct exposure is unsafe.
- **`deployed_sessions` cannot carry per-session header overrides today** — only chat and eval
  paths do.
- **`AGENT_TOOL_MAX_DEPTH` (optional, default `3`, integer ≥ 1; any other value fails boot) caps sub-agent nesting.** The root
  session is depth `0`; an `agent` tool call from a session at depth `AGENT_TOOL_MAX_DEPTH`
  is refused with a tool error returned to the calling LLM (not a turn failure). Applies
  identically in `open-bbcd` (BO chat, deployed runtime) and in `aikdm evaluate` /
  `train-agent`, which read the value from `eval-input.yaml`.
- **`AGENT_TOOL_MAX_PARALLEL` (optional, default `4`, integer ≥ 1; any other value fails boot) caps concurrent sub-agent runs per
  assistant step.** When one assistant message issues several `agent` tool calls, up to
  this many run concurrently; the rest queue. Counted per session node, not globally.
- **Agent-tool topology is a DAG over pinned versions, acyclic by construction.** Only an
  `INITIALIZING` / `DRAFT` caller can save agent-tool config, and only a `READY` /
  `DEPLOYED` target can be bound. Since status only moves forward, a caller never has
  incoming edges. A binding that would make the caller reachable from its target (including
  self-binding) is also refused at the repo layer, as defence in depth. Because targets are
  pinned `agent_version_id`s, the graph is static. A version forked from a caller (prompt
  save, land, training complete) copies its config but has no incoming edges, so it cannot
  introduce a cycle.
- **Sub-agent targets must be `READY` or `DEPLOYED` at bind time.** `INITIALIZING`,
  `PENDING`, `DRAFT`, and `TRAINING` versions cannot be bound. A target does **not** need
  to be `DEPLOYED` — workers may never be user-facing. The pinned id is never re-resolved
  at call time. Agent-tool config (the checkbox and every binding) is writable only while
  the caller version is `INITIALIZING` or `DRAFT`. On any later status it is frozen and
  writes return `409`.
- **`ARTIFACT_MAX_UPLOAD_MB` env var caps per-artifact upload size.** No default is shipped
  — deployers set this explicitly at install time (rationale: any single default would be
  wrong for either text-heavy or media-heavy deployments). Enforced at the upload boundary
  on both `POST /agent_versions/{v}/chat/{s}/artifacts` and
  `POST /deployed/{agent_id}/sessions/{sid}/artifacts?user_id=X` (`413`, before any store
  call); retrieval is not capped so refs stored under a lower prior cap remain readable.
- **`ARTIFACT_MAX_PENDING` env var caps pending artifacts per session.** Optional, default
  `10`, must be ≥1; an invalid value fails boot. An upload that would exceed the cap on a
  session's not-yet-consumed artifacts is refused with `409` (a dedup hit on an
  already-pending blob does not count). Applies to both the BO and deployed upload routes.
- **Artifact stores are configured via env vars only — no REST/DB surface.** Each store is
  declared through `ARTIFACT_STORE_<ID>_KIND` plus kind-specific vars (e.g.
  `ARTIFACT_STORE_<ID>_ENDPOINT`, `_BUCKET`, `_ACCESS_KEY`, `_SECRET_KEY` for the
  `s3_compatible` kind). Registry is loaded at `open-bbcd` boot; runtime CRUD is not
  supported — reconfiguration requires a redeploy. Credentials handling matches the LLM-
  API-key pattern (env / operator secret store), not `tool_backends`.
- **Exactly one `ARTIFACT_STORE_DEFAULT` at boot.** Nominates which store new writes go
  to. If unset and any `artifact_ref`-emitting capability is exercised, the request is
  rejected with a clear error. Refs already stored under a previous default remain
  resolvable via their embedded `store_id` — flipping the default does not break history.
- **`<ID>` slug in env-var names is the `store_id` on refs.** Refs carry the slug
  verbatim; renaming a store id between deploys breaks every historical ref that pointed
  at it. Ids must be stable across the lifetime of any blob any locked session references.
- **Artifact bytes must not be stored in Postgres when the artifact registry is enabled.**
  `chat_messages.content` and `deployed_messages.content` JSONB carry only `artifact_ref`
  pointers (`{store_id, uri, mime, size_bytes, sha256}`); blob bytes flow through the
  `artifact-store-adapter` to the deployer's Object store. Repo-layer guards reject any
  attempt to persist rendered media bytes in a content block. With the registry disabled,
  tool-result normalisation does not run and raw tool output (possibly carrying base64
  media) is persisted as before.
- **Artifact-store kinds are versioned via the `kind` env-var value.** First-shipped kind:
  `s3_compatible`. Adding a new kind is a code change (register the adapter + declare its
  env-var schema) — not a runtime plug-in surface.
- **`ARTIFACT_SIGNED_URL_TTL_SECONDS` (optional, default `300`) caps presigned-URL TTL.**
  Applies only when an adapter's `PreferredDelivery()` is `SignedURL` — the framework passes
  this value to `adapter.Sign` as `ttl`. Deployers may tune it; the default `300s` (five
  minutes) balances CDN-cacheable link lifetime against replay risk. Ignored by adapters
  whose `PreferredDelivery()` is `Bytes` (proxied read).

<!-- migrated from _migration-quarantine/PRODUCTION.md § 1, § 4, § 5, § 8, ARCHITECTURE.md § MCP wiring, § Feedback + datasets, § Evals, § Training sessions, § Chat header overrides, § Docker deployment, DESIGN.md § Tech Stack on 2026-09-28. Updated 2026-09-28 for OpenBBC PR #50 (mig 025 PENDING + 026 discovery_zip + Helm chart + aikdm-runner + published GHCR images). Updated 2026-09-28 for artifact-support — added ARTIFACT_MAX_UPLOAD_MB, is_default invariant, no-bytes-in-Postgres rule, kind-versioning rule. Updated 2026-09-30 for multiagent-tools — added AGENT_TOOL_MAX_DEPTH, AGENT_TOOL_MAX_PARALLEL, DAG topology rule, target-status rule. Updated 2026-10-01 for sync-deployed-runtime-artifacts — migration head 028, nested upload paths, ARTIFACT_MAX_PENDING, bytes rule scoped to an enabled registry. Updated 2026-10-05 for sync-multiagent-feature — migration head 029, AGENT_TOOL_MAX_* validation, DAG acyclic by construction, config frozen outside INITIALIZING/DRAFT. -->
