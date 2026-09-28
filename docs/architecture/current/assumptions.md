# Assumptions

## Scope assumptions

- **Client backend can be plain HTTP/REST — no pre-existing MCP server required.** At runtime
  `open-bbcd` bridges each registered backend as an MCP tool via one of two `tool_backends`
  kinds:
  - `http_endpoint` — `open-bbcd` calls a REST endpoint directly, exposing it to the agent as
    an MCP tool (the built-in MCP-over-REST bridge).
  - `mcp_client` — `open-bbcd` proxies to an existing MCP server the operator points it at.
  Either kind is a first-class shipping mode.
- **Discovery is a proposer, not an MCP-server generator.** The `flow-map-compiler` Claude
  Code skill scans the client **frontend** repo, extracts every backend call site, and emits
  `endpoints/<id>.md` files carrying `proposed: true` — HTTP method, path, params, response
  shape, proposed MCP tool name. Its LOCKED anti-goals include "never generate MCP server
  code" and "never assume an MCP server exists". Wiring those proposed endpoints into a
  runnable MCP surface is downstream engineering (or a future generator skill).
- Client frontend uses the **AG-UI protocol** for the deployed-agent chat surface. The FE
  never talks to the client backend directly — it talks to `open-bbcd`, which dispatches tool
  calls to the MCP-bridged or MCP-proxied backend.
- **Single AI agent on day one.** Multi-agent orchestration is out of scope; multiple
  concurrent deployments per agent chain are not supported.
- **`aikdm` (the Python CLI) is DB-unaware** — it talks REST to `open-bbcd` via `scripts/run_eval.sh`
  and `scripts/train_from_session.sh`. The alpha-drainer path is a scripted composition
  (`process_pending_alphas.sh` → `generate_alpha.sh` → `aikdm generate-agent` + `seed_bundle.py`)
  where `seed_bundle.py` — packaged into the `aikdm-runner` image alongside `aikdm` — writes
  the resulting bundle directly to Postgres. `aikdm` itself remains DB-unaware; the runner
  image is not.

<!-- migrated from _migration-quarantine/DESIGN.md § Assumptions, § Out of Scope, ARCHITECTURE.md § System Overview, § MCP wiring on 2026-09-28. Updated 2026-09-28 for OpenBBC PR #50 (mig 025 PENDING alpha + 026 discovery_zip + Helm chart + aikdm-runner image + published GHCR images) and flow-map-compiler skill LOCKED anti-goals. -->

## Design decisions (locked)

- **Auth-agnostic ship** — no built-in auth on any route. Operators front `open-bbcd` with
  their own gateway. Rationale: auth policy varies wildly (SSO, mTLS, API gateway, tenant
  scoping) and baking one in would push assumptions onto every operator. Date: current shipping
  design as of migration 024. Trade-off documented in `nfrs.md § Security` and
  `ddd/access-model.md`.
- **Agent-level architecture vs version-level prompts (migration 017)** — endpoint→backend
  wiring is agent-keyed (`agent_endpoint_backend`) because endpoints are structural and frozen
  on first version; MCP attachments are version-keyed (`agent_version_mcp_backend`) with an
  editable `note` because prompt guidance varies per version. Date: migration 017.
- **At most one DEPLOYED per agent chain (migration 011).** Deploying a new version implicitly
  rotates the previous one. Rationale: keep the deployed runtime unambiguous. Date: migration
  011.
- **At most one DRAFT per dataset (migration 019).** Closing a DRAFT seeds the next DRAFT with
  the CLOSED version's sessions (migration 020) so users see cumulative content, not an empty
  next version. Date: migrations 019–020.
- **Score formula is global pass-rate** — `sum(passed_criteria) / sum(total_criteria)` across
  all sessions in the eval (migration 022). Every criterion counts equally; longer sessions
  weigh proportionally more. Date: migration 022.
- **Distroless runtime image, CGO off.** `open-bbcd/Dockerfile` runtime on
  `gcr.io/distroless/static-debian12:nonroot`; the container `HEALTHCHECK` uses the
  `open-bbcd healthcheck` subcommand (no `curl` in distroless). Date: current shipping design.
