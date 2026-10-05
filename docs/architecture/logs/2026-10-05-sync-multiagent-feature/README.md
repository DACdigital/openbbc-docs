---
date: 2026-10-05
codename: sync-multiagent-feature
kind: sync
---

# Sync — align current/ with 2026-10-01-multiagent-feature-design

## Driver
Post-approval sync of `docs/architecture/current/` with the following spec(s):
- `docs/superpowers/specs/2026-10-01-multiagent-feature-design.md` (approved and merged in
  openbbc-docs PR #13)

## Decision
Apply the deltas listed below to `docs/architecture/current/`. AMBIGUOUS deltas (if any) are
recorded here for human resolution but were NOT applied.

In short, current/ now matches the implementation contract of the agent tool.

- **AG-UI wire.**
  - `stepName = "<binding name>:<toolCallId>"`.
  - Child tool calls carry a `"<childSessionId>:"`-prefixed `toolCallId` and
    `rawEvent.childSessionId`.
  - The result travels on `TOOL_CALL_RESULT`, and the stream ends with `RUN_FINISHED` /
    `RUN_ERROR`.
  - Clients that ignore steps see worker tool calls as extra top-level calls.
- **Agent tool contract.** Text in, text out, plus a defined error-code set and the
  endpoint-name collision rule.
- **Child sessions.** Every per-session route on both surfaces resolves root sessions only,
  and a child id returns `404`. Children are read through new root-scoped child-transcript
  routes. Deployed children carry the **root's** `agent_id` and `user_id`. Close-draft locks
  the whole session tree, and no child can be created under a locked root.
- **Topology.** It is acyclic by construction: only `INITIALIZING`/`DRAFT` callers bind, only
  `READY`/`DEPLOYED` targets can be bound, and status only moves forward. A cycle check
  remains as defence in depth. Config is frozen after DRAFT, and forks copy it.
- **Delete guards.** A version or agent delete returns `409` when it would remove a binding
  target, a deployed child's version, or a version with locked BO sessions pinned to it.
  This replaces the stale "versions are never deleted".
- **Routes and migration.** Configurator writes live under `…/architecture/agents/*`. The
  close-draft mutation route is `POST /datasets/{id}/close-draft`. The migration head is
  `029_agent_tool`.

The temporary eval/training gate is deliberately **not** written into current/ as target
state (spec Deviation 11). `ddd/contexts/evaluation.md` "Evals run the real topology" stays.

## Rationale
Approved spec is the design contract; `current/` is the architecture-of-record and must reflect it.
Per project rule: L2 modularity nodes are inspirational — specs are the impl contract; sync applies
CONTRACT_SHAPE deltas without altering module Purpose, and only introduces new L1/L2 surface for
SCOPE_MOVE deltas.

## Alternatives rejected
- Leaving current/ stale until the next `/modularize` run — rejected: drift accumulates silently and
  a new dev reading current/ builds against the wrong contract.
- Auto-applying all deltas including AMBIGUOUS ones — rejected: silent overrides of prior arch-log
  decisions require human review.

## Impact

### CONTRACT_SHAPE deltas applied (no purpose change)
1. `c4/integrations.md § Contracts (AG-UI event stream)`:
   - full event list;
   - `stepName` and `rawEvent` fields;
   - prefixed child `toolCallId`;
   - `TOOL_CALL_RESULT` carries the result;
   - ordering per `agent` call;
   - the compatibility sentence is qualified: step-unaware SDKs see worker tool calls as
     extra top-level calls (spec Deviations 1, 2, 10).
2. `c4/integrations.md § Contracts (Agent tool)`:
   - presented only with at least one binding, immediately after `Skill`;
   - required properties only;
   - text result with no envelope;
   - error-code set;
   - `409` on endpoint-name collision.
3. `ddd/contexts/deployed-runtime.md § Domain events / § Invariants (CUSTOM) / § Published
   surface`: event lists corrected to `TOOL_CALL_RESULT`, `STEP_*`, `RUN_FINISHED`,
   `RUN_ERROR`. The emitted contract now points to `integrations.md`.
4. `ddd/contexts/deployed-runtime.md § Invariants (Sub-agent progress)`:
   - `stepName` format;
   - prefixed child ids;
   - `TOOL_CALL_RESULT` carries the result;
   - one `RUN_STARTED` and one `RUN_FINISHED`.
5. `ddd/contexts/deployed-runtime.md § Invariants + § Published surface`: the root-only rule
   for every per-session route, plus the new
   `GET /deployed/{agent_id}/sessions/{root_id}/children/{child_id}?user_id=X` route
   (spec Deviation 4).
6. `c4/data-flows.md § Multi-agent delegated turn`: diagram lines updated for the step name,
   the child-row columns, tagged child events, `TOOL_CALL_RESULT` and `RUN_FINISHED`.
7. `c4/data-flows.md § Feedback → dataset / § Deployment`:
   - the close-draft route;
   - the tree lock;
   - `RUN_FINISHED`.
8. `glossary.md`:
   - AG-UI row event list;
   - Child session row: root `user_id`/`agent_id` on deployed, header copy on BO, tree lock,
     `404` on per-session routes;
   - Agent topology row: acyclic by construction.
9. `bizbok/information-map.md`:
   - Child session row: root `user_id`/`agent_id`;
   - Sub-agent binding row: editable only in `INITIALIZING`/`DRAFT`, copied on every fork.
10. `constraints.md § Hard technical limits`:
    - migration head `029_agent_tool`;
    - boot validation of both caps;
    - DAG rule restated as acyclic by construction;
    - config frozen after DRAFT.
11. `ddd/contexts/agent-lifecycle.md § Invariants + § Published surface`:
    - "append-only / never deleted" replaced by forking plus delete guards (spec Deviation 9);
    - config frozen after DRAFT;
    - DAG by construction;
    - MCP and Agents tab routes corrected to `…/configure/<tab>` with writes under
      `…/architecture/<thing>/*` (spec Deviation 3);
    - version/agent delete `409`s.
12. `ddd/contexts/feedback-datasets.md`:
    - child entity: header copy, never created under a locked root;
    - close-draft mutation route (spec Deviation 6) and tree lock;
    - `assign-dataset` on a child returns `404`, not `409` (spec Deviation 5);
    - lock-cascade rationale corrected;
    - new root-only per-session invariant;
    - published-surface datasets line.
13. `bizbok/value-streams.md`:
    - Feedback → dataset step 4: route and tree lock;
    - Multi-agent topology step 2: DRAFT-only config, forks copy it.
14. `ddd/access-model.md`:
    - Admin agent-tool row: DRAFT-only writes, delete `409`s;
    - End-user `sub-agent-dispatch` row: child-transcript route, `404` elsewhere;
    - tenant-scoping child bullet: root `user_id` + `agent_id`.
15. `nfrs.md § Security (Sub-agent trust model)`:
    - deployed children carry the root's `user_id` and `agent_id`;
    - children are readable only through the root-scoped route;
    - worker tool output (not reasoning) is visible on the root stream.
16. `c4/deployment.md § Threat model`:
    - fan-out row: acyclic by construction;
    - child-session disclosure row: root partition, `404`s, root-scoped reads, tool results
      forwarded but no text tokens or child `ARTIFACT_REF`.
17. `c4/containers.md § open-bbcd + § postgres`:
    - the orchestrator (not the builder) owns the agent tool;
    - child-session columns, root `agent_id`/`user_id`;
    - migration head `029_agent_tool`.
18. `modularity/open-bbcd/README.md § Purpose` (L1): the agent tool is added and dispatched
    by the orchestrator. Purpose is unchanged.
19. `bizbok/capabilities.md`:
    - `agent-tool-configuration`: caller-status check, by-construction DAG, forks copy;
    - `sub-agent-dispatch`: orchestrator-owned, root `agent_id`, `TOOL_CALL_RESULT`
      forwarding, root-scoped child reads.
20. `ddd/contexts/evaluation.md § Published surface (export.yaml)`: documents the one-shot
    fail-forward `409` while multi-agent eval is unsupported (spec Deviation 8). The "Evals
    run the real topology" invariant is untouched.
21. Modularity L2 READMEs, wording within Purpose only:
    - `open-bbcd/agent-lifecycle`: `agent_version_subagent`, the Agents tab and architecture
      writes, delete `409`s;
    - `open-bbcd/feedback-datasets`: child-session columns, child-transcript route, root-only
      rule, tree lock;
    - `open-bbcd/deployed-runtime`: child columns, corrected event list, child route,
      cascade;
    - `open-bbcd/tool-runtime`: the agent tool is not built or dispatched here.

### SCOPE_MOVE deltas applied (new surface / ownership / boundary)
- `ddd/contexts/deployed-runtime.md § Aggregates & entities` (Child deployed session):
  - a child carries the **root's** `agent_id` (previously unspecified) as well as its
    `user_id`, so a session tree stays inside the root agent's partition;
  - deleting the root's agent cascades the tree;
  - deleting a worker agent that a child pins is refused with `409`;
  - roots keep `agent_version_id` NULL (spec Deviation 7).

### AMBIGUOUS deltas NOT applied — human review required
- `ddd/contexts/agent-lifecycle.md § Invariants / § Published surface`, plus
  `ddd/access-model.md` (Admin row) and `modularity/open-bbcd/agent-lifecycle/README.md`.
  This is the config-write half of the eval/training gate.
  - Spec sentence: "Second, a version with eval or training work in flight can't turn the
    agent tool on or change its bindings."
  - Repository invariants: "caller has no `evals` row in `PENDING`/`IN_PROGRESS` and no
    `training_sessions` row with `parent_version_id` = caller in `PENDING`/`IN_PROGRESS` |
    `ErrEvalOrTrainingActive` → `409`".
  - Why ambiguous: this rule is temporary, like the eval gate the spec's Deviation 11 keeps
    out of current/. Recording it would put temporary state into current/, while leaving it
    out leaves the configurator `409` undocumented.

### Intentionally unchanged (non-deltas)
- **Spec Deviation 11 (temporary eval/training gate).** These describe the `multiagent-eval`
  target state recorded by `2026-09-30-multiagent-tools`, which this spec defers rather than
  overrides, so they all stay as written:
  - `ddd/contexts/evaluation.md` ("Evals run the real topology", nested eval transcripts);
  - `bizbok/value-streams.md` Multi-agent topology step 5;
  - `nfrs.md` eval wall-clock and nested eval transcripts;
  - `constraints.md` aikdm cap clause;
  - `c4/integrations.md` eval-input `agent_tool`/`subagents`;
  - `bizbok/capabilities.md` `eval-scoring`;
  - `ddd/context-map.md` agent-lifecycle → evaluation row;
  - `c4/containers.md § aikdm`;
  - `modularity/aikdm/README.md`;
  - the aikdm narrative in `c4/data-flows.md`;
  - `ddd/contexts/training.md`.
- **Below the implementation line.** None of these is recorded in current/:
  - SSE write-deadline fix;
  - `Turn` sink ownership;
  - per-turn tool-handler `Close()`;
  - vendored htmx `response-targets`;
  - `statusFor` / `renderAgentsError`;
  - JSON error-body shape;
  - rollback / mixed-rollout guidance;
  - test TRUNCATE lists.

## Links
- Source spec(s): `docs/superpowers/specs/2026-10-01-multiagent-feature-design.md`
  (https://github.com/DACdigital/openbbc-docs/pull/13)
- Related prior arch-log entries:
  - `docs/architecture/logs/2026-09-30-multiagent-tools/README.md`. Relationship:
    - **Extends / confirms:** implements its agent tool, child sessions and caps, and keeps
      the bind-time cycle refusal as defence in depth.
    - **Supersedes in part:** the AG-UI wire naming ("step name = binding `name`",
      `child_session_id`, "`TOOL_CALL_END` carries the result"), per the spec's explicit
      Deviations 1–2.
    - **Refines:** "cycles are refused at bind time" becomes acyclic by construction.
    - **Leaves untouched:** the eval-topology decisions (Deviation 11).
  - `docs/architecture/logs/2026-10-01-sync-deployed-runtime-artifacts/README.md`.
    Relationship:
    - **Confirms:** SCOPE_MOVE H (per-session artifact scope, no child `ARTIFACT_REF`, child
      artifact routes return `404`).
    - **Corrects:** the leftover feedback-datasets lock-cascade rationale ("so their
      `artifact_ref`s stay resolvable for retrieval").
