# declarative-seed — declarative seed import/export with static IDs

**Date**: 2026-11-24
**Codename**: declarative-seed

**Driver**: new: deploy locally-trained agent versions onto clusters we don't operate, declaratively, with no backoffice step

**Decision**: make agents, versions and MCP backends declarable in a versioned `openbbc-seed-v1`
YAML seed set that any instance can export and that `open-bbcd` imports at boot (after
migrations, before serving, in one transaction — backends "file wins", agents and versions
"create if missing, otherwise leave alone", optional `if_none`/`force` deploy), with
write-once static IDs as the cross-cluster identity and every version made immutable once it
leaves `INITIALIZING`/`DRAFT`.

**Rationale**: agents, versions and their wiring exist only in Postgres and are created only
through the backoffice, and every ID is a per-database UUID, so a version trained locally has
no identity that means anything on another cluster. A boot-time seed gives a fresh cluster a
working agent with no backoffice step, while never overwriting the client's own versions or
deployment: our version is guaranteed present, so the client can always deploy or roll back to
it. Static IDs only stay meaningful if the content behind them cannot change, which is why
version immutability is enforced for every writer and forks append at the tip.

**Alternatives rejected**:
- Helm post-install Job calling the REST API: the app is already serving before the import runs, so a fresh cluster briefly (or, on failure, indefinitely) serves without its agent.
- Init container running the import: migrations run in the main container, so the schema is absent when an init container would run.
- Kubernetes operator / CRDs: overkill for a handful of agents per cluster; adds a controller to build, ship and secure.
- Overwrite-to-match-file for agents and versions: silently wipes a client's edits and deployments on every restart.
- Deterministic UUIDs instead of static IDs: no human-readable reference, and a version cannot be named before it is exported.

**Impact**:
- `docs/architecture/current/bizbok/capabilities.md` — new L2 `declarative-seeding` under Agent lifecycle management (+ L1 text and map row); immutability + append-at-tip in `version-management`; managed backends and MCP write lock in `mcp-backend-management`; static IDs in `agent-deployment`; seed-import binding exception in `agent-tool-configuration`.
- `docs/architecture/current/bizbok/value-streams.md` — new stream "Ship a trained version to a client cluster".
- `docs/architecture/current/bizbok/information-map.md` — new concepts Seed set, Static ID, Managed backend; Agent version immutability and lineage.
- `docs/architecture/current/glossary.md` — new terms Seed set, Static ID, Tip, Managed backend (disambiguated from `seed_bundle.py` and dataset seeding); Agent version now immutable once it leaves `INITIALIZING`/`DRAFT`; Agent topology seed-import exception; Alpha drainer `seed_bundle.py` limits.
- `docs/architecture/current/ddd/contexts/agent-lifecycle.md` — static IDs, tip lineage, managed backends, Seed set value object; invariants for repository-level immutability, append-at-tip forks serialized on the agent row, `READY` entry via seed import, seed-import binding exception, seed import/export rules, finalized architecture never rewritten; published surface adds export, static IDs on deploy/undeploy, `409`s, seed CLI and file contract.
- `docs/architecture/current/ddd/contexts/deployed-runtime.md` — `{agent_id}` accepts the agent static ID; sessions stay keyed by UUID.
- `docs/architecture/current/ddd/access-model.md` — Operator (deploy-time) seeding policy; Admin export policy; MCP writes `409` on locked versions; managed backends read-only.
- `docs/architecture/current/c4/containers.md` — open-bbcd boot sequence with seed import, `seed validate|apply` subcommands, export route, static IDs in routes, new columns.
- `docs/architecture/current/c4/data-flows.md` — new flows "Seed export" and "Boot seed import".
- `docs/architecture/current/c4/deployment.md` — Helm `seed:` block (ConfigMap at `/etc/openbbc/seed`, `SEED_PATH`, `${VAR}`s from the env Secret, checksum roll, startupProbe); seed ConfigMap in diagram; deploy-time config trust boundary; four threats (boot DoS, backend takeover by name, export leaking URL-embedded tokens, `seed apply` as a Data-zone path).
- `docs/architecture/current/constraints.md` — seed-import exception to the acyclic-by-construction DAG rule; version immutability + append-at-tip; static ID format; all-or-nothing boot import.
- `docs/architecture/current/assumptions.md` — "architecture frozen after finalize" is load-bearing for the import identity check; sub-agent targets need not be deployed; new locked decision for declarative seeding.
- `docs/architecture/current/modularity/open-bbcd/agent-lifecycle/README.md` — Purpose and Scope: seed import/export, static IDs, immutability, managed backends.
- `docs/architecture/current/modularity/open-bbcd/deployed-runtime/README.md` — Purpose and Scope: static agent ID in routes.
- downstream: OpenBBC (open-bbcd, deploy/helm/openbbc, scripts/seed_bundle.py)

**Links**:
- Spec: docs/superpowers/specs/2026-11-24-declarative-seed-design.md (spec PR to follow)
