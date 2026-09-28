# Deployment

## Environments

- **Local Go** — `make build && ./bin/open-bbcd` with `$DATABASE_URL` pointing at Postgres.
  Migrations auto-apply on boot (goose embedded via `//go:embed`).
- **Docker Compose (single-instance)** — top-level `docker-compose.yml` brings up
  `postgres` + `open-bbcd`; the plain `aikdm` image sits behind the `aikdm` compose profile
  (`docker compose --profile aikdm run --rm aikdm …`). Suited for local dev + single-node
  production. After migration 026 the compose file no longer needs the `discovery-data`
  named volume.
- **Kubernetes (Helm chart `deploy/helm/openbbc/`)** — shipping k8s path as of OpenBBC PR
  #50. Ships one `Deployment` replica of `open-bbcd` + `Service` + optional `Ingress`, an
  optional in-cluster Postgres `StatefulSet` (or point `externalDatabase.url` at a managed
  DB), and three `CronJob`s running the `aikdm-runner` image: alphas
  (`*/5 * * * *`, mounts `DATABASE_URL`), evals (`*/10 * * * *`), trainings
  (`*/15 * * * *`). Chart default image repositories point at
  `ghcr.io/dacdigital/openbbc/{open-bbcd,aikdm-runner}` — pick a `TAG`
  (`pr-<num>`, `main`, `sha-<short>`, semver, `latest`) and `--set image.tag=$TAG`.

<!-- migrated from _migration-quarantine/PRODUCTION.md § 1 Deploying, § 8 Known gaps, ARCHITECTURE.md § Docker deployment on 2026-09-28. Updated 2026-09-28 for OpenBBC PR #50 (Helm chart shipped; discovery-data volume removed per mig 026). -->

## Network zones

- **Public zone** — the operator's gateway (Envoy / nginx / API gateway) terminating
  customer auth. Only surface exposed to the internet.
- **App zone** — `open-bbcd` (`:8080`); reachable from the gateway and from the internal
  automation surface (operator, cron scripts, k8s CronJobs). Backoffice UI lives in this
  zone. No local disk state after migration 026 — the pod is stateless.
- **Data zone (trust boundary)** — `postgres` (`:5432`) with `postgres-data` persistent
  volume; only `open-bbcd` and (on the alpha-drainer path) `aikdm-runner`'s `seed_bundle.py`
  reach it.
- **Job zone** — `aikdm` (compose profile, one-shot) and `aikdm-runner` (k8s CronJob pods).
  Mounts LLM keys as Secrets; reaches `open-bbcd`'s REST + the LLM providers. The
  alpha-drainer CronJob additionally mounts `DATABASE_URL`, so that pod crosses into the
  Data zone.
- **External integration zone** — LLM providers (Anthropic / OpenAI / Gemini) and the
  client's backend (either as an `http_endpoint` REST bridge or as an `mcp_client` MCP
  proxy). Reached from the app zone + job zone by outbound HTTPS / SSE / Streamable HTTP.

Every container in [`containers.md`](containers.md) is placed:

