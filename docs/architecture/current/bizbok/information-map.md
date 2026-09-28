# Information map

| Concept | Description | Owning capability | Regulatory tag |
|---------|-------------|-------------------|----------------|
| Agent | Named, versioned prompt-bundle-plus-wiring configuration. Structural on `agents`, prompts on `agent_versions`. Linked-list versioning via `agents.parent_version_id`. | `agent-configuration` / `version-management` | ARCH_GAP |
| Agent version | One node in an agent's version linked list. Owns editable prompts + version-scoped MCP attachments. | `version-management` | ARCH_GAP |
| Capability | Backend interface (endpoint / tool) the agent uses. Discovered from the frontend, carried through `FlowMapConfig.Capabilities`, exposed at runtime as tool calls. | `flow-map-compilation` | ARCH_GAP |
| Tool backend | Row in `tool_backends`, kind ∈ `{http_endpoint, mcp_client}`, opaque `config` JSONB. | `mcp-backend-management` | ARCH_GAP |
| MCP attachment | Version-scoped attachment (`agent_version_mcp_backend`) with editable `note`. | `mcp-backend-management` | ARCH_GAP |
| Endpoint→backend wiring | Agent-scoped mapping (`agent_endpoint_backend`) from discovery endpoint id to `tool_backends` row. | `agent-configuration` | ARCH_GAP |
| `.flow-map/` | Directory + zip: flows, capabilities, agents. Uploaded via wizard to seed agent v1. | `flow-map-compilation` | ARCH_GAP |
| Bundle | Aikdm's YAML: `metadata`, `main_prompt`, `capabilities[]`, `skills[]`, `external_actions[]`. Section structure declared in `aikdm/schemas/prompt-v1.yaml`. | `agent-bundle-generation` | ARCH_GAP |
| Chat session | Row in `chat_sessions`: BO test-chat session per (agent version × user). Carries `backend_header_overrides` and `locked_at`. | `backoffice-chat` | ARCH_GAP |
| Chat message | Row in `chat_messages` — user/assistant turns in a chat session. | `backoffice-chat` | ARCH_GAP |
| Feedback | Row in `chat_message_feedback` attached to an assistant message: `rating`, `comment`, `expected_output`, `judge_criteria` (JSONB array). | `judge-criteria-capture` | ARCH_GAP |
| Dataset | Named collection of chat sessions; two-state lifecycle DRAFT → CLOSED. | `dataset-authoring` | ARCH_GAP |
| Dataset version | Row in `dataset_versions`: `status ∈ {DRAFT, CLOSED}`, `version_num`, `close_note`. | `dataset-authoring` | ARCH_GAP |
| Eval | Row in `evals`: one run of an agent version against a closed dataset version. States `PENDING → IN_PROGRESS → DONE|FAILED`. | `eval-run` | ARCH_GAP |
| Eval session | Row in `eval_sessions` (migration 022): per-simulated-session breakdown for an eval, with full `transcript` and per-criterion `judgments` JSONB. | `eval-scoring` | ARCH_GAP |
| Eval header override | Flat `map[string]string` on the eval row (migration 023) proxied to real MCP calls. | `header-override-management-for-evals` | ARCH_GAP |
| Training session | Row in `training_sessions` (migration 024): one automated hill-climb loop from an eval. States `PENDING → IN_PROGRESS → DONE|FAILED`. On DONE, `new_version_id` points at a newly-created agent version. | `training-session-lifecycle` | ARCH_GAP |
| Training report | JSONB blob on the training-session row: per-epoch teacher patches, candidate scores, promote/reject decisions, stopped reason. | `hill-climb-loop` | ARCH_GAP |
| Deployed session | Row in `deployed_sessions`: production runtime session scoped by `user_id`. AG-UI over SSE. | `deployed-session-management` | ARCH_GAP |
| Deployed message | Row in `deployed_messages`: user/assistant turns in a deployed session. | `deployed-session-management` | ARCH_GAP |
| Discovery zip | The `.flow-map/` archive uploaded via the wizard; referenced by every agent version from a persistent `DISCOVERY_STORAGE_DIR` (default `/data/discovery`). | `agent-configuration` | ARCH_GAP |

<!-- migrated from _migration-quarantine/ARCHITECTURE.md § PostgreSQL, § MCP wiring, § Feedback + datasets, § Evals, § Training sessions, DESIGN.md § Versioning, PRODUCTION.md § 1b Standalone containers on 2026-09-28 -->

<!-- ARCH_GAP: no data classification / regulatory tags sourced for any information concept.
     Section: Regulatory tag column
     Fill with: per concept, applicable regime + classification (e.g. GDPR-PII, none, tenant-scoped).
     See: .claude/skills/check-setup/arch-schema.md#bizbok-information-map -->
