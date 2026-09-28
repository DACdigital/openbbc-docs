# fill-arch-gaps — resolve 12 ARCH_GAP markers left by the schema migration

**Date**: 2026-09-28
**Codename**: fill-arch-gaps

**Driver**: `/check-setup` reported check 4b (non-placeholder) failing on 12 ARCH_GAP
markers left by the 2026-09-28-arch-schema-migration brownfield reshape. Walked through
each with the DM/PL and decided N/A-vs-fill per marker.

**Decision**: apply 10 `N/A because <reason>` escape-hatches and fill 2 sections with
concrete content, so `/check-setup` check 4b passes.

**Rationale** (per cluster):

- **Regulatory / compliance (4 markers)** — OpenBBC ships as self-hosted open-source
  middleware; DAC does not operate any deployment. Regulatory scope is a deployer
  concern; obligations flow to whoever runs a deployment against their own data classes.
  → 3 × N/A (`nfrs.md § Compliance`, `constraints.md § Regulatory regimes`,
  `constraints.md § Compliance obligations`) + 1 × fill
  (`bizbok/information-map.md § Regulatory tag column` with the coarse three-value
  scheme `system-metadata / user-content (deployer-classified) / credentials`, plus a
  new dedicated *Chat header override* row for symmetry with the existing *Eval header
  override* row).
- **NFR quantitative targets (2 markers)** — SLOs are operator-set per deployment; the
  shipped code carries no runtime commitment. Deployment constraints (single-replica
  default, migrations-on-boot race) already live in `constraints.md` + `assumptions.md`;
  drainer cron cadences live in `bizbok/capabilities.md`. → 2 × N/A
  (`nfrs.md § Availability`, `nfrs.md § Performance`).
- **Observability (1 marker)** — source describes something concrete on the logging
  axis; metrics/tracing/alerting are deliberately not shipped. → fill
  (`nfrs.md § Observability`) with what the platform actually provides: aikdm structured
  JSON stderr on non-zero exits, `open-bbcd` and cron drainer ISO-8601 stdout, retention
  owned by cron/journald or kubelet; explicit "metrics/traces/alerts not shipped —
  deployer attaches"; health-probe contract for the `open-bbcd healthcheck` subcommand.
- **Domain events across 5 DDD contexts (5 markers)** — OpenBBC deliberately has no
  event bus / outbox / message topics. State transitions are Postgres-only; downstream
  consumers poll REST (`GET /agent_versions.json?status=PENDING`,
  `GET /evals.json?status=PENDING`, `GET /training-sessions.json?status=PENDING`) — the
  drainers work by polling, that IS the shipped design. → 5 × N/A on
  `ddd/contexts/{agent-lifecycle, feedback-datasets, evaluation, training,
  deployed-runtime}.md § Domain events`. The AG-UI wire chunks in `deployed-runtime`
  (`RUN_STARTED`, `TEXT_MESSAGE_*`, `TOOL_CALL_*`, `TURN_END`) are transport-layer
  message framing, not domain events another context subscribes to.

**Alternatives rejected**:
- Fabricate a candidate event list per context — rejected: schema-forbidden invention
  ("never invent"). N/A with a reason is the correct escape hatch when the platform
  intentionally lacks the mechanism.
- Ship default SLO numbers (99.5% uptime, p95 500ms, etc.) — rejected: same, would be
  invention. Real SLOs are operator-set.

**Impact**:
- edited: `docs/architecture/current/nfrs.md` (Availability + Performance + Compliance
  N/A; Observability filled)
- edited: `docs/architecture/current/constraints.md` (Regulatory regimes + Compliance
  obligations N/A)
- edited: `docs/architecture/current/bizbok/information-map.md` (Regulatory-tag column
  filled with 3-value scheme; Chat header override row added; Discovery-zip description
  corrected to reflect migration 026 inlining)
- edited: `docs/architecture/current/ddd/contexts/{agent-lifecycle,feedback-datasets,evaluation,training,deployed-runtime}.md` § Domain events (5 × N/A)
- + docs/architecture/logs/2026-09-28-fill-arch-gaps/README.md

**Links**:
- `.claude/skills/check-setup/arch-schema.md#arch-gap-marker-format`
- `.claude/skills/check-setup/arch-schema.md#non-placeholder-rule`
- 2026-09-28-arch-schema-migration/README.md (source of the ARCH_GAPs this entry resolves)