| Container | Zone(s) |
|-----------|---------|
| [`flow-map-compiler`](containers.md#flow-map-compiler) | Discovery author's machine (outside all zones); output enters App zone via wizard upload |
| [`open-bbcd`](containers.md#open-bbcd) | App zone |
| [`aikdm`](containers.md#aikdm) | Job zone (compose profile only) |
| [`aikdm-runner`](containers.md#aikdm-runner) | Job zone (all three drainers); alpha drainer additionally crosses into Data zone |
| [`postgres`](containers.md#postgres) | Data zone |

<!-- migrated from _migration-quarantine/ARCHITECTURE.md § Docker deployment, PRODUCTION.md § 1, § 6 Batch operations, § 7 Provider LLM keys on 2026-09-28. Updated 2026-09-28 for OpenBBC PR #50 (added aikdm-runner CronJobs; app zone stateless per mig 026; alpha drainer crosses Data zone). -->

## Trust boundaries

- **Public ↔ App (via gateway)** — the gateway is the only allowed ingress from the public
  zone. `open-bbcd` trusts the `user_id` the gateway forwards; direct exposure of `open-bbcd`
  to the public zone bypasses the entire auth model.
- **App ↔ Data** — Postgres reachable only from `open-bbcd` on a private network, plus the
  alpha-drainer pod (`aikdm-runner` running `process_pending_alphas.sh` →
  `generate_alpha.sh` → `seed_bundle.py`) which is granted `DATABASE_URL` explicitly.
  Marked as a dashed trust boundary in the diagram.
- **App ↔ External integration** — outbound tool calls to the client backend (as
  `http_endpoint` REST bridge OR `mcp_client` MCP proxy) and outbound HTTPS to LLM providers
  cross the boundary. Tool calls carry server-to-server credentials bound to the
  `tool_backends` row (plus per-backend `header_overrides` in the BO chat and eval paths —
  not on the deployed path).
- **Job ↔ App** — `aikdm-runner` reaches `open-bbcd` via REST for all three drainers.
  Compose-profile `aikdm` uses the same path.

## Diagram

```mermaid
flowchart TB
    subgraph PUBLIC[Public zone]
        USER[End user]
        GW[Operator's gateway]
    end

    subgraph APP[App zone]
        OBBCD[open-bbcd<br/>stateless]
    end

    subgraph JOB[Job zone]
        AIKDMR_A[aikdm-runner<br/>alphas CronJob]
        AIKDMR_E[aikdm-runner<br/>evals CronJob]
        AIKDMR_T[aikdm-runner<br/>trainings CronJob]
        AIKDM[aikdm<br/>compose profile]
    end

    subgraph DATA[Data zone — trust boundary]
        DB[(postgres)]
    end

    subgraph EXT[External integration zone]
        BE[Client backend<br/>REST or MCP]
        LLM[LLM providers<br/>Anthropic / OpenAI / Gemini]
    end

    USER --> GW
    GW --> OBBCD
    OBBCD --> DB
    OBBCD --> BE
    OBBCD --> LLM

    AIKDMR_A --> OBBCD
    AIKDMR_E --> OBBCD
    AIKDMR_T --> OBBCD
    AIKDM --> OBBCD

    AIKDMR_A -. seed_bundle.py .-> DB

    AIKDMR_A --> LLM
    AIKDMR_E --> LLM
    AIKDMR_T --> LLM
    AIKDM --> LLM

    classDef trust stroke-dasharray: 5 5
    class DATA trust
```

<!-- migrated from _migration-quarantine/ARCHITECTURE.md § Docker deployment, PRODUCTION.md § 1 Deploying, § 5 Auth model, § 7 Provider LLM keys on 2026-09-28. Updated 2026-09-28 for OpenBBC PR #50 — three CronJobs, stateless open-bbcd, alpha drainer's dashed line into Data zone. -->

## Threat model

| Boundary | Threat | Mitigation |
|----------|--------|-----------|
| Public ↔ App | **Spoofing** — attacker forges `user_id` in a request body/query to impersonate a real user. | Operator's gateway must verify auth and **rewrite** `user_id` to the verified identity before forwarding; never accept the client's `user_id` verbatim. |
| Public ↔ App | **Information disclosure** — attacker enumerates session ids across users. | `GET /deployed/{agent_id}/sessions/{id}?user_id=X` returns `404` (not `403`) on mismatch, blocking existence leaks. |
| Public ↔ App | **Elevation of privilege** — attacker reaches the backoffice UI directly, bypassing the gateway. | Restrict network reachability of the backoffice surface (private VPC / VPN / employee SSO) — the surface is too large for per-route allowlisting. |
| App ↔ External integration | **Tampering / repudiation** — MCP call to the client backend attributed to the wrong tenant. | Backoffice chat + eval paths carry `header_overrides` merged into MCP calls, allowing auth tokens / tenant scoping / correlation ids to flow through. Deployed-runtime path today uses server-to-server credentials only — mitigate via one MCP backend per tenant or per-user auth inside the MCP backend against a shared secret + gateway-injected header. |
| App ↔ Data | **Denial of service** — misbehaved migration on multi-replica boot races Postgres. | Currently mitigated by the Helm chart shipping one `open-bbcd` `Deployment` replica by default. Scaling `openbbcd.replicaCount > 1` needs the follow-up work tracked in `assumptions.md`: goose `Provider` + `SessionLocker`, or a pre-install migrations `Job` in the chart. |
| Public ↔ App | **Denial of service** — no rate limits on the deployed runtime. | ARCH_GAP — no rate-limit / abuse-control policy sourced. Rely on gateway for now. |
| App ↔ External integration | **Provider credential leak** — LLM provider API key exposure. | Keys must come from platform secret store in production, not `.env`. `open-bbcd` and `aikdm` read `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, `GEMINI_API_KEY` from env only. |

<!-- migrated from _migration-quarantine/PRODUCTION.md § 4 Headers, § 5 Auth model, § 7 Provider LLM keys, § 8 Known gaps on 2026-09-28 -->
