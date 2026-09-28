---
id: tool-runtime
level: 2
parent: open-bbcd
title: tool-runtime
---

# tool-runtime

## Purpose

Shared LLM tool machinery consumed by both orchestrator instances (BO chat inside
[`feedback-datasets`](../feedback-datasets/README.md) + production runtime inside
[`deployed-runtime`](../deployed-runtime/README.md)). Turns a version's structural wiring
(agent-scoped `agent_endpoint_backend` rows + per-version backend visibility on
`agent_version_mcp_backend`) into the live tool set the LLM sees at chat time, and
dispatches each invocation to a concrete backend.

Components:
- **Stateless `tools.Builder`** (`internal/handler/api.go:123`) — assembles the tool set
  per chat/turn from agent+version state.
- **`toolBackendStoreAdapter`** (`internal/handler/api.go:355`) — resolves an endpoint id
  to a `tool_backends` row at dispatch time.
- **MCP-over-REST bridge** — the runtime side of `tool_backends.kind = http_endpoint`:
  `open-bbcd` calls the registered REST endpoint directly and exposes the result to the
  agent as if it were an MCP tool. No client MCP server required.
- **MCP client** — the runtime side of `tool_backends.kind = mcp_client`: proxies to an
  existing MCP server over SSE / Streamable HTTP.

Does not own any Postgres tables — reads structural + per-version state via the
[`agent-lifecycle`](../agent-lifecycle/README.md) repositories. Publishes no external REST
surface — its only contract is the internal one to the two orchestrator instances.

Implements the `mcp-tool-dispatch` and `mcp-over-rest-bridge` L2 capabilities in
`bizbok/capabilities.md`. Header-override merging (from
[`feedback-datasets`](../feedback-datasets/README.md) chat sessions and eval runs) happens
in this layer at dispatch time; the deployed-runtime path passes only static
server-to-server credentials on `tool_backends.config`.

## Scope (in / out)
