# Integrations

| System | Kind | Protocol | Auth | SLA/regulatory |
|--------|------|----------|------|----------------|
| Client frontend | Inbound consumer | AG-UI over Server-Sent Events (SSE), HTTPS | `user_id` in request body/query, trusted from the operator's gateway | ARCH_GAP — no SLA sourced |
| Client backend (MCP-wrapped) | Outbound tool provider | MCP over SSE or Streamable HTTP | Server-to-server credentials in the `tool_backends.config` JSONB; per-session HTTP header overrides merged in the BO chat + eval paths (`chat_sessions.backend_header_overrides` migration 016; `evals.header_overrides` migration 023). No per-session overrides on the deployed path today. | ARCH_GAP — no SLA / regulatory constraints sourced |
| LLM providers — Anthropic (default), OpenAI, Gemini | Outbound completion | HTTPS via Google ADK + LiteLLM (aikdm); direct Anthropic API (open-bbcd) | `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, `GEMINI_API_KEY` from env (in production: platform secret store) | Provider-defined SLAs; ARCH_GAP for internal-policy targets |
| Operator's auth gateway | Inbound trust mediator | HTTPS (whatever the gateway speaks upstream: session cookie, bearer, mTLS, SSO) | Gateway-owned; injects a verified `user_id` before forwarding to `open-bbcd` | Operator-owned; ARCH_GAP for internal policy |
| Claude Code (`flow-map-compiler`) | Discovery-side skill host | Local IPC (Claude Code plugin API) | Runs client-side on the discovery author's machine; the resulting zip is uploaded via wizard authenticated by the same operator gateway that fronts the backoffice | ARCH_GAP |

<!-- migrated from _migration-quarantine/PRODUCTION.md § 2 Integrating your frontend, § 3 MCP layer, § 4 Headers, § 5 Auth model, § 7 Provider LLM keys, ARCHITECTURE.md § Protocols, § flow-map-compiler on 2026-09-28 -->

## Contracts

- **AG-UI event stream (client frontend ↔ open-bbcd).** Event types: `RUN_STARTED`,
  `TEXT_MESSAGE_START/CONTENT/END`, `TOOL_CALL_START/ARGS/END`, `TURN_END`, `ERROR`.
  Upstream spec: [ag-ui-protocol/ag-ui](https://github.com/ag-ui-protocol/ag-ui).
- **MCP (open-bbcd ↔ client backend).** SSE + Streamable HTTP transports. Two `tool_backends`
  kinds: `http_endpoint` (`open-bbcd` acts as an MCP client to an HTTP endpoint wrapped by an
  MCP shim) and `mcp_client` (native MCP client to an existing MCP server).
- **`.flow-map/` (discovery → wizard).** Schema in
  `bbc-discovery/flow-map-compiler/references/output-schemas.md` (versioned; contract-triple
  linted by `references/lint-contract.md`).
- **aikdm bundle format.** `aikdm/schemas/prompt-v1.yaml` — sections `metadata`,
  `main_prompt`, `capabilities[]`, `skills[]`, `external_actions[]`. Versioned; schema
  changes bump the file.
- **Aikdm eval input.** `eval-input.yaml` — agent version + dataset version pair, consumed by
  `aikdm evaluate` and `aikdm train-agent`. Structural shape defined by aikdm; served by
  `open-bbcd` at `GET /evals/{id}/export.yaml`.

<!-- migrated from _migration-quarantine/ARCHITECTURE.md § flow-map-compiler, § aikdm, § MCP wiring, § Protocols, PRODUCTION.md § 2, § 3 MCP layer on 2026-09-28 -->
