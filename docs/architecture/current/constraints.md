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
  `026_agent_discovery_zip`.
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
- **`ARTIFACT_MAX_UPLOAD_MB` env var caps per-artifact upload size.** No default is shipped
  — deployers set this explicitly at install time (rationale: any single default would be
  wrong for either text-heavy or media-heavy deployments). Enforced at the upload boundary
  on both `POST /chat-sessions/{id}/artifacts` and
  `POST /deployed/{agent_id}/sessions/{id}/artifacts`; retrieval is not capped so refs
  stored under a lower prior cap remain readable.
- **Exactly one `artifact_stores.is_default = true` at any time** — partial unique index
  (`WHERE is_default = true`). Fresh deployments start with zero configured stores; the
  first artifact-carrying turn is rejected until an admin picks a default via
  `POST /artifact-stores/{id}/set-default`.
- **Artifact bytes must not be stored in Postgres.** `chat_messages.content` and
  `deployed_messages.content` JSONB carry only `artifact_ref` pointers (`{store_id, uri,
  mime, size_bytes, sha256}`); blob bytes flow through the `artifact-store-adapter` to the
  deployer's Object store. Repo-layer guards reject any attempt to inline base64 payloads
  in a content block.
- **Artifact-store kinds are versioned via `artifact_stores.kind` string.** First-shipped
  kind: `s3_compatible`. Adding a new kind is a code + migration change (register the
  adapter + declare its `config` JSONB schema) — not a runtime plug-in surface.

<!-- migrated from _migration-quarantine/PRODUCTION.md § 1, § 4, § 5, § 8, ARCHITECTURE.md § MCP wiring, § Feedback + datasets, § Evals, § Training sessions, § Chat header overrides, § Docker deployment, DESIGN.md § Tech Stack on 2026-09-28. Updated 2026-09-28 for OpenBBC PR #50 (mig 025 PENDING + 026 discovery_zip + Helm chart + aikdm-runner + published GHCR images). Updated 2026-09-28 for artifact-support — added ARTIFACT_MAX_UPLOAD_MB, is_default invariant, no-bytes-in-Postgres rule, kind-versioning rule. -->
