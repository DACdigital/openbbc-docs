# multiagent-tools — Claude-style agent tool for multi-agent topologies over pinned versions

**Date**: 2026-09-30
**Codename**: multiagent-tools

**Driver**: new feature: multiagent-tools. Multi-agent support has been on the roadmap since
the start. An existing downstream project now needs it: one entrypoint agent that
delegates to specialised agents (planner → coordinator → workers). The locked scope
assumption "single AI agent on day one; multi-agent orchestration out of scope" is being
lifted.

**Decision**: add a built-in **agent tool**, modelled on Claude Code's sub-agent / Task
tool, that an agent version can enable via the `agent_versions.agent_tool_enabled`
checkbox. Its allow-list is a set of version-scoped **sub-agent bindings** on the caller
(`agent_version_subagent`: pinned `target_version_id`, tool-facing `name`, prompt `note`).
The LLM sees one tool, `agent(subagent: enum, description, prompt, artifacts?)`. A call
runs the pinned target version's full turn loop **in-process** inside `open-bbcd`, using
its own prompts and wiring. The sub-agent starts with a fresh context holding only the
prompt and artifacts, and returns its final text plus `artifact_ref`s. Each spawn is
persisted as a **child session** (same `chat_sessions` / `deployed_sessions` tables,
linked by `parent_session_id` + `parent_tool_call_id`, with `depth`). Sub-agents inherit
`user_id`, the artifact scope and (on BO/eval paths) `header_overrides`.

Guardrails:
- the topology must be a DAG; cycles are refused at bind time
- `AGENT_TOOL_MAX_DEPTH` (default 3)
- `AGENT_TOOL_MAX_PARALLEL` (default 4)
- targets must be `READY` or `DEPLOYED`

The AG-UI stream shows sub-agent progress as standard `STEP_STARTED` / `STEP_FINISHED`
events plus child-tagged `TOOL_CALL_*`; sub-agent text tokens are not streamed. Evals run
the real topology: `eval-input.yaml` carries pinned sub-agent bundles transitively and
aikdm runs them in-process. `mock_mcp_tools` applies only to leaf MCP tools. Training
copies the agent-tool config verbatim and hill-climbs the root version's prompts only.
No new service, bounded context or modularity node.

**Rationale**:
- **Claude Code as the model.** Its sub-agent pattern is proven in practice and needs no
  new protocol. An in-process call reuses the session, message, artifact, tool-dispatch
  and AG-UI machinery that already exists.
- **Config on the caller's version.** The caller's prompt decides when and how to
  delegate, so the config varies per version the same way MCP attachments and their notes
  already do.
- **Pinned target versions.** They make BO chat, evals and training reproducible. They
  also let `aikdm` (DB-unaware) receive the whole topology in one export, and they make
  cycle detection a static graph check. Workers never need to be user-facing or deployed.
- **Linked child sessions.** Worker transcripts stay inspectable and queryable without
  bloating parent message JSONB. Each child uses the same message and `artifact_ref`
  shape, and locks with its root so replays stay deterministic.
- **Step events for streaming.** They give the frontend a Claude-Code-like "worker is
  doing X" view without leaking worker reasoning to end users.
- **Inheritance rules.** Sub-agents inherit `user_id` and header overrides, so a turn's
  tenant and auth scope never widens. Not inheriting the parent transcript matches Claude
  and keeps worker context small.

**Alternatives rejected**:
- **A2A (Agent-to-Agent protocol) or another inter-agent communication layer.** Rejected:
  it adds a network hop, a second auth surface and a protocol dependency for what is an
  in-process call inside one binary. The Claude-style tool pattern already works in
  practice.
- **Exposing each agent as an MCP server that other agents consume as a `tool_backends`
  kind.** Rejected for the same reasons as A2A. It would also force every worker to be
  network-addressable and would lose session linkage and artifact-scope inheritance.
- **Agent-scoped (not version-scoped) agent-tool config.** Rejected: how the tool is used
  is controlled by the version's prompt, so the config must vary with it.
- **Targets that follow the target agent's DEPLOYED version.** Rejected: eval and training
  results would not be reproducible, every worker would have to be deployed, and `aikdm`
  could not receive a static topology. An optional pin was also rejected, because two
  resolution paths are more to specify and test.
- **Embedding the sub-transcript in the parent's tool-result block**, or keeping only the
  final answer. Rejected: parent rows bloat, sub-runs cannot be queried, and worker
  behaviour cannot be debugged.
- **Always mocking the agent tool in evals.** Rejected: evals and training would never
  exercise the real workers.
- **aikdm calling back into open-bbcd to run sub-agents.** Rejected: eval would then depend
  on a new internal runtime route and on server-side LLM calls, and the topology would
  split across two execution engines mid-eval.
- **One tool per allowed target.** Rejected: the tool count would grow with topology width.
  A single enum-typed `agent` tool matches Claude.
- **Streaming full sub-agent token output to the frontend.** Rejected: it leaks worker
  reasoning to end users and complicates the frontend contract.