- **`open-bbcd` is stateless.** Discovery zip lives in `agents.discovery_zip BYTEA`
  (migration 026); Postgres is the only stateful component. `internal/storage/storage.go`
  has been removed; the deprecated `discovery_file_path` column is retained ignored to keep
  the migration reversible. Date: migration 026 (OpenBBC PR #50).
- **PENDING is the alpha-generation state.** `agent_versions.status` now includes `PENDING`
  between `INITIALIZING` and `READY` (migration 025). Wizard Finalize transitions a root
  version `INITIALIZING → PENDING`; the async drainer (`scripts/process_pending_alphas.sh`
  → `generate_alpha.sh`) generates the bundle and lands it via `seed_bundle.py`, then
  transitions `PENDING → READY`. Date: migration 025 (OpenBBC PR #50).
- **Contract between `aikdm` and `open-bbcd` is REST + a versioned YAML schema.** No shared
  library. Section structure declared in `aikdm/schemas/prompt-v1.yaml`.
- **Three images published to GHCR** on every PR / merge-to-main / `v*` tag:
  `ghcr.io/dacdigital/openbbc/open-bbcd`, `.../aikdm-runner`, `.../aikdm`. The chart defaults
  point at these paths. Tags: `pr-<num>`, `main`, `sha-<short>`, semver. Date: OpenBBC PR #50
  (`.github/workflows/publish-images.yml`).
- **Helm chart is the shipping k8s deployment path** (`deploy/helm/openbbc/`): open-bbcd
  Deployment + Service + optional Ingress, optional in-cluster Postgres StatefulSet, and three
  CronJobs (alphas / evals / trainings) running the `aikdm-runner` image. Date: OpenBBC PR
  #50.

<!-- migrated from _migration-quarantine/ARCHITECTURE.md § MCP wiring, § Feedback + datasets, § Evals, § Docker deployment, DESIGN.md, PRODUCTION.md § 1a Docker Compose, § 1b Standalone containers, § 6 Batch operations on 2026-09-28. Updated 2026-09-28 for OpenBBC PR #50. -->

## Open questions

- **Auth middleware bundle.** Ship an optional JWT verifier / mTLS shim so operators aren't
  forced to run a gateway. Blocked on picking a first-supported scheme.
- **Header pass-through on deployed sessions.** Backoffice chat has per-session, per-backend
  `header_overrides`; deployed runtime does not. Extension needs a new column on
  `deployed_sessions` and a `POST /deployed/{agent_id}/sessions/{id}/headers` route.
- **Multi-replica-safe migrations.** Helm chart ships open-bbcd as a `Deployment` and each pod
  runs migrations on boot via embedded `goose` — replicas racing on migrations is still open.
  Fix: switch to goose's `Provider` API with `SessionLocker`, or move migrations to a `Job`
  hook the chart runs pre-install/pre-upgrade.
- **MCP-server generator.** The flow-map-compiler skill only proposes tool names + specs; no
  skill or tool ships that turns `endpoints/*.md` into a runnable MCP server. Today the
  `http_endpoint` `tool_backends` kind is the built-in bridge; a proper generator would live
  outside `open-bbcd`.
- **GHCR public-visibility flip.** The three images (`open-bbcd`, `aikdm-runner`, `aikdm`)
  are published but the packages may still be private under `DACdigital` (see PR #50 body's
  post-merge checklist). Chart pulls anonymous fine only once flipped public.
- **Timeout-based reset of stuck IN_PROGRESS items.** If a batch script dies mid-run, evals /
  training sessions stay IN_PROGRESS forever; today the fix is manual DB update.
- **Agent operator / multi-tenant runtime.** Roadmap mentions an operator pattern for
  multi-agent deployments; unscoped.

<!-- migrated from _migration-quarantine/PRODUCTION.md § 8 Known gaps, ARCHITECTURE.md § Docker deployment future, DESIGN.md § Out of Scope on 2026-09-28. Updated 2026-09-28 for OpenBBC PR #50 (Helm chart + GHCR publish removed from roadmap; MCP generator + multi-replica migrations remain open). -->
