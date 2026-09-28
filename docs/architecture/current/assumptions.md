# Assumptions

## Scope assumptions

- Client backend is **already wrapped** by some MCP server (FastAPI MCP, Spring MCP, or a
  bespoke shim). OpenBBC does not build MCP servers for you.
- Client frontend uses the **AG-UI protocol** for the chat surface. The FE never talks to the
  client backend directly — it talks to `open-bbcd`, which mediates via MCP.
- **Single AI agent on day one.** Multi-agent orchestration is out of scope; multiple
  concurrent deployments per agent chain are not supported.
- `aikdm` is out-of-process and DB-unaware; it only talks to `open-bbcd` through the REST API
  via `scripts/run_eval.sh` and `scripts/train_from_session.sh`.

<!-- migrated from _migration-quarantine/DESIGN.md § Assumptions, § Out of Scope, ARCHITECTURE.md § System Overview on 2026-09-28 -->

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
- **Aikdm is out-of-process, DB-unaware.** No shared library between `open-bbcd` and `aikdm`;
  contract is REST + a stable YAML schema (`aikdm/schemas/prompt-v1.yaml`).

<!-- migrated from _migration-quarantine/ARCHITECTURE.md § MCP wiring, § Feedback + datasets, § Evals, § Docker deployment, DESIGN.md on 2026-09-28 -->

## Open questions

- **Auth middleware bundle.** Ship an optional JWT verifier / mTLS shim so operators aren't
  forced to run a gateway. Blocked on picking a first-supported scheme.
- **Header pass-through on deployed sessions.** Backoffice chat has per-session, per-backend
  `header_overrides`; deployed runtime does not. Extension needs a new column on
  `deployed_sessions` and a `POST /deployed/{agent_id}/sessions/{id}/headers` route.
- **Multi-replica-safe migrations.** Switch from goose's package API to `Provider` with
  `SessionLocker` so replicas can boot simultaneously without racing.
- **Registry publish + Helm chart.** No published images yet; deploy by building locally or in
  your CI. No k8s manifests shipped.
- **Timeout-based reset of stuck IN_PROGRESS items.** If a batch script dies mid-run, evals /
  training sessions stay IN_PROGRESS forever; today the fix is manual DB update.
- **Agent operator / multi-tenant runtime.** Roadmap mentions an operator pattern for
  multi-agent deployments; unscoped.

<!-- migrated from _migration-quarantine/PRODUCTION.md § 8 Known gaps, ARCHITECTURE.md § Docker deployment future, DESIGN.md § Out of Scope on 2026-09-28 -->
