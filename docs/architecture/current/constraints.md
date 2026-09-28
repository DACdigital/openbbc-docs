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

- **Postgres 15+ required.** `goose` migrations embedded (`//go:embed`) currently at
  `024_training_sessions`.
- **Go 1.22+, `database/sql` + `lib/pq`.** Runtime image is
  `gcr.io/distroless/static-debian12:nonroot`, CGO off.
- **Python 3.12+ for `aikdm`, managed with `uv`.** Multi-provider LLM via Google ADK +
  LiteLLM (Anthropic, OpenAI, Gemini).
- **At most one `DEPLOYED` version per agent chain** (migration 011, DB-enforced singleton).
  Deploying a new version implicitly rotates the previous one.
- **At most one `DRAFT` per dataset** (migration 019, partial unique index).
- **At most one `PENDING` or `IN_PROGRESS` training session per eval** (migration 024, partial
  unique index `idx_ts_one_active_per_eval`).
- **A session belongs to at most one dataset** — enforced at the repo layer, not the schema
  (migration 020 dropped schema uniqueness to allow cross-version reuse within one dataset).
- **`chat_message_feedback.judge_criteria` must be non-empty on every session's feedback rows
  before dataset close-draft succeeds.**
- **Discovery-storage volume required.** `DISCOVERY_STORAGE_DIR` (default `/data/discovery`)
  must be a persistent volume; it holds uploaded discovery zips referenced by every agent
  version.
- **Multi-replica deployments are not migration-safe** — a pre-deploy `open-bbcd migrate` job
  must exit 0 before `serve` replicas start. Currently OK for single-instance compose.
- **`open-bbcd` ships auth-agnostic** — no route enforces authentication or authorization.
  Every operator must front it with a gateway; direct exposure is unsafe.
- **`deployed_sessions` cannot carry per-session header overrides today** — only chat and eval
  paths do.

<!-- migrated from _migration-quarantine/PRODUCTION.md § 1, § 4, § 5, § 8, ARCHITECTURE.md § MCP wiring, § Feedback + datasets, § Evals, § Training sessions, § Chat header overrides, § Docker deployment, DESIGN.md § Tech Stack on 2026-09-28 -->