- **Allowing self-spawn / cyclic topologies.** Rejected: only the depth cap would protect
  against runaway recursion. A DAG over pinned versions can be checked statically.

**Impact**:
- `current/assumptions.md` — scope assumption "Single AI agent on day one" replaced with
  "Single entrypoint agent; multi-agent via the agent tool". New locked decision
  "Multi-agent = an in-process agent tool over pinned versions; no inter-agent protocol"
  (dated 2026-09-30). Three new open questions: whether training may patch sub-agent notes,
  pinned-target bump ergonomics, and a per-turn LLM budget. Clarified the existing
  agent-operator open question.
- `current/constraints.md` — added `AGENT_TOOL_MAX_DEPTH` (default 3),
  `AGENT_TOOL_MAX_PARALLEL` (default 4), the DAG-topology rule and the target-status rule
  (`READY` | `DEPLOYED`, pinned, never re-resolved).
- `current/nfrs.md` — Performance: latency and LLM-spend amplification for multi-agent
  turns. Security: new sub-agent trust model (what is and isn't inherited; the sub-agent
  uses its own wiring). Observability: new "Sub-agent traces" subsection (child sessions).
- `current/glossary.md` — added Agent tool, Sub-agent, Sub-agent binding, Child session and
  Agent topology. The AG-UI row now lists `STEP_*` and `ARTIFACT_REF`.
- `current/bizbok/capabilities.md` — L1 "Agent lifecycle management" now mentions
  topologies. New L2 `agent-tool-configuration` (under Agent lifecycle management) and
  `sub-agent-dispatch` (under Deployed agent runtime; shared with BO chat). The
  `eval-scoring` L2 now runs real topologies. Two new capability→context/container map
  rows.
- `current/bizbok/information-map.md` — added Sub-agent binding and Child session rows.
- `current/bizbok/value-streams.md` — new "Multi-agent topology" stream.
- `current/ddd/access-model.md` — Admin CRUD policy extended with
  `agent-tool-configuration` / Sub-agent binding (admin-only, with the reason). New End-user
  `sub-agent-dispatch` policy. New tenant-scoping bullet: child sessions inherit the
  root's `user_id`.
- `current/ddd/context-map.md` — the agent-lifecycle → feedback-datasets / evaluation /
  deployed-runtime interfaces now mention sub-agent bindings and the transitive export.
  Diagram unchanged.
- `current/ddd/contexts/agent-lifecycle.md` — Agent version gains `agent_tool_enabled`. New
  Sub-agent binding entity. Invariants added: version-scoped config, pinned targets, DAG,
  verbatim inheritance. Clarified that the agent tool is not a `tool_backends` kind.
  Published surface gains `/agent_versions/{id}/configure/agents`.
- `current/ddd/contexts/feedback-datasets.md` — new Child chat session entity. Invariants
  added: only roots are dataset members, locking cascades to the session tree, feedback
  goes on root messages only. Read-only child-transcript route.
- `current/ddd/contexts/deployed-runtime.md` — new Child deployed session entity.
  Invariants added: children are hidden from the session list, delete cascades,
  `STEP_*` progress without sub-agent tokens, caps. The emitted AG-UI contract now lists
  `STEP_STARTED` / `STEP_FINISHED`.
- `current/ddd/contexts/evaluation.md` — eval-session transcripts nest sub-agent runs. New
  invariant "Evals run the real topology". `export.yaml` gains a `subagents` section.
- `current/ddd/contexts/training.md` — the new version copies agent-tool config verbatim.
  Hill-climb patches the root version only.
- `current/c4/containers.md` — open-bbcd purpose now covers the in-process agent tool.
  Data ownership adds `agent_version_subagent`, `agent_tool_enabled` and child-session
  linkage. aikdm published API notes that `evaluate` / `train-agent` run topologies
  in-process and that `generate-agent` does not generate agent-tool config. Diagram
  unchanged (no new edges).
- `current/c4/data-flows.md` — new "Multi-agent delegated turn" sequence diagram.
- `current/c4/integrations.md` — AG-UI contract: step events for sub-agent progress and
  `child_session_id` tagging. Eval-input contract: `agent_tool` + transitive `subagents`
  sections (`prompt-v1.yaml` unchanged). New "Agent tool" contract entry.
- `current/c4/deployment.md` — three new threat-model rows: sub-agent fan-out cost/DoS,
  confused deputy via sub-agent wiring, and child-session information disclosure.
- + `docs/architecture/logs/2026-09-30-multiagent-tools/README.md` (this file).

Not touched: the modularity tree (`modularity/open-bbcd/{agent-lifecycle,tool-runtime,
feedback-datasets,deployed-runtime,evaluation,training}`, `modularity/aikdm/{eval-scoring,
training-loop}`). Their purpose/scope text should be refined through
`/modularize --refine`.

**Links**:
- (user may add related spec PRs or tracker items before merge)
