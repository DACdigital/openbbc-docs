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

N/A because OpenBBC ships as self-hosted open-source middleware and DAC does not operate
any deployment; runtime SLOs (uptime, MTTR, maintenance windows) are set by the deployer
against their own infrastructure. Shipping-time deployment constraints — single-replica
default in the Helm chart, migrations run on boot via embedded `goose` so racing pods can
corrupt migration state — live in [`constraints.md`](constraints.md) and
[`assumptions.md § Open questions`](assumptions.md#open-questions).

## Performance

Training epochs are minutes-per-epoch — the recommended cron cadence for
`process_pending_trainings.sh` is `*/15` because faster invocations just no-op faster behind
the `flock`. Evals are session-count-bound; the score formula is a global pass-rate
`sum(passed_criteria) / sum(total_criteria)` across all sessions.

<!-- migrated from _migration-quarantine/PRODUCTION.md § 6 Batch operations, ARCHITECTURE.md § Evals on 2026-09-28 -->

N/A because latency and throughput targets are operator-set per deployment; the shipped
code carries no runtime SLO commitment. What the platform does prescribe is drainer cron
cadence — alpha drainer at `*/5` (one aikdm LLM call per PENDING version, ~30–60s),
eval drainer at `*/10`, training drainer at `*/15` (minutes-per-epoch under `flock`) —
documented in [`bizbok/capabilities.md § Batch drainer operations`](bizbok/capabilities.md#l2-capabilities).
Deployers set p50/p95/p99 for deployed AG-UI turns, backoffice chat, MCP tool calls, and
eval wall-clock against their own workload.

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

**Artifact-store credentials and refs.** With the `artifact-support` capability, `open-bbcd`
reads server-to-server credentials for the deployer's Object store **from env vars**
(`ARTIFACT_STORE_<ID>_ACCESS_KEY`, `_SECRET_KEY`, and equivalents per kind) — same handling
class as LLM provider keys, not persisted in Postgres. This is a stronger posture than
`tool_backends.config`: one fewer secret class in the DB, and the credentials never leave
the deploy-time env / operator secret store. Artifact reads are session-scoped:
`GET /artifacts/{store_id}/{uri}?session_id=…&user_id=…` returns 404 on session/user
mismatch (same trust boundary as messages) — an `artifact_ref` `{store_id, uri}` pair is
not a bearer capability. When an adapter returns a presigned URL in place of proxied
bytes, the URL inherits the store's TTL and access is auditable through the deployer's
object-store logs.

<!-- migrated from _migration-quarantine/PRODUCTION.md § 4 Headers, § 5 Auth model on 2026-09-28. Updated 2026-09-28 for artifact-support — added artifact-store credentials + ref-access model. -->

## Compliance

N/A because OpenBBC ships as self-hosted open-source middleware and DAC does not operate
any deployment; regulatory scope (GDPR, HIPAA, SOC 2, ISO 27001, PCI, …) is a deployer
concern flowing through to whoever runs a deployment against their own data classes and
user population. The concepts the platform stores are tagged coarsely in
[`bizbok/information-map.md`](bizbok/information-map.md) under the *Regulatory tag* column
(`system-metadata`, `user-content (deployer-classified)`, `credentials`) so a deployer's
compliance analysis has a starting inventory.

**Artifacts add a new locus of user content — outside Postgres.** With the `artifact-support`
capability, deployers now also need to run their compliance analysis against the configured
Object store's residency, retention, right-to-erasure, encryption-at-rest, and access-log
properties. `open-bbcd` holds only refs in Postgres; blob bytes and their metadata (creation
time, byte count, MIME, checksum) live in the deployer's chosen storage backend and inherit
whatever compliance envelope that backend provides.

## Observability

### Logs

- **`aikdm` non-zero exits** — structured JSON on stderr: `{"error":"<kind>","details":"<msg>"}`.
  Exit codes: `1` unexpected, `2` input/config, `3` LLM. Stdout carries progress lines.
- **`open-bbcd` and the three cron drainers** — ISO-8601 timestamped stdout.
- **Retention** — cron/journald in Docker Compose (local dev); kubelet plus deployer's
  log aggregator (Loki, ES, CloudWatch, …) in the k8s deployment. Not shipped by the chart.

### Metrics, traces, alerts

Not shipped. Deployer's telemetry stack attaches externally (OpenTelemetry sidecar,
Prometheus exporter, log-based alerts of choice). The chart does not open a metrics port
or wire tracing.

### Health probes

`open-bbcd healthcheck` subcommand probes `http://127.0.0.1:$SERVER_PORT/health`, exits
`0` or `1`. Reads only `SERVER_PORT` so a broken `DATABASE_URL` cannot fail the probe.
Used by the container `HEALTHCHECK` and reusable for k8s `livenessProbe` /
`readinessProbe`.
