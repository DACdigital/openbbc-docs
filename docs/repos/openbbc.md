# openbbc — index

**Type:** backend monorepo (Go service + Python CLI + Claude Code plugin marketplace)
**Language / stack:** Go 1.22+ (open-bbcd) · Python 3.12+ with uv (aikdm) · Claude Code plugin skill (bbc-discovery/flow-map-compiler) · PostgreSQL 15+ · htmx (backoffice UI) · AG-UI over SSE (deployed runtime) · MCP (SSE, Streamable HTTP) for tool calls · Google ADK + LiteLLM (multi-provider: Anthropic, OpenAI, Gemini) · Helm chart (k8s deployment)
**Git host:** github/DACdigital/OpenBBC
**Last indexed:** 2026-09-28

## Layout

- `open-bbcd/` — Go daemon: backoffice UI + REST API + deployed agent runtime + MCP-over-REST bridge (single binary). Owns everything in Postgres transactionally.
- `aikdm/` — Python CLI: `generate-agent`, `evaluate`, `train-agent`. Out-of-process, DB-unaware, REST-only against `open-bbcd`.
- `bbc-discovery/` — Claude Code plugin marketplace. Currently ships one plugin (`flow-map-compiler`) — a discovery skill that compiles a frontend repo into a `.flow-map/` wiki (schema v2). Pure markdown + plugin manifests, no build step.
- `deploy/helm/openbbc/` — Helm chart shipping `open-bbcd` Deployment + Service + optional Ingress, optional in-cluster Postgres StatefulSet, and three CronJobs (alphas, evals, trainings) running the `aikdm-runner` image. Default `image.repository` values point at `ghcr.io/dacdigital/openbbc/{open-bbcd,aikdm-runner}`.
- `scripts/` — one-shot + drainer scripts consumed by the CronJobs and by operators: `process_pending_alphas.sh`, `process_pending_evals.sh`, `process_pending_trainings.sh`, `generate_alpha.sh`, `run_eval.sh`, `train_from_session.sh`, `seed_bundle.py`. All `flock`-protected, serial, continue-on-error.
- `docs/` — `ARCHITECTURE.md`, `DESIGN.md`, `PRODUCTION.md`. The mirror source for `../docs/architecture/current/_migration-quarantine/` in the docs repo.
- `docker-compose.yml` — local-dev-only stack (Postgres + `open-bbcd`; `aikdm` behind an `aikdm` compose profile). Not a shipping deployment path.
- `Dockerfile.aikdm-runner` — image used by the Helm chart's CronJobs; packages bash + curl + tini + python 3.12 + uv + `aikdm/` source + `scripts/`.
- `.github/workflows/` — `ci.yml` (Go + Python tests on PRs) and `publish-images.yml` (multi-arch build+push of `open-bbcd`, `aikdm-runner`, `aikdm` to GHCR on every PR / merge to `main` / `v*` tag).
- `CLAUDE.md` — guidance for Claude Code when working inside this repo.

## Entry points

- `open-bbcd/cmd/open-bbcd/main.go` — Go binary. Dispatches on first arg: `serve` (default) / `migrate` / `healthcheck`.
- `open-bbcd/internal/handler/api.go:194` — HTTP mux entrypoint; server-rendered `html/template` + htmx surface, no SPA.
- `open-bbcd/internal/handler/api.go:123` — `tools.Builder` shared by BO chat + deployed orchestrators.
- `open-bbcd/internal/llm/tools/builder.go:92` — runtime dispatch for `http_endpoint` tool-backends (the MCP-over-REST bridge).
- `open-bbcd/migrations/` — embedded via `//go:embed`; currently at `026_agent_discovery_zip`. Applied automatically on `serve` boot.
- `aikdm/aikdm/cli.py` — click CLI. Three subcommands: `generate-agent`, `evaluate`, `train-agent`.
- `aikdm/aikdm/train/orchestrator.py` — `run_training`: teacher/judge/eval hill-climb loop.
- `aikdm/schemas/prompt-v1.yaml` — bundle format schema (versioned).
- `bbc-discovery/flow-map-compiler/skills/flow-map-compiler/SKILL.md` — the discovery skill itself. Fully agent-driven; no scripted pipeline.
- `bbc-discovery/flow-map-compiler/references/output-schemas.md` — `.flow-map/` schema v2 contract.
- `scripts/seed_bundle.py` — writes generated bundles to Postgres and flips `agent_versions.status` from `PENDING` to `READY`.

