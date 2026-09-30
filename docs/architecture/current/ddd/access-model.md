# Access model

## Model

**Gateway-delegated.** `open-bbcd` ships **auth-agnostic**: no route enforces authentication
or authorization. Every operator must front the service with an auth gateway (Envoy / nginx /
API gateway / SSO / mTLS / …) that:

1. terminates customer auth at ingress,
2. rewrites the request to inject a **verified** `user_id`, and
3. forwards to `open-bbcd`.

Inside `open-bbcd`, the trust model is: `user_id` on the request body/query is authoritative.
Cross-user requests return `404 Not Found` (not `403`) to prevent session-id enumeration.

The backoffice surface is not authenticated at the route level either — operators are
expected to restrict network reachability (private VPC, VPN, or employee SSO in front) rather
than allowlist individual routes, because the surface is large enough that per-route
allowlisting is brittle.

<!-- migrated from _migration-quarantine/PRODUCTION.md § 5 Auth model on 2026-09-28 -->

## Policies

| Actor | Capability | Info concept | Ops | Conditions |
|-------|------------|--------------|-----|-----------|
| End user | `deployed-session-management`, `ag-ui-turn-streaming` | Deployed session (own), Deployed message (own) | create, read, update-title, delete, turn | Gateway must have verified the caller and rewritten `user_id` to the verified identity. `open-bbcd` returns 404 on `user_id` mismatch. |
| End user | `mcp-tool-dispatch` | Client backend capability (indirect) | invoke (via deployed session) | Tool call dispatched by `open-bbcd` over MCP with server-to-server credentials on the registered `tool_backends` row; no per-session header pass-through today. |
| End user | `deployed-runtime-artifacts` | Artifact (own session's) | upload, read | Same trust boundary as deployed messages. `POST /deployed/{agent_id}/sessions/{id}/artifacts?user_id=X` and `GET /artifacts/{store_id}/{uri}?session_id=…&user_id=…` both 404 on session/user mismatch. `ARTIFACT_MAX_UPLOAD_MB` env cap applies. |
| Admin / domain expert | `agent-configuration`, `mcp-backend-management`, `version-management`, `agent-deployment`, `agent-tool-configuration` | Agent, Agent version, Tool backend, Endpoint→backend wiring, MCP attachment, Sub-agent binding | full CRUD | Full backoffice surface behind the operator's network restriction (VPC/VPN/SSO). No per-user scoping at `open-bbcd`. Sub-agent bindings are admin-only because a sub-agent runs with its own tool wiring — binding a worker grants the caller indirect reach to that worker's backends. |
| End user | `sub-agent-dispatch` | Child session (own root's), Sub-agent binding (indirect) | invoke (via the root agent's turn), read child messages | End users never pick sub-agents; the root LLM does, from the version's allow-list. Child sessions inherit the root's `user_id`; reads 404 on mismatch. Not listed by `GET /deployed/{agent_id}/sessions`. |
| Operator (deploy-time only) | `artifact-store-adapter` (config surface) | Artifact store (registry entries) | declare via env at deploy time | Stores are configured through `ARTIFACT_STORE_<ID>_*` env vars — no runtime BO / REST surface. Credentials handled at the deploy env / secret store, same class as LLM API keys. |
| Admin / domain expert | `backoffice-chat`, `judge-criteria-capture`, `dataset-authoring`, `chat-artifacts` | Chat session, Chat message, Feedback, Dataset, Dataset version, Artifact (own BO session's) | create/read/close-draft, upload/read | Same trust boundary as configurator. Artifact upload capped by `ARTIFACT_MAX_UPLOAD_MB`. |
| Admin / domain expert | `eval-run`, `header-override-management-for-evals`, `training-session-lifecycle` | Eval, Eval session, Training session, Training report | create (via BO buttons), read | Same trust boundary. `mock_mcp_tools` toggle and `header_overrides` set at eval creation. |
| Operator | `eval-run`, `training-session-lifecycle` | Eval, Training session | start / result / complete / fail (state transitions), list PENDING | Cron scripts (`process_pending_evals.sh`, `process_pending_trainings.sh`) hit REST unauthenticated; same trust boundary as backoffice. |
| Discovery author | `flow-map-compilation` | `.flow-map/` bundle | produce | Runs client-side inside a Claude Code session against a target repo; no direct contact with `open-bbcd` REST. |
| Client backend owner | `mcp-tool-dispatch` (server side) | Client backend capability | expose | Registers MCP server(s) that `tool_backends` rows point at; server-to-server credentials only. |
| Operator's auth gateway | `deployed-session-management` (mediator) | Deployed session (via `user_id`) | verify + rewrite | External to OpenBBC. Injects a verified `user_id` before forwarding. Required for any non-trusted-network deployment. |

<!-- migrated from _migration-quarantine/PRODUCTION.md § 4 Headers, § 5 Auth model, § 6 Batch operations, ARCHITECTURE.md § Session Proxying on 2026-09-28. Updated 2026-09-30 for multiagent-tools — agent-tool-configuration on Admin row; End-user sub-agent-dispatch row. -->

## Tenant scoping

- **`user_id` string scoping only** — no first-class tenant concept inside `open-bbcd`.
  `deployed_sessions` are scoped by `(agent_id, user_id)` where `user_id` is an opaque string
  the gateway supplies.
- **No multi-tenant hard isolation inside a single instance.** All data lives in one
  database; deployed sessions are user-scoped by string comparison only. Operators needing
  regulatory tenant isolation are expected to **run one `open-bbcd` per tenant**.
- **Child sessions inherit the root session's `user_id`** — sub-agent runs never widen or
  change the tenant scope of a turn.
- **Header pass-through is a partial tenant-scoping tool** in the backoffice chat and eval
  paths (per-session `header_overrides` proxied to MCP backends), but is **not supported on
  the deployed runtime** today (see [`../assumptions.md § Open questions`](../assumptions.md#open-questions)).

<!-- migrated from _migration-quarantine/PRODUCTION.md § 5 Auth model, § 4 Headers, § 8 Known gaps on 2026-09-28 -->
