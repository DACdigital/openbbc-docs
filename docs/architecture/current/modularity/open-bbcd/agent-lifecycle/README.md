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
deployment. State machine on `agent_versions.status`: `INITIALIZING → PENDING → READY`
(alpha drainer, migration 025); `READY → TRAINING → READY` (hill-climb); `READY →
DEPLOYED` (DB-enforced singleton per agent chain).

Owns tables: `agents` (+ inline `discovery_zip BYTEA` per migration 026), `agent_versions`,
`tool_backends` (kinds `http_endpoint` = MCP-over-REST bridge, `mcp_client` = MCP proxy),
`agent_endpoint_backend` (agent-scoped structural wiring, migration 017),
`agent_version_mcp_backend` (per-version backend visibility + editable prompt notes,
migration 015).

Publishes: `/agents/*`, `/agents/wizard`, `/agent_versions/*/configure/*` (prompts +
architecture + MCP + finalize), `/mcp*` (backend CRUD + test-connection), `/agents/*/deploy`,
`/agents/*/undeploy`.

Implements the [`agent-lifecycle`](../../../ddd/contexts/agent-lifecycle.md) DDD context on
the open-bbcd side.

## Scope (in / out)