## Commands

**open-bbcd** (from `open-bbcd/`):

- Build: `make build` (outputs `bin/open-bbcd`).
- Test: `make test` (Go `-race`, unit).
- Test (CI parity): `make test-integration` (`-p 1`, requires local Postgres).
- Migrations: `make migrate-up`, `make migrate-down`, `make migrate-status`.
- Run local: `DATABASE_URL=postgres://... make run` (or `./bin/open-bbcd`).

**aikdm** (from `aikdm/`):

- Install deps: `uv sync --all-extras`.
- Test (mocked LLM): `make test`.
- Test (real LLM smoke): `RUN_SMOKE=1 make test-smoke`.
- Lint / format: `make lint`, `make fmt`.
- Run: `uv run aikdm generate-agent --config … --output …`, `uv run aikdm evaluate --input …`, `uv run aikdm train-agent --input … --epochs N --patience K --out …`.

**Full stack (compose, local dev only)**:

- `docker compose up -d` — Postgres + `open-bbcd` on `:8080`; migrations auto-apply on boot.
- `docker compose --profile aikdm run --rm aikdm …` — one-shot `aikdm` invocation.

**k8s (Helm chart)**:

- `helm upgrade --install openbbc deploy/helm/openbbc --namespace openbbc --create-namespace --set openbbcd.image.tag=$TAG --set aikdmRunner.image.tag=$TAG --set aikdmRunner.secrets.ANTHROPIC_API_KEY=… …`
- Tags shipped by `publish-images.yml`: `pr-<num>`, `main`, `sha-<short>`, semver.

**Drainer scripts** (one-shot; also run inside the Helm CronJobs):

- `OPENBBCD_URL=http://localhost:8080 scripts/process_pending_alphas.sh` — cron `*/5`; needs `DATABASE_URL` and LLM keys.
- `OPENBBCD_URL=http://localhost:8080 scripts/process_pending_evals.sh` — cron `*/10`.
- `OPENBBCD_URL=http://localhost:8080 scripts/process_pending_trainings.sh` — cron `*/15`.

## Conventions detected

- **Build tools:** Go `make` targets + go modules; `uv` for Python deps; `goose` for SQL migrations embedded via `//go:embed`.
- **Testing:** Go `-race` unit tests + `-p 1` integration tests against a real Postgres in CI (`services.postgres` in `ci.yml`); Python tests via `make test` with LLM mocked, gated real-LLM smoke test via `RUN_SMOKE=1`; e2e via Playwright noted in PR history.
- **CI:** GitHub Actions — `test-go` + `test-python` + `check` on PRs (`.github/workflows/ci.yml`); `publish-images` on PR/merge/tag (`.github/workflows/publish-images.yml`).
- **Runtime images:** distroless (`gcr.io/distroless/static-debian12:nonroot` for `open-bbcd`, CGO off), python-slim for `aikdm` and `aikdm-runner`. All multi-arch (`linux/amd64`, `linux/arm64`) via BuildKit.
- **Persistence:** all state in PostgreSQL 15+. `open-bbcd` keeps no local disk state after migration 026 inlined the discovery zip on `agents.discovery_zip BYTEA`.
- **`agent_versions.status` state machine** (migration 025): `INITIALIZING → PENDING → READY → TRAINING → READY → DEPLOYED`.
- **Tool-backend model:** two `tool_backends.kind` values — `http_endpoint` (built-in MCP-over-REST bridge, no client-side MCP server required) or `mcp_client` (proxy to an existing MCP server).
- **Auth-agnostic ship:** no route enforces auth; operators front `open-bbcd` with a gateway that verifies the caller and rewrites `user_id`.
- **Discovery contract triple:** `bbc-discovery/flow-map-compiler/references/output-schemas.md` ↔ `references/lint-contract.md` ↔ `assets/templates/*.tmpl` must stay in sync.
- **Anti-goals (LOCKED) for the discovery skill:** never generate MCP server code, never assume an MCP server exists, never run target-repo code.
