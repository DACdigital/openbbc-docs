# Non-functional requirements

## Availability

Single-instance deployment is the shipping target today: one `open-bbcd` binary + one Postgres.
Migrations auto-apply on boot via embedded `goose`. Multi-replica deployments are not
migration-safe yet — running >1 replica requires a pre-deploy `open-bbcd migrate` job that exits
0 before `serve` replicas start.

<!-- migrated from _migration-quarantine/PRODUCTION.md § 1 Deploying open-bbcd, § 8 Known gaps, ARCHITECTURE.md § Docker deployment on 2026-09-28 -->

<!-- ARCH_GAP: no quantitative availability target sourced.
     Section: Availability
     Fill with: uptime SLO (e.g. 99.5%), MTTR target, planned maintenance windows.
     See: .claude/skills/check-setup/arch-schema.md#nfrs -->

## Performance

Training epochs are minutes-per-epoch — the recommended cron cadence for
`process_pending_trainings.sh` is `*/15` because faster invocations just no-op faster behind
the `flock`. Evals are session-count-bound; the score formula is a global pass-rate
`sum(passed_criteria) / sum(total_criteria)` across all sessions.

<!-- migrated from _migration-quarantine/PRODUCTION.md § 6 Batch operations, ARCHITECTURE.md § Evals on 2026-09-28 -->

<!-- ARCH_GAP: no p50/p95/p99 targets for AG-UI turn latency, chat turn latency, MCP tool-call latency, or eval throughput sourced.
     Section: Performance
     Fill with: concrete latency + throughput targets per surface (deployed AG-UI turn, backoffice chat, MCP tool call, eval run wall-clock).
     See: .claude/skills/check-setup/arch-schema.md#nfrs -->

## Security

`open-bbcd` ships **auth-agnostic**: no built-in authentication or authorization on any route.
Every operator must front it with an auth gateway. The deployed runtime accepts `user_id` from
the request body/query as the sole scope; `open-bbcd` trusts that value unconditionally, which
is only safe when a gateway has already verified the caller and rewritten `user_id` to the
verified identity.

Session-scoping defence in depth:
- `GET /deployed/{agent_id}/sessions/{id}?user_id=X` returns `404` (not `403`) on `user_id`
  mismatch, blocking session-id enumeration across users.
- Backoffice chat sessions and eval runs both carry per-session `header_overrides` that get
  merged into outbound MCP calls (`chat_sessions.backend_header_overrides` migration 016;
  `evals.header_overrides` migration 023). Deployed-runtime sessions do **not** yet carry
  header overrides — auth to backend MCPs from production traffic uses static server-to-server
  credentials only.

Recommended gateway pattern: terminate auth at ingress, rewrite `user_id` to the verified
identity, forward to `open-bbcd`. Prefer network reachability restrictions (private VPC / VPN /
SSO) for the backoffice surface over per-route allowlisting.

<!-- migrated from _migration-quarantine/PRODUCTION.md § 4 Headers, § 5 Auth model on 2026-09-28 -->

## Compliance

<!-- ARCH_GAP: source describes no regulatory regime (GDPR / HIPAA / SOC2 / ISO27001 / PCI etc.) applying to OpenBBC itself.
     Section: Compliance
     Fill with: applicable regimes with scope statement, data classes affected, audit obligations.
     See: .claude/skills/check-setup/arch-schema.md#nfrs -->

## Observability

<!-- ARCH_GAP: no observability targets sourced. Source mentions structured JSON errors on aikdm stderr (`{"error":"<kind>","details":"<msg>"}`) and cron scripts log to stdout with ISO-8601 timestamps, but no metrics/tracing/log-aggregation target.
     Section: Observability
     Fill with: metrics (RED / USE), traces (OpenTelemetry?), log aggregation endpoint, alerting rules, SLIs/SLOs.
     See: .claude/skills/check-setup/arch-schema.md#nfrs -->
