# Non-functional requirements

## Availability

Shipping deployment path as of OpenBBC PR #50 is Kubernetes via the Helm chart at
`deploy/helm/openbbc/`: open-bbcd `Deployment` (default one replica) + `Service` + optional
`Ingress`, optional in-cluster Postgres `StatefulSet` (or point `externalDatabase.url` at a
managed DB), and three `CronJob`s (alphas, evals, trainings) running the `aikdm-runner`
image.

`open-bbcd` keeps no local disk state (discovery zip lives in Postgres per migration 026),
so scaling out the app tier is a matter of replica count — **except** that migrations still
run on boot via embedded `goose` and racing pods can corrupt the migration state. Multi-replica
deployment is therefore not migration-safe today; the chart ships one replica by default and
a proper fix (goose `Provider` + `SessionLocker`, or a pre-install migrations `Job`) is
tracked as an open question in `assumptions.md`.

Docker Compose (`docker-compose.yml` at the OpenBBC repo root) is a **local-dev-only**
setup, not a shipping path — see [`deployment.md § Environments`](../c4/deployment.md#environments)
for the full environment table.

<!-- migrated from _migration-quarantine/PRODUCTION.md § 1 Deploying open-bbcd, § 8 Known gaps, ARCHITECTURE.md § Docker deployment on 2026-09-28. Updated 2026-09-28 for OpenBBC PR #50 (Helm chart shipped; open-bbcd stateless per mig 026). Corrected 2026-09-28 to drop Docker Compose from the shipping-paths list — it is a local-dev tool only. -->

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
