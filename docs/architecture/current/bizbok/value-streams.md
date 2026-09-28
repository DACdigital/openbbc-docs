# Value streams

## Streams

### Discovery → alpha agent

Trigger: a discovery author points Claude Code at a target frontend repo.

1. Discovery author runs [`flow-map-compilation`](capabilities.md#l2-capabilities) inside the
   target frontend repo → `.flow-map/` (schema v2) + zip. The output is a complete
   business-and-technical understanding of the app: user journeys (`flows/`), business-domain
   specialties (`skills/`), app-wide invariants + conventions + boundaries (`APP.md`), a
   domain glossary, **and** the proposed backend surface (`endpoints/`). It feeds both
   prompt generation (skills + flows + glossary become the runtime agent's context) and
   tool wiring (endpoints become `tool_backends` bindings). LOCKED anti-goal: no MCP server
   code is generated.
2. Admin uploads the zip via `/agents/new` → [`agent-configuration`](capabilities.md#l2-capabilities)
   captures flows, endpoints, and endpoint→backend wiring. The zip is stored inline on
   `agents.discovery_zip BYTEA` (migration 026); no persistent volume is used.
3. For each discovered endpoint, admin binds a `tool_backends` row via
   [`mcp-backend-management`](capabilities.md#l2-capabilities) — either an `http_endpoint`
   (OpenBBC's [`mcp-over-rest-bridge`](capabilities.md#l2-capabilities) wraps a plain REST
   endpoint as MCP for the agent) or an `mcp_client` (proxy to an existing MCP server).
4. Admin fleshes out scope, guardrails, personality in the configurator, then hits Finalize
   → root version transitions `INITIALIZING → PENDING` (migration 025).
5. Operator (or a k8s CronJob) runs
   [`alpha-drainer`](capabilities.md#l2-capabilities):
   `process_pending_alphas.sh` → `generate_alpha.sh` runs
   [`agent-bundle-generation`](capabilities.md#l2-capabilities) (aikdm generate-agent's
   two-agent generator+critic loop) → `seed_bundle.py` writes the bundle to Postgres and
   transitions the version `PENDING → READY`.

Outcome: agent v1 in `READY` status, ready to test in the backoffice chat.

<!-- migrated from _migration-quarantine/DESIGN.md § Phase 0, § Phase I, ARCHITECTURE.md § Data Flow / Flow 1 on 2026-09-28. Updated 2026-09-28 for OpenBBC PR #50 — Finalize now asynchronous via alpha drainer + mig 025 PENDING state + mig 026 inline zip; clarified two tool_backends kinds. -->

### Feedback → dataset

Trigger: admin picks an agent version to iterate on.

1. Admin runs [`backoffice-chat`](capabilities.md#l2-capabilities) at
   `/agent_versions/{id}/chat`, interacting with the agent version.
2. Per assistant message the admin captures feedback via
   [`judge-criteria-capture`](capabilities.md#l2-capabilities): rating, comment, expected
   output, and a JSONB array of acceptance criteria.
3. Admin assigns the chat session to a DRAFT dataset via
   [`dataset-authoring`](capabilities.md#l2-capabilities); at most one DRAFT per dataset.
4. Admin closes the DRAFT via `POST /datasets/{id}/close-draft/confirm`
   ([`dataset-authoring`](capabilities.md#l2-capabilities)); member sessions flip to
   `chat_sessions.locked_at`. The next DRAFT is seeded with the CLOSED version's sessions
   (cumulative).

Outcome: a CLOSED dataset version exists, ready to evaluate against.

<!-- migrated from _migration-quarantine/DESIGN.md § Phase II, ARCHITECTURE.md § Feedback + datasets, § Data Flow / Flow 2 on 2026-09-28 -->

### Evaluation

Trigger: admin picks an agent version to score against a CLOSED dataset version.

1. Backoffice: admin clicks Evaluate on the version detail's Versions tab
   ([`eval-run`](capabilities.md#l2-capabilities)) — server creates an `evals` row in PENDING
   with `mock_mcp_tools` and optional header overrides.
2. Operator runs `scripts/run_eval.sh <eval_id>` — script exports input, transitions to
   IN_PROGRESS, invokes aikdm ([`eval-scoring`](capabilities.md#l2-capabilities)).
3. Aikdm per session: simulator → target → tool_mock/real → judge; each session becomes an
   `eval_sessions` row with `transcript` + per-criterion `judgments`.
4. Script posts result → server flips eval to DONE (or FAILED) and computes global pass-rate
   `sum(passed_criteria) / sum(total_criteria)`.

Outcome: DONE eval with a score in `[0, 1]`. If `score < 1.0`, the detail page reveals the
Train button (see next stream).

<!-- migrated from _migration-quarantine/DESIGN.md § Phase III, ARCHITECTURE.md § Evals, § Data Flow / Flow 3 on 2026-09-28 -->

### Automated training

Trigger: an eval detail with `status = DONE`, `score < 1.0`, no active training session for
the eval.

1. Admin clicks Train on the eval detail
   ([`training-session-lifecycle`](capabilities.md#l2-capabilities)); server inserts a
   `training_sessions` row in PENDING.
2. Operator runs `scripts/train_from_session.sh <session_id>` — script transitions to
   IN_PROGRESS and invokes `aikdm train-agent`.
3. Aikdm runs the [`hill-climb-loop`](capabilities.md#l2-capabilities): baseline eval →
   epoch (teacher patches → apply → eval as reward → promote if strictly greater than best) →
   early stop on `patience` non-improvements or perfect score.
4. Script prints score diff, prompts operator y/N, posts `/complete` → server creates a new
   agent version, flips session to DONE (or `/fail` → FAILED with reason).

Outcome: DONE training session pointing at a newly-created agent version via `new_version_id`
(or FAILED with a `stopped_reason`).

<!-- migrated from _migration-quarantine/DESIGN.md § Phase IV, ARCHITECTURE.md § Training sessions, § Data Flow / Flow 4 on 2026-09-28 -->

### Multimodal chat with artifacts

Trigger: user or admin needs to exchange a file with the agent — inbound (context material,
image the agent reasons about, document to summarise) or outbound (chart, report, generated
image, tool-produced artefact).

1. Operator has configured at least one artifact store via
   [`artifact-store-management`](capabilities.md#l2-capabilities) and marked it `is_default`.
2. User submits a turn carrying an inline file (multipart on the BO chat path via
   [`chat-artifacts`](capabilities.md#l2-capabilities); AG-UI-side upload endpoint on the
   deployed path via [`deployed-runtime-artifacts`](capabilities.md#l2-capabilities)) —
   `open-bbcd` streams the bytes to the store through the
   [`artifact-store-adapter`](capabilities.md#l2-capabilities) and records an `artifact_ref`
   content block on the user-role message.
3. The runtime tool builder ([`mcp-tool-dispatch`](capabilities.md#l2-capabilities))
   dispatches the turn to the agent. When the agent calls an MCP tool with an artifact
   argument, the framework materialises the ref for the tool according to the backend
   contract (inline base64, presigned URL, or MCP `resource` reference).
4. If the tool result carries an `ImageContent` or `EmbeddedResource` (native MCP file
   payload), [`chat-artifacts`](capabilities.md#l2-capabilities) /
   [`deployed-runtime-artifacts`](capabilities.md#l2-capabilities) unpack it — bytes go to
   the artifact store; an `artifact_ref` content block lands on the `tool`-role message.
5. Assistant response may emit its own `artifact_ref` content block; on BO chat it renders
   inline in the transcript; on the deployed path it streams through the AG-UI wire as an
   `artifact_ref` event (AG-UI event-type extension — see
   [`../ddd/contexts/deployed-runtime.md`](../ddd/contexts/deployed-runtime.md)).
6. On BO chat, dataset close-draft ([`dataset-authoring`](capabilities.md#l2-capabilities))
   captures the artifact refs verbatim on the frozen session — eval replay resolves them
   through the same adapter as at chat time.

Outcome: the user/admin exchanges any-MIME files with the agent in either direction; bytes
remain in the deployer's chosen artifact store; the session transcript stays deterministic
and replayable for evals.

<!-- new value stream added 2026-09-28 for artifact-support. -->

### Deployment

Trigger: admin has a tested agent version they want to expose to end users.

1. Admin marks the version DEPLOYED
   ([`agent-deployment`](capabilities.md#l2-capabilities)) via `POST /agents/{agent_id}/deploy`;
   the DB-enforced singleton implicitly rotates the previously-deployed version in the same
   chain.
2. End user's frontend calls `POST /deployed/{agent_id}/sessions`
   ([`deployed-session-management`](capabilities.md#l2-capabilities)) to start a session,
   passing `user_id` (verified upstream by the operator's gateway).
3. Frontend streams turns via `POST /deployed/{agent_id}/sessions/{session_id}/turn`
   ([`ag-ui-turn-streaming`](capabilities.md#l2-capabilities)) → AG-UI SSE event stream.
4. At runtime, [`mcp-tool-dispatch`](capabilities.md#l2-capabilities) resolves each
   agent-initiated tool call to a `tool_backends` row and proxies over MCP (SSE / Streamable
   HTTP) to the client backend.

Outcome: the end user gets streamed agent responses; the client backend gets MCP-mediated
tool calls scoped to the session's `user_id`.

<!-- migrated from _migration-quarantine/DESIGN.md § Phase V, ARCHITECTURE.md § Data Flow / Flow 5, PRODUCTION.md § 2 Integrating your frontend on 2026-09-28 -->
