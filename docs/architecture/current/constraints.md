# Constraints

## Regulatory regimes

<!-- ARCH_GAP: source describes no regulatory regime applying to OpenBBC as shipped.
     Section: Regulatory regimes
     Fill with: applicable regimes (GDPR / HIPAA / SOC2 / ISO27001 / PCI / …) with scope statements (which surfaces, which data classes).
     See: .claude/skills/check-setup/arch-schema.md#constraints -->

## Compliance obligations

<!-- ARCH_GAP: no compliance obligations sourced. PRODUCTION.md notes hard-tenant isolation is not supported ("run one open-bbcd per tenant"), but no explicit obligation is stated.
     Section: Compliance obligations
     Fill with: obligations per regime (audit logging, data residency, retention, right-to-erasure, breach notification).
     See: .claude/skills/check-setup/arch-schema.md#constraints -->

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

<!-- migrated from _migration-quarantine/PRODUCTION.md § 1, § 4, § 5, § 8, ARCHITECTURE.md § MCP wiring, § Feedback + datasets, § Evals, § Training sessions, § Chat header overrides, § Docker deployment, DESIGN.md § Tech Stack on 2026-09-28. Updated 2026-09-28 for OpenBBC PR #50 (mig 025 PENDING + 026 discovery_zip + Helm chart + aikdm-runner + published GHCR images). -->
