---
id: agent-lifecycle
level: 2
parent: open-bbcd
title: agent-lifecycle
---

# agent-lifecycle

## Purpose

Agents + versions + capabilities + endpoint→backend wiring + per-version MCP visibility
(which backends each version's LLM can see, plus editable per-version prompt notes) +
deployment + declarative seed import/export. State machine on `agent_versions.status`:
`INITIALIZING → PENDING → READY` (alpha drainer, migration 025); `READY → TRAINING → READY`
(hill-climb); `READY → DEPLOYED` (DB-enforced singleton per agent chain); seed import
inserts versions directly as `READY`. Versions are immutable once they leave
`INITIALIZING`/`DRAFT` (guarded in the repository layer), and every fork appends at the
agent's tip under the agent-row lock. Agents and versions carry write-once static IDs as
their cross-cluster identity.

Owns tables: `agents` (+ inline `discovery_zip BYTEA` per migration 026), `agent_versions`,
`tool_backends` (kinds `http_endpoint` = MCP-over-REST bridge, `mcp_client` = MCP proxy;
`managed_by_seed` locks a backend owned by the seed set),
`agent_endpoint_backend` (agent-scoped structural wiring, migration 017),
`agent_version_mcp_backend` (per-version backend visibility + editable prompt notes,
migration 015), `agent_version_subagent` (sub-agent bindings on the caller version, +
`agent_versions.agent_tool_enabled`).

Publishes: `/agents/*`, `/agents/wizard`, `/agent_versions/*/configure/*` (prompts +
architecture + MCP + Agents tab + finalize), `/agent_versions/*/architecture/{mcp,agents}/*`
(agent-tool writes return `409` outside `INITIALIZING`/`DRAFT`), and version/agent delete
(`409` when a binding target, deployed child version or locked-session version would be
removed), `/mcp*` (backend CRUD + test-connection; `409` on managed backends),
`/agents/*/deploy`, `/agents/*/undeploy` (both accept static IDs),
`POST /agent_versions/*/export` (`openbbc-seed-v1` YAML), and the boot-time seed import
(`SEED_PATH`) plus the `open-bbcd seed validate|apply` CLI. MCP visibility and note writes
return `409` outside `INITIALIZING`/`DRAFT`.

Implements the [`agent-lifecycle`](../../../ddd/contexts/agent-lifecycle.md) DDD context on
the open-bbcd side.

## Scope (in / out)

**In:** agent, version, backend and wiring state and its invariants; the version state
machine and immutability guard; append-at-tip forking; deploy/undeploy; static IDs and
their resolution on lifecycle routes; seed file validation, boot import, `seed` CLI and
version export.

**Out:** running turns (BO chat lives in `feedback-datasets`, production turns in
[`deployed-runtime`](../deployed-runtime/README.md)); bundle generation (aikdm, via the
alpha drainer); the Helm seed ConfigMap and probes (deployment, not code); deleting or
updating existing agents/versions from a seed; import through REST or the backoffice.
