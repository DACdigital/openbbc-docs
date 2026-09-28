# arch-schema-migration — reshape docs/architecture/current/ into schema

**Date**: 2026-09-28
**Codename**: arch-schema-migration

**Driver**: template pulled schema-enforcing /check-setup — migrate to conform

**Decision**: reshape docs/architecture/current/ from freestyle into the 20-file schema
defined in .claude/skills/check-setup/arch-schema.md

**Rationale**: /check-setup now enforces the schema; existing freestyle content (three
files adopted from DACdigital/OpenBBC docs/) must be bucketed into schema shape (or
quarantined for human review) before /check-setup can pass.

**Alternatives rejected**:
- Keep freestyle — rejected: /check-setup fails on missing required files.
- Manual reshape — rejected: LLM bucketing preserves provenance and structure at scale.

**Impact**:
- reshaped: docs/architecture/current/** (20 files written per schema — 4 top-level, 5 bizbok, 3 ddd + 6 context files, 6 c4)
- quarantined: docs/architecture/current/_migration-quarantine/{ARCHITECTURE.md,DESIGN.md,PRODUCTION.md} (source retained as audit trail)
- + docs/architecture/logs/2026-09-28-arch-schema-migration/README.md

**Amendments during PR review** (2026-09-28, same day):
- Corrected the scope assumption "client backend is already wrapped by MCP" — reality is
  OpenBBC ships a built-in MCP-over-REST bridge (`tool_backends.kind = http_endpoint`) so
  clients can integrate a plain REST backend without building an MCP server. The flow-map-
  compiler skill's LOCKED anti-goal is explicit: "never generate MCP server code, never
  assume an MCP server exists".
- Corrected discovery-skill outputs: `.flow-map/` schema v2 is `AGENTS.md`, `APP.md`,
  `glossary.md`, `skills/<id>.md`, `flows/<id>.md`, `endpoints/<id>.md` — the old
  `capabilities/`, `agents/` layout from ARCHITECTURE.md was stale.
- Absorbed OpenBBC PR #50 (merged 2026-09-28):
  - migration 025 — `agent_versions.status` gains `PENDING` between `INITIALIZING` and
    `READY`; wizard Finalize is now async, drained by `scripts/process_pending_alphas.sh`.
  - migration 026 — `agents.discovery_zip BYTEA` inlines the discovery zip; `open-bbcd` is
    now stateless (no PVC / `DISCOVERY_STORAGE_DIR` required); `internal/storage/storage.go`
    removed.
  - Helm chart `deploy/helm/openbbc/` — k8s deployment path with one Deployment + three
    CronJobs (`alphas`, `evals`, `trainings`) running the new `aikdm-runner` image.
  - `Dockerfile.aikdm-runner` + `.github/workflows/publish-images.yml` — three images
    published to `ghcr.io/dacdigital/openbbc/{open-bbcd,aikdm-runner,aikdm}` on every PR /
    merge-to-main / `v*` tag.
  - New L1 capability "Batch drainer operations" + L2 capabilities
    `alpha-drainer`, `eval-drainer`, `training-drainer`, `mcp-over-rest-bridge`.
  - New `c4/containers.md` container `aikdm-runner`; deployment / data-flows / integrations
    updated for the new topology.

**Links**:
- .claude/skills/check-setup/arch-schema.md
- docs/superpowers/specs/2026-09-18-arch-current-schema-design.md
- https://github.com/DACdigital/OpenBBC/pull/50 (source of the amendments)
- bbc-discovery/flow-map-compiler/skills/flow-map-compiler/SKILL.md (source of the anti-goal correction)
