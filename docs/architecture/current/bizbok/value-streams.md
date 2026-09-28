# Value streams

## Streams

### Discovery → alpha agent

Trigger: a discovery author points Claude Code at a target frontend repo.

1. Discovery author runs [`flow-map-compilation`](capabilities.md#l2-capabilities) inside the
   target frontend repo → `.flow-map/` zip.
2. Admin uploads the zip via `/agents/new` → [`agent-configuration`](capabilities.md#l2-capabilities)
   captures flows, endpoints, and endpoint→backend wiring.
3. Admin fleshes out scope, guardrails, personality in the configurator (still
   [`agent-configuration`](capabilities.md#l2-capabilities)).
4. [`agent-bundle-generation`](capabilities.md#l2-capabilities) runs the two-agent
   generator+critic loop → agent v1 lands in Postgres (structural on `agents`, prompts on
   `agent_versions`).

Outcome: agent v1 exists, ready to test.

<!-- migrated from _migration-quarantine/DESIGN.md § Phase 0, § Phase I, ARCHITECTURE.md § Data Flow / Flow 1 on 2026-09-28 -->

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
