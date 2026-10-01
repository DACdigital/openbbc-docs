# multiagent-feature — agent tool, sub-agent bindings, and child sessions on BO chat + deployed runtime

**Date:** 2026-10-01
**Status:** Draft — pending `/spec-review`
**Modularity attachment:** `docs/architecture/current/modularity/open-bbcd/tool-runtime/` (L2, primary). It also touches these sibling L2s:
- `agent-lifecycle`: config, fork copy, delete guards
- `feedback-datasets`: BO child sessions, dataset rules
- `deployed-runtime`: deployed child sessions, AG-UI wire
- `evaluation` and `training`: temporary gate
- `artifacts`: child-session rules

**Arch-of-record:**
- `agent-tool-configuration` L2 capability (under Agent lifecycle management) and `sub-agent-dispatch` L2 capability (under Deployed agent runtime, shared with BO chat) in `bizbok/capabilities.md`.
- Locked decision "Multi-agent = an in-process agent tool over pinned versions; no inter-agent protocol" in `assumptions.md`.
- Introduced by `docs/architecture/logs/2026-09-30-multiagent-tools/` (PR #9). The artifact-scope rules were narrowed by `docs/architecture/logs/2026-10-01-sync-deployed-runtime-artifacts/` (PR #12).

**Grounded against:** OpenBBC `main` @ `7566151`, which includes chat-artifacts #53 and deployed-runtime-artifacts #54–#56. All file:line references are relative to `open-bbcd/` at that commit, except paths starting with `scripts/`, which are relative to the OpenBBC repo root.

## Business value / Why

Multi-agent support has been on the roadmap since the start, and an existing downstream
project needs it now. That project wants one entrypoint agent that delegates to
specialised agents, such as a planner → coordinator → workers topology. Today every
`open-bbcd` turn runs exactly one agent version. The only way to get specialisation is one
very large prompt with every tool attached, which is hard to test, hard to improve with
the hill-climb loop, and grows context without bound.

The merged architecture picked the Claude Code sub-agent pattern: a built-in `agent` tool,
pinned target versions, and child sessions. This spec implements it in `open-bbcd` on both
chat surfaces. Success means:

- **An admin composes a topology without code.** In the BO configurator they tick "Enable
  agent tool" on a version and bind existing READY or DEPLOYED versions of any agent (including older versions of its own) by
  name and note. There are no new services and no MCP server per agent.
- **The root agent delegates, and the admin can see it happen.** In BO test-chat, an `agent`
  call renders as a live card, and the worker's full transcript is one click away at any
  nesting depth.
- **End users see one agent with visible progress.** On the deployed AG-UI stream,
  sub-agent work appears as standard `STEP_STARTED` / `STEP_FINISHED` events plus the
  worker's tool calls. Worker reasoning tokens and worker artifacts never reach the end
  user.
- **Topologies are reproducible and safe by construction.**
  - Targets are pinned versions, and config is frozen once a version leaves `DRAFT`.
  - Only `INITIALIZING`/`DRAFT` versions can bind, and only `READY`/`DEPLOYED` versions can
    be bound. Status only moves forward, so a caller never has incoming edges and the
    topology is acyclic by construction. An explicit cycle check stays as defence in depth.
  - Depth and fan-out are capped.

Eval and training over topologies is a follow-up spec (`multiagent-eval`). Until it lands,
two rules hold. First, a version with the agent tool enabled can't start eval or training
work. Second, a version with eval or training work in flight can't turn the agent tool on
or change its bindings. Together they mean no score or training run is ever produced from
an export that leaves out the topology.

## Change level

**C2.** The spec changes:

- **Postgres schema.** Migration `029_agent_tool.sql`:
  - a column on `agent_versions`;
  - a new `agent_version_subagent` table;
  - parent-link columns on `chat_sessions` and `deployed_sessions`.
- **Published REST surface.**
  - New configurator routes.
  - New child-transcript read routes on BO and deployed.
  - A rule that every other per-session route on both surfaces resolves root sessions only.
  - New `409` responses on eval create, start and export, on training create and start, on
    agent and version delete, and on config writes while eval or training work is active.
- **AG-UI event contract emitted to client frontends.** `STEP_*` events, plus a
  `childSessionId` tag and an id prefix on forwarded tool-call events.
- **LLM tool contract.** A new built-in `agent` tool definition.
- **Session-scope trust path.** Child sessions inherit the root's `user_id` and `agent_id`
  (deployed) and its header context, and child reads are scoped through the root.
- **`open-bbcd` boot-time configuration.** New env vars `AGENT_TOOL_MAX_DEPTH` and
  `AGENT_TOOL_MAX_PARALLEL`.

Per `docs/process/README.md`, a change to DB, API, events or contracts is C2.

## Scope

### In scope

- **Migration `029_agent_tool.sql`.** See *Contracts → Schema*.
- **Agent-tool config on the caller version:**
  - the `agent_versions.agent_tool_enabled` checkbox;
  - `agent_version_subagent` binding rows: pinned `target_version_id`, tool-facing `name`
    and prompt `note`.

  Server-side checks on save, enforced in the repository under row locks:
  - the caller's status is `INITIALIZING`/`DRAFT`;
  - the target's status is `READY`/`DEPLOYED`;
  - the caller has no `PENDING`/`IN_PROGRESS` eval or training;
  - the binding would not create a cycle (defence in depth).
- **Version fork copies agent-tool config.** `insertVersionFromPromptsTx`
  (`internal/repository/agent_version.go:311-347`) copies `agent_tool_enabled` and every
  binding row, in the same way it already copies `agent_version_mcp_backend`. This covers
  SavePrompts (DRAFT), LandPrompts (READY) and training Complete.
- **BO configurator tab "Agents":**
  - the agent-tool checkbox;
  - the bindings table (add, edit note, delete);
  - a target picker over `READY`/`DEPLOYED` versions of any agent.

  It uses the same htmx row-swap pattern as the MCP tab (`web/templates/configurator/mcp.html`,
  `partials.html`). Outside `INITIALIZING`/`DRAFT` the tab is read-only, and any write
  returns `409`.
- **The built-in `agent` tool.**
  - **Owner.** The orchestrator owns it end to end. It inserts the `agent` tool definition
    immediately after `Skill` in the tool list `Composite.Tools()` returns, intercepts
    `agent` calls in its dispatch loop, and hands them to an internal `subAgentRunner`.
  - **Name collision.** HTTP endpoint tools are not prefixed (`composite.go:54-67`,
    `builder.go:75`). Enabling the tool or adding a binding is refused with `409`
    `ErrToolNameCollision` when the agent's architecture has an endpoint whose **sanitised**
    tool name (`sanitizeToolName`, defined at `llm/tools/mock.go:96` and applied at `builder.go:75`) is `agent`. The same sanitised
    comparison is used at turn time. At turn time, if one appears anyway, the orchestrator omits the `agent` tool
    and logs a warning, rather than sending a duplicate name the provider would reject.
  - **Import cycle.** `agent` never goes through `Composite.Call`, so there is no
    `chat → tools → chat` import cycle.
  - **Execution.** The runner re-enters `chat.Orchestrator.Turn` in-process for the pinned
    target version, passing a new `TurnOpts`.
- **Child sessions:**
  - one per `agent` call, in `chat_sessions` (BO) or `deployed_sessions` (deployed);
  - linked to the parent by `parent_session_id` + `parent_tool_call_id`, with a `depth`;
  - inherit the request context: forwarded headers, the header-routing envelope, and BO
    session header overrides;
  - deployed children also inherit the root's `user_id` **and** `agent_id`.
- **Parallel sub-agent fan-out.** Within one assistant step, `agent` calls run concurrently,
  bounded by `AGENT_TOOL_MAX_PARALLEL`. Non-agent tool calls stay sequential, as today.
  Results are written in the original `tool_use` order.
- **Depth cap.** `AGENT_TOOL_MAX_DEPTH`: the root is depth 0, and a call from a session at
  depth `AGENT_TOOL_MAX_DEPTH` is refused with a tool error.
- **`childSink` stream adapter.**
  - Brackets each child run with `STEP_STARTED` / `STEP_FINISHED`.
  - Forwards the child's tool-call and tool-result events tagged with `childSessionId`.
  - Passes events from deeper levels through unchanged.
  - Drops the child's run, text, `ARTIFACT_REF` and error events.
- **Sink ownership moves to the handlers.** `Turn` never closes its sink, and both turn
  handlers `defer sink.Close()`. This also fixes the current behaviour: `Turn` closes only on
  success (`orchestrator.go:486`), and its `failTurn` paths return without closing.
- **Every turn closes its tool handler.**
  - `Turn` defers `Close()` on the handler that `Build` returns, when it implements
    `io.Closer`.
  - `Composite` gains `Close()`, which closes its MCP backends.
  - This covers child turns and also fixes today's root-path MCP session leak:
    `MCPClientBackend.Close` is only called in `handler/backends.go:313`.
- **Root-only per-session routes.** Every existing per-session route on both surfaces
  resolves root sessions only and returns `404` for a child id. Children are read only
  through the new child-transcript routes.
- **BO chat surface:**
  - root-only session list;
  - a live sub-agent card in `web/static/chat.js` and a server-rendered card in the history
    view;
  - a read-only child transcript route;
  - close-draft locks the whole session tree;
  - children can't be spawned under a locked root.
- **Deployed surface:**
  - root-only session list;
  - a child transcript read route, scoped through the root;
  - `DELETE` of a root cascades to its children;
  - `STEP_*` and tagged tool-call events on the AG-UI SSE stream.
- **Sub-agent artifact rules.** The deployed-runtime-artifacts spec (§ *Sub-agents*)
  assigns these to whichever of the two specs lands second. That spec has shipped, so this
  one implements and tests them:
  - artifact scope is per session, with no inheritance;
  - child tool results are recorded on the child only;
  - no `ARTIFACT_REF` is forwarded for child tool results;
  - artifact routes return `404` for child sessions;
  - refs written by the LLM are never trusted;
  - the BO child transcript renders child refs as labels without a link.
- **Delete guards.** Deleting an agent or a version returns `409` when it would remove:
  - a binding target;
  - a deployed child's version;
  - a version with locked BO sessions pinned to it.
- **Temporary eval and training gate.** It applies when the version has
  `agent_tool_enabled`.
  - `409` on eval create, export and start, and on training create and start.
  - Export and start also move a `PENDING` eval or training session to `FAILED`, so
    drainers never retry it.
  - The BO Evaluate and Train buttons render disabled with a tooltip.
- **SSE write-deadline fix.**
  - The BO chat turn and deployed turn handlers clear the per-connection write deadline
    once the stream opens (`http.ResponseController(w).SetWriteDeadline(time.Time{})`).
  - Multi-agent turns routinely exceed the server-wide `WriteTimeout = 30s`
    (`internal/handler/api.go:29-33`), and nothing clears it today.
  - The 30s timeout stays in force for every non-streaming route.

### Out of scope

- **Eval and training over multi-agent topologies** (`multiagent-eval`):
  - the transitive `subagents` export in `eval-input.yaml`;
  - running the topology in-process in aikdm `evaluate` / `train-agent`;
  - lifting the gate.

  `multiagent-eval` must implement this spec's text-only `agent` tool contract unchanged in
  aikdm, so evals exercise the production contract.
- **aikdm `generate-agent` producing agent-tool config.** Bindings are always written by an
  admin.
- **Training patches to binding `note`s.** This is an open question in `assumptions.md`;
  v1 copies notes verbatim.
- **A "bump pinned targets" UI** and a "which callers pin this version" view. Both are open
  questions in `assumptions.md`.
- **A per-turn token or cost budget.** Only the depth and parallel caps ship.
- **Header overrides on deployed sessions.** This gap already exists: deployed sub-agents
  use static `tool_backends.config` credentials, as deployed roots do.
- **Passing artifacts across the agent boundary**, in either direction. This matches the
  current arch. A sub-agent that needs file content gets it through its own tools, and the
  caller can name the file in `prompt` for the worker's tools to fetch.
- **Streaming sub-agent text tokens** to BO or AG-UI clients.
- **Server-side serialisation of concurrent turns on one session.** It is unchanged from
  today; see *Risks*.
- **A BO session delete route.** None exists today, and none is added.

## Contracts

### Env vars (read in `internal/config/config.go`)

| Var | Default | Meaning |
|-----|---------|---------|
| `AGENT_TOOL_MAX_DEPTH` | `3` | Maximum child depth; the root is 0. A call from a session at this depth returns a tool error. Must be an integer ≥ 1; any other value fails boot with an error naming the var. |
| `AGENT_TOOL_MAX_PARALLEL` | `4` | Maximum number of `agent` calls running at the same time per assistant step, per session node. Extra calls wait on the semaphore. Must be an integer ≥ 1; any other value fails boot. |

### Schema — migration `029_agent_tool.sql`

```sql
-- +goose Up
ALTER TABLE agent_versions
  ADD COLUMN agent_tool_enabled BOOLEAN NOT NULL DEFAULT false;

CREATE TABLE agent_version_subagent (
  caller_version_id UUID NOT NULL REFERENCES agent_versions(id) ON DELETE CASCADE,
  target_version_id UUID NOT NULL REFERENCES agent_versions(id) ON DELETE NO ACTION,
  name              TEXT NOT NULL CHECK (name ~ '^[a-z][a-z0-9_-]{0,39}$'),
  note              TEXT NOT NULL DEFAULT '',
  created_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
  PRIMARY KEY (caller_version_id, name),
  UNIQUE (caller_version_id, target_version_id),
  CHECK (caller_version_id <> target_version_id)
);
CREATE INDEX idx_avs_target ON agent_version_subagent(target_version_id);

ALTER TABLE chat_sessions
  ADD COLUMN parent_session_id   UUID REFERENCES chat_sessions(id) ON DELETE CASCADE,
  ADD COLUMN parent_tool_call_id TEXT,
  ADD COLUMN depth               INT NOT NULL DEFAULT 0,
  ADD CONSTRAINT chat_sessions_parent_link_chk
    CHECK ((parent_session_id IS NULL) = (parent_tool_call_id IS NULL)),
  ADD CONSTRAINT chat_sessions_depth_chk
    CHECK ((parent_session_id IS NULL) = (depth = 0));
CREATE UNIQUE INDEX idx_chat_sessions_parent ON chat_sessions(parent_session_id, parent_tool_call_id)
  WHERE parent_session_id IS NOT NULL;

ALTER TABLE deployed_sessions
  ADD COLUMN parent_session_id   UUID REFERENCES deployed_sessions(id) ON DELETE CASCADE,
  ADD COLUMN parent_tool_call_id TEXT,
  ADD COLUMN depth               INT NOT NULL DEFAULT 0,
  ADD COLUMN agent_version_id    UUID REFERENCES agent_versions(id) ON DELETE NO ACTION,
  ADD CONSTRAINT deployed_sessions_parent_link_chk
    CHECK ((parent_session_id IS NULL) = (parent_tool_call_id IS NULL)),
  ADD CONSTRAINT deployed_sessions_child_version_chk
    CHECK ((parent_session_id IS NULL) = (agent_version_id IS NULL)),
  ADD CONSTRAINT deployed_sessions_depth_chk
    CHECK ((parent_session_id IS NULL) = (depth = 0));
CREATE UNIQUE INDEX idx_deployed_sessions_parent ON deployed_sessions(parent_session_id, parent_tool_call_id)
  WHERE parent_session_id IS NOT NULL;

-- +goose Down
-- Children first: once the parent link is dropped, a former child would read as a
-- root session of its agent and appear in that agent's session list.
DELETE FROM deployed_sessions WHERE parent_session_id IS NOT NULL;
DELETE FROM chat_sessions     WHERE parent_session_id IS NOT NULL;
-- then drop the indexes, constraints, columns and table above in reverse order
```

**Semantics:**

- **Key of `agent_version_subagent`.** The natural primary key `(caller_version_id, name)`
  deviates on purpose from `persistence.md`'s "no natural-key-only tables":
  - it matches the repo's existing wiring tables, `agent_version_mcp_backend`
    (migration 015) and `agent_endpoint_backend` (migration 017);
  - inside the version aggregate, a binding's identity is `(caller, name)`;
  - the delete route and the notes form key on `name`.

  The repo has no triggers, so every note update sets `updated_at = now()` explicitly in
  the `UPDATE`.

- **Existing rows.** Every existing session becomes a root (`depth = 0`, NULL links), and
  every existing version gets `agent_tool_enabled = false`. No backfill.
- **Rollback.** `goose down` deletes every child session, and their messages and
  session-artifact rows go with them through the existing `ON DELETE CASCADE` FKs.
  - **App-only rollback is unsafe.** A previous binary running on schema 029 treats
    children as roots, so they would be listed and addressable. Rolling back the app must
    be paired with `goose down`.
  - **Mixed-version rollouts.** In any window where old and new binaries run together,
    leave the agent tool disabled on every version until the rollout completes. The Helm
    chart ships one replica by default, so there is normally no such window.
- **Root deployed sessions** keep `agent_version_id IS NULL` and resolve the DEPLOYED
  version on each turn, as today.
- **Child deployed sessions:**
  - carry the pinned target in `agent_version_id`;
  - carry the **root's** `agent_id` and `user_id`.

  Using the root's `agent_id` keeps a whole tree inside one agent's partition. Deleting the
  root's agent cascades the entire tree. Deleting a worker agent never cascades into another
  agent's users' transcripts; it is refused with `409` by the delete guard, as below.

  Both new FKs into `agent_versions` are `NO ACTION`, not `RESTRICT`. `NO ACTION` is checked
  at the end of the statement, so one `DELETE FROM agents` that cascades through a binding
  *and* its target (an agent binding its own older version) succeeds once the referencing
  rows are gone too.
- **Child BO sessions:**
  - `agent_version_id` (the existing column) is the pinned target;
  - `backend_header_overrides` is copied from the root at creation (at runtime the
    overrides flow through `ctx`);
  - `locked_at` is NULL. Creating a child under a locked root is refused, as the next
    section describes.

  `chat_sessions` has no `user_id` column.

### Repository invariants

Every config write runs in one transaction. The transaction starts with
`SELECT … FOR UPDATE` on the caller's `agent_versions` row, and for a binding add also on
the target's row, locking the two rows in ascending `id` order. This serialises the status
checks against concurrent status transitions: Finalize `INITIALIZING → PENDING`, Land,
deploy and undeploy.

| Operation | Rule | Failure |
|-----------|------|---------|
| Toggle `agent_tool_enabled`; add, update or delete a binding | caller `status ∈ {INITIALIZING, DRAFT}` | `ErrVersionLocked` → `409` |
| same | caller has no `evals` row in `PENDING`/`IN_PROGRESS` and no `training_sessions` row with `parent_version_id` = caller in `PENDING`/`IN_PROGRESS` | `ErrEvalOrTrainingActive` → `409` |
| Add a binding | target `status ∈ {READY, DEPLOYED}` | `ErrTargetNotRunnable` → `400` |
| Add a binding | caller not reachable from target (recursive CTE over `agent_version_subagent` from `target_version_id`); target ≠ caller. Unreachable under the status rules (a caller has no incoming edges); kept as defence in depth. | `ErrTopologyCycle` → `400` |
| Add a binding | `name` is unique per caller; the target is unique per caller | `ErrBindingConflict` → `409` |
| `insertVersionFromPromptsTx` | copy `agent_tool_enabled` and all `agent_version_subagent` rows, rewriting `caller_version_id` to the new version | — |
| Create a child session (`CreateChildSession`) | inside one transaction: lock the **root** row with `FOR SHARE` (it conflicts with close-draft's `UPDATE … SET locked_at`, which `FOR KEY SHARE` would not, cf. `session_artifacts.go:264-267`, while parallel sibling spawns can still share it); for BO, read the root's `locked_at` and refuse if it is set; insert the child with `depth = parent.depth + 1` | BO locked root → `ErrSessionLocked`, surfaced to the caller LLM as the `agent` tool error `session_locked`. Any other store error becomes the `agent` tool error `subagent_failed`. It is logged at warn level when the parent or root no longer exists (a legitimate race with a concurrent delete), and at error level otherwise, as a programming error. The stream is already open with `200`, so no HTTP status is involved. |
| Version delete (`POST /agent_versions/{version_id}/delete`) | the pre-check (`agent_version.go:76-88`) also refuses when the version is a binding target, is the `agent_version_id` of any deployed child session, or has any `chat_sessions` row with `locked_at IS NOT NULL` | `ErrVersionReferenced` → `409` |
| Agent delete (`POST /agents/{agent_id}/delete`, `repository/agent.go:134-158`) | the same three conditions, checked for every version of the agent before the cascade, **ignoring references that come from the agent's own cascade set**. That set is: bindings whose caller is one of its versions; deployed sessions with its `agent_id` (roots and their children); and BO session trees whose **root's** `agent_version_id` belongs to the agent, including their descendants. A locked BO child pinned to one of its versions under **another** agent's root is not in the set, so the delete is still refused | `ErrVersionReferenced` → `409` |
| Any FK violation `23503` raised by the new `NO ACTION` FKs | translated to `ErrVersionReferenced` in the repository, so a missed pre-check still returns `409`, not `500` | `ErrVersionReferenced` → `409` |

New sentinels go in `internal/types/errors.go` and are mapped in `handler.Error`:
- `409`: `ErrVersionLocked`, `ErrEvalOrTrainingActive`, `ErrBindingConflict`,
  `ErrVersionReferenced`, `ErrMultiAgentEvalUnsupported`, `ErrToolNameCollision`;
- `400`: `ErrTargetNotRunnable`, `ErrTopologyCycle`. These are semantic validation
  failures, mapped to `400` like the repo's analogous `ErrTrainingSessionEvalNotEligible`.
  `handler.Error` therefore needs no new status code.

There is no child-specific sentinel. A child id on a root-only route returns the existing
`ErrNotFound` (`404`) on both surfaces, so the API never reveals whether a child exists.

`ErrVersionInUse` (`types/errors.go:60`, "this version is currently deployed") is not
reused.

Every JSON error body uses the existing `handler.ErrorResponse` shape
`{"error": "<sentinel text>"}` (`internal/handler/handler.go:14-16`). This deliberately
follows the repo's established shape rather than the `docs/conventions/api.md` envelope.
The sentinel text for `ErrMultiAgentEvalUnsupported` is the single string
**`multi-agent eval unsupported: evaluating or training versions with the agent tool enabled is not supported yet`**,
used verbatim in response bodies, in `evals.error_message`, and in
`training_sessions.error_message`.

**Versioning and idempotency (deviations from `api.md`).**
- **Versioning.** The repo has no `/v1/` URL prefix and no `Idempotency-Key` handling.
  This spec adds neither. Every change is additive: new routes, plus new `404`/`409`/`400`
  responses only for states that this feature creates. No existing response changes, so
  no version bump is needed.
- **Retry safety.** The new mutations are safe to retry without an idempotency key:
  - the toggle sets an absolute value;
  - a duplicate binding add returns `409 ErrBindingConflict`;
  - a note save writes absolute values;
  - deleting an already-deleted binding returns `404`.

### REST — BO configurator (`internal/handler/configurator.go`)

The routes deliberately follow the existing configurator pattern:
- the tab lives under `…/configure/<tab>`;
- writes live under `…/architecture/<thing>/…`, as in
  `/agent_versions/{version_id}/architecture/mcp/{backendID}/toggle` (`api.go:269-283`);
- the `snake_case` `agent_versions` segment and the verb segments are kept for consistency
  with the neighbouring routes.

| Method + path | Body | Response |
|---------------|------|----------|
| `GET /agent_versions/{version_id}/configure/agents` | — | HTML tab with the checkbox, the bindings table and the target picker. Read-only rendering unless the status is `INITIALIZING`/`DRAFT` and no eval or training work is active. When eval work blocks edits, a banner says so. |
| `POST /agent_versions/{version_id}/architecture/agents/toggle` | form `enabled=on\|off` | `200` checkbox fragment; `409` when locked or eval work is active |
| `POST /agent_versions/{version_id}/architecture/agents` | form `name`, `target_version_id`, `note` | `200` bindings-table fragment; `409` when locked, on conflict, or when eval work is active; `400` when the target is not runnable or a cycle would form. |
| `POST /agent_versions/{version_id}/architecture/agents/notes` | form `note[<name>]` for each binding | `200` tab fragment with `Saved=true`; `409` when locked or eval work is active. `note[x]` for a name that is not bound is silently ignored, as the MCP notes route does (`configurator.go:655-659`). |
| `POST /agent_versions/{version_id}/architecture/agents/{name}/delete` | — | `200` bindings-table fragment; `409` when locked or eval work is active; `404` when `name` is not bound on this version |

**Error rendering.**
- **Body format.** Every configurator write returns an **HTML fragment**, never JSON. On
  `4xx` the body is a small error fragment containing the sentinel text.
- **Why htmx needs help.** The shipped htmx 2.0.4 (`web/static/htmx.min.js`) does not swap
  `4xx` responses by default.
- **Mechanism.** The Agents tab template sets `hx-target-error="#agents-error"` on its forms
  and buttons, and loads htmx's `response-targets` extension. That extension is not in the repo yet: it
  is newly vendored as `web/static/htmx-response-targets.min.js`, recorded in
  `web/static/VENDORED.md` like `htmx.min.js`, and enabled with
  `hx-ext="response-targets"` on the tab root. Error fragments
  therefore swap into a dedicated `#agents-error` slot.
- **Effect.** The bindings table and checkbox stay unchanged on error, and the real status
  codes are kept for tests and non-htmx clients.
- **Status mapping without JSON.** `handler.Error` (`internal/handler/handler.go:67-127`)
  always writes a JSON `ErrorResponse`, and the MCP tab calls it directly
  (`configurator.go:643`). So the sentinel → status `switch` inside `handler.Error` is
  extracted into `statusFor(err) int`.
  - `handler.Error` keeps calling `statusFor`, so its behaviour is unchanged for every
    existing route.
  - The Agents-tab write handlers call a new `renderAgentsError(w, err)`. It writes
    `statusFor(err)` and the HTML error fragment.
  - The Agents-tab handlers never call `handler.Error`.

The target picker lists `READY`/`DEPLOYED` versions of every agent except the caller
version itself. Entries are grouped by agent and labelled `<agent name> · v<n> · <status>`.

### REST — root-only rule and child transcripts

**Rule.** Every existing per-session route on both surfaces resolves **root** sessions only.
A child id returns `404` (`ErrNotFound`, indistinguishable from an unknown id) with no side effect.
- **BO:**
  - `GET …/chat/{session_id}`
  - `PATCH …/title`
  - `POST …/turn`
  - `GET|POST …/headers`
  - `…/messages/{message_id}/feedback`
  - `…/assign-dataset`
  - `POST|GET …/artifacts[/{path...}]`
  - `GET …/pending-artifacts`
  - `DELETE …/pending-artifacts/{id}`
- **Deployed:**
  - `GET|DELETE …/sessions/{session_id}`
  - `PATCH …/title`
  - `POST …/turn`
  - `POST|GET …/artifacts[/{path...}]`
  - `GET …/pending-artifacts`
  - `DELETE …/pending-artifacts/{id}`

How each query changes:
- **Getters used by route preambles** add `AND parent_session_id IS NULL`. These are BO
  `ChatRepository.GetSession` (`repository/chat.go:52-76`) and deployed
  `DeployedRepository.GetSession` (`repository/deployed.go:56-71`).
- **Deployed writes** add `AND parent_session_id IS NULL AND agent_id = $agent` and return
  `ErrNotFound` on 0 rows. These are `UpdateSessionTitle` and `DeleteSession`
  (`repository/deployed.go:114-160`, which today filter only by `id AND user_id`).
- **`GetSessionByID` is exempt.** It is `DeployedRepository.GetSessionByID`
  (`deployed.go:76-86`), used by `DeployedChatStore.EnsureSession`
  (`chat/deployed_store.go:40-43`) so child turns can run. No route preamble calls it.
- **BO preambles that tolerate NotFound.** Several BO handlers treat `ErrNotFound` from
  `GetSession` as "session not created yet" and carry on. With children filtered out of
  `GetSession`, a child id would pass as a fresh session. Each of these handlers therefore
  first calls a new `ChatRepository.IsChildSession(id)` and returns `404` when it is true:
  - `ChatView` (`chat.go:441-477`), which otherwise renders an unscoped `LoadMessages`;
  - `UpdateSessionTitle` (`chat.go:570-587`);
  - `ShowHeaderOverridesModal` (`chat.go:752-762`, which calls no getter today) and
    `UpdateHeaderOverrides` (`chat.go:771-806`);
  - `AssignDatasetModal` / `AssignDataset` (`chat.go:825-874`) and `UnassignDataset`
    (`chat.go:879-886`), none of which call a getter today;
  - `Turn` (`chat.go:615-671`), whose lazy-create path would otherwise reach
    `EnsureSession`'s `INSERT … ON CONFLICT DO NOTHING` (`repository/chat.go:33-49`) and
    adopt the child.
- **BO writes and reads with their own queries.** These add `AND parent_session_id IS NULL`:
  - `ChatRepository.UpdateSessionTitle` (`repository/chat.go:78`), which today filters only
    on `id` and version;
  - `ChatRepository.SetSessionHeaderOverrides`;
  - the session reads inside `DatasetRepository.AssignSessionToDraft`
    (`repository/dataset.go:307-349`), which returns `ErrNotFound` for a child.
- **Feedback is scoped to the path session.** `FeedbackRepository.Get` / `Upsert` / `Delete`
  (`repository/feedback.go:28,73,102`) key on `message_id` alone today, and the handlers
  (`handler/feedback.go:37-104`) ignore the `session_id` path segment.
  - A new check, `FeedbackRepository.MessageInRootSession(sessionID, messageID)`, joins
    `chat_messages → chat_sessions` and requires `m.session_id = <path session>` and
    `s.parent_session_id IS NULL`.
  - All three handlers (`Footer`, `Upsert`, `Delete`) call it first. When it returns false
    they respond `404` and touch nothing.
  - `Get` keeps its current meaning: `ErrNotFound` still means "no feedback yet". That is
    why the scope check must be separate. `Footer` discards `Get`'s error
    (`handler/feedback.go:41`), so a scoped `Get` alone would return `200` with an empty
    footer.
  - Result: a child's message id posted under a root's path gets `404`.
- **Artifact preamble.** BO `boSession` (`handler/artifacts.go:57-73`) already maps a
  `GetSession` NotFound to `404`, so the getter filter covers it.
- **Session lists** add `AND parent_session_id IS NULL`: `ChatRepository.ListSessions`
  (count and page queries, `repository/chat.go:116-156`) and
  `DeployedRepository.ListSessions` (`repository/deployed.go:88-112`).

**Child transcripts:**

| Method + path | Response |
|---------------|----------|
| `GET /agent_versions/{v}/chat/{s}/children/{child_id}` (BO) | HTML read-only transcript of the child, with nested child cards. `404` unless `s` is a root of version `v` and `child_id` descends from `s`. Child `artifact_ref` blocks render as filename labels, not links. |
| `GET /deployed/{agent_id}/sessions/{root_id}/children/{child_id}?user_id=X` | JSON with the same shape as `GET /deployed/{agent_id}/sessions/{id}`, plus `parent_session_id`, `parent_tool_call_id`, `depth` and `agent_version_id`. Returns `404` unless both hold: (1) the deployed preamble passes for `root_id` as a root (`requireDeployed` + `GetSession(root_id, user_id)` + agent match, `deployed.go:63-78`); (2) `child_id` descends from `root_id`. Child `artifact_ref` blocks are returned as data, but no artifact route serves them. |

A child read walks down from `root_id` with a recursive CTE over `parent_session_id`, so it
works at any depth. The deployed children route nests three levels below `/deployed`,
which is deeper than `api.md`'s maximum of 2. The repo already has deep routes like this,
and the deviation is listed below.

### REST — eval and training gate

**Defence in depth.**
- **Primary rule.** Config writes are refused while eval or training work is active
  (*Repository invariants*). Eval create and training create take `FOR SHARE` on the
  version's `agent_versions` row in the same transaction as their insert, and re-read
  `agent_tool_enabled` under it. Config writes hold `FOR UPDATE` on that row, so the two
  serialise. No eval or training row can therefore be `PENDING` or `IN_PROGRESS` while its
  version has the agent tool enabled, unless the row was inserted outside the API.
- **Second check.** Every route that starts or creates work also checks the version's
  **current** `agent_tool_enabled`.
- **Drainer-aware.** The checks fail rows forward, so a drainer never retries them. Shipped
  drainer order: `scripts/run_eval.sh:14-16` and `scripts/train_from_session.sh:59-64` (repo root) fetch
  `export.yaml` **before** calling `/start`.

| Route | Check location | Response when `agent_tool_enabled` |
|-------|----------------|-------------------------------------|
| `POST /agent_versions/{version_id}/evals` (`handler/eval.go:108-180`) | `EvalRepository.Create` (`repository/eval.go:74-89`), under `FOR SHARE` on the version row, before the insert | `409` via `handler.Error` (JSON `ErrorResponse`) for both JSON and form requests. This matches how sentinel errors already leave this handler (`eval.go:156-167`). No row is created. |
| `GET /evals/{id}/export.yaml` (`eval.go:191-206`) | handler, before `eval.Build` | In one transaction: a `PENDING` eval moves to `FAILED` with `error_message` = the sentinel text, and any `PENDING` training session whose `source_eval_id` is this eval moves to `FAILED` with `error_message` = the sentinel text. Then `409`. |
| `POST /evals/{id}/start` (`eval.go:182`) | repository, inside the `PENDING → IN_PROGRESS` transaction | The eval moves to `FAILED` with `error_message` = the sentinel text; `409`. |
| `POST /training-sessions` (`handler/training_session.go:89-131`) | training-session repository create, under `FOR SHARE` on the source eval's version row, after the existing handler gate checks | `409` via `handler.Error`. No row is created. |
| `POST /training-sessions/{id}/start` | repository, inside the `PENDING → IN_PROGRESS` transaction; checks the source eval's version | The session moves to `FAILED` with `error_message` = the sentinel text; `409`. |

**BO buttons.** When the version has `agent_tool_enabled`:
- the Evaluate button on the version page renders disabled;
- the Train button on the eval detail page renders disabled.

Both show the tooltip "Evaluating multi-agent versions is not supported yet".

### LLM tool contract — `agent`

The orchestrator inserts the `agent` tool definition immediately after `Skill` when the
version has `agent_tool_enabled = true` and at least one binding, and no endpoint tool is
named `agent` (see *Scope*).

```json
{
  "name": "agent",
  "description": "Delegate a self-contained task to a specialised sub-agent. The sub-agent starts with NO access to this conversation or its files — put everything it needs in `prompt`. It returns its final answer as text.\n\nAvailable sub-agents:\n- <name>: <note>\n- …",
  "input_schema": {
    "type": "object",
    "required": ["subagent", "description", "prompt"],
    "additionalProperties": false,
    "properties": {
      "subagent":    {"type": "string", "enum": ["<name>", "…"]},
      "description": {"type": "string", "description": "3-10 word summary of the task, shown to the user as progress"},
      "prompt":      {"type": "string", "description": "Full task instructions for the sub-agent"}
    }
  }
}
```

**Success result.** `tool_result.content` is the plain text string formed by concatenating
the `text` blocks of the child's last assistant message, with `is_error: false`. This matches
the arch contract `agent(…) → text`.
- No ids or JSON envelope are added to LLM-visible content.
- No `artifact_ref` block is ever added to the root's tool-role message for an `agent` call.
- The child session is located by `(parent_session_id, parent_tool_call_id)`, which is
  indexed. The BO card and the transcript links resolve the child this way, so no LLM-visible
  id is needed.

**Error result.** `is_error: true` with `content = "<code>: <details>"`. The codes are:
- `max_depth_exceeded`: no child is created.
- `unknown_subagent`: `subagent` names no binding of this version. No child is created.
- `invalid_input`: the `tool_use` input cannot be parsed, `subagent` / `description` /
  `prompt` is missing or empty, or there is an unknown property. No child is created.
- `session_locked` has two causes:
  - (a) `CreateChildSession` finds the BO root already locked, and no child is created;
  - (b) the child turn fails with `ErrSessionLocked` (`orchestrator.go:241-245`), because
    close-draft committed between `CreateChildSession` and the child's `AppendUserTurn`.
    The empty, locked child row remains; see *Risks*.
- `subagent_max_tool_rounds`: the child hit `OPENBBC_MAX_TOOL_ROUNDS`. `details` carries its
  partial text.
- `subagent_failed`: any other error from `CreateChildSession` or the child turn (LLM, tool
  handler or store).
- `cancelled`: the parent context was cancelled.

### Go internal contracts (`open-bbcd/internal/chat`)

```go
type TurnOpts struct {
    Depth            int    // 0 for root turns
    ParentSessionID  string // empty for root turns
    ParentToolCallID string // raw tool_use id from the parent's persisted message (never prefixed)
    RootSessionID    string // id of the tree's root session; equals the session id for root turns
}

// subAgentRunner is internal to package chat; an interface only so tests can fake it.
type subAgentRunner interface {
    Run(ctx context.Context, req subAgentRequest) (subAgentResult, error)
}
type subAgentRequest struct {
    ParentSessionID  string
    ParentToolCallID string // raw tool_use id, as persisted
    RootSessionID    string
    ParentDepth      int
    Binding          SubAgentBinding // Name, TargetVersionID, Note
    Description      string
    Prompt           string
    ParentSink       transport.Sink
}
type subAgentResult struct {
    ChildSessionID string
    Text           string
    StopReason     string // provider stop reason (e.g. "end_turn", "max_tokens") or "max_tool_rounds"
}
```

**`Turn`.** `Orchestrator.Turn(ctx, agentID, sessionID, userInput, sink)`
(`orchestrator.go:143-148`) gains a trailing `opts TurnOpts`. Existing callers (both turn
handlers) pass `TurnOpts{}`, and `Turn` normalises the value: when `opts.RootSessionID` is
empty it sets it to `sessionID`. A zero-value `TurnOpts` is therefore always a correct root
turn.

The interface the handlers actually call, `handler.TurnRunner` (`internal/handler/chat.go:79`,
used by `deployed.go:37`), takes the same new signature
`(…, opts chat.TurnOpts) (string, error)`. So do its test stub `stubTurnRunner`
(`handler/chat_test.go:87`) and every existing orchestrator test that calls `o.Turn(...)`.

`Turn` changes in five ways. The change to tool-loop dispatch is described separately,
under *Dispatch*:
- It no longer calls `sink.Close()`; today it does at `orchestrator.go:486`. The BO and
  deployed turn handlers `defer sink.Close()` right after creating the sink, which makes the
  comment at `handler/chat.go:691-692` true.
- It defers `Close()` on the handler returned by `Build` (`orchestrator.go:199`) whenever
  that handler implements `io.Closer`.
- It loads the version's bindings (new `AgentReader.ListSubAgentBindings`) and inserts the
  `agent` tool definition immediately after `Skill` in `toolHandler.Tools(...)`.
- It carries `opts.Depth` and `opts.RootSessionID` into the runner, so nested calls know
  their depth and their tree's root. `opts.ParentSessionID` and `opts.ParentToolCallID` are
  used only as structured log attributes (`parent_session_id`, `parent_tool_call_id`) on
  the child turn's log lines.
- It returns its stop reason: the signature becomes `Turn(...) (stopReason string, err
  error)`. The value is the provider's last non-`tool_use` stop reason (`end_turn`,
  `max_tokens`, `stop_sequence`, `refusal`, …; `orchestrator.go:381-386`), or
  `max_tool_rounds`, or empty on error. The runner maps only `max_tool_rounds` to
  `subagent_max_tool_rounds`. Every other non-empty reason counts as success, and the child's
  text blocks are returned as written. Today the stop reason exists only on
  `TurnEndEvent` (`orchestrator.go:481-485`), which `childSink` drops. Handlers ignore the
  value.

**Dispatch.** In the tool loop (`orchestrator.go:397-442`):
- `agent` calls in the current step are **removed from the `Composite.Call` path**.
- Each one runs in its own goroutine through `subAgentRunner.Run`, limited by a semaphore
  of size `AGENT_TOOL_MAX_PARALLEL`.
- Other tools keep their sequential dispatch through `Composite.Call` in the existing loop.
- Within one step, the sequential non-agent tools run first, in `tool_use` order. Then the
  step's `agent` calls are started together under the semaphore.
- Each call's `ToolResultEvent` is sent when that call finishes.
- Results are assembled in the original `tool_use` order into the single tool-role message,
  which `AppendToolMessage` commits (`orchestrator.go:459-467`).

The parent's `Composite` is never called concurrently, since only `agent` calls run in
parallel and they never touch it. The parent's messages are written only by the parent
loop, after all children of the step return.

**`subAgentRunner.Run`.**
1. Checks the depth cap.
2. Calls the new `ChatStore.CreateChildSession(ctx, rootID, parentID, parentToolCallID,
   targetVersionID)`, passing `RootSessionID` from `TurnOpts`. The root therefore never has
   to be found by walking up the tree. Both the BO and the deployed store implement it.
   `parent_tool_call_id` stores the **raw** `tool_use` id. The history card's lookup
   against the parent's persisted `tool_use` block depends on this, so the prefixed wire id
   is never stored.
3. Calls `Turn(target_version_id, [TextBlock(prompt)], childSink, TurnOpts{Depth:
   parent+1, …})`. The child's `AppendUserTurn` claims nothing, because children never
   have pending uploads.
4. Reads the result text from the child's last persisted assistant message. The stop reason
   comes from `Turn`'s return value.

One runner is built per orchestrator instance, so BO children land in `chat_sessions` and
deployed children in `deployed_sessions`.

### AG-UI stream (deployed) and BO stream

**New transport events** in `internal/transport/transport.go`:
- `StepStartedEvent{StepName, ToolCallID, ChildSessionID, Description}`
- `StepFinishedEvent{StepName, ToolCallID, ChildSessionID, IsError}`

The existing `ToolCallStart` / `ToolCallArgs` / `ToolCallEnd` / `ToolResult` events gain an
optional `ChildSessionID`.

**`childSink`** wraps the parent sink. Its rules, for the child it was created for:
1. It sends `StepStarted` before the child turn starts and `StepFinished` after it returns.
   `ToolCallID` is the **wire id** of the parent's `agent` call, and the runner computes it
   itself: `ParentSessionID + ":" + ParentToolCallID` when `ParentDepth > 0`, otherwise the
   raw `ParentToolCallID`. Nested cards use it to attach to their parent card. The AG-UI
   mapper never prefixes `ToolCallID` on `STEP_STARTED` / `STEP_FINISHED`. Only
   `ToolCall*` / `ToolResult` events are prefixed, from their own `ChildSessionID`.
2. Events the child emits itself (`ChildSessionID` empty):
   - tool-call and tool-result events are forwarded with `ChildSessionID` set to this child;
   - `SessionStart`, `TurnEnd`, `Text*`, `ArtifactRef` and `Error` are dropped. A child error
     surfaces as the `agent` tool error result instead.
3. Events that already carry a `ChildSessionID`, meaning they come from a grandchild or
   deeper, are forwarded **unchanged**, including `StepStarted` / `StepFinished`. The
   innermost child's id and prefix are never overwritten or prefixed twice.
4. `Close()` is a no-op.
5. `Send` is serialised with a mutex, so parallel children can share one parent sink. The
   AG-UI sink already has its own mutex (`agui.go:76-105`).

**AG-UI mapping** (`internal/transport/agui/agui.go:109-174`). It uses the SDK's
`NewStepStartedEvent` / `NewStepFinishedEvent` and the base event's `RawEvent` field.

| Transport event | AG-UI wire |
|-----------------|-----------|
| `StepStartedEvent` | `STEP_STARTED`, `stepName = "<binding name>:<ToolCallID>"`, `rawEvent = {childSessionId, parentToolCallId: <ToolCallID>, description}` |
| `StepFinishedEvent` | `STEP_FINISHED`, same `stepName`, `rawEvent = {childSessionId, isError}` |
| `ToolCall*` / `ToolResult` with `ChildSessionID` | `TOOL_CALL_START/ARGS/END/RESULT`, `toolCallId = "<ChildSessionID>:<tool_use id>"` (the same value as `messageId` for results), `rawEvent = {childSessionId}` |
| child `SessionStart`, `Text*`, `ArtifactRef`, `TurnEnd`, `Error` | not emitted |

The prefix is applied once, by the AG-UI mapper, from the event's own `ChildSessionID`.
Child session ids are globally unique, so ids never collide at any depth.

**Ordering guarantee** for each `agent` call on the root stream:
1. the root's `TOOL_CALL_START/ARGS/END` for `agent`;
2. `STEP_STARTED`;
3. zero or more tagged child events, including nested `STEP_*` pairs, interleaved across
   parallel children;
4. `STEP_FINISHED`;
5. the root's `TOOL_CALL_RESULT` for `agent`.

The stream carries exactly one `RUN_STARTED` and one `RUN_FINISHED`.

**Client compatibility.**
- **Versions without the agent tool.** Their streams are byte-for-byte unchanged.
- **Clients that ignore `STEP_*` and `rawEvent`.** With the agent tool enabled, such a
  client renders the worker's tool calls as extra top-level tool calls. Each has a unique
  prefixed `toolCallId`, so nothing collides. It still renders the root's text and tool
  calls correctly.
- **`events.md` envelope fields.** `schemaVersion` and `eventId` apply to the repo's own
  event contracts, not to the third-party AG-UI stream. Their absence there is not a
  deviation.

**BO.** BO uses the same AG-UI SSE transport by default (`OPENBBC_CHAT_TRANSPORT=agui`).
- **Live card.** `web/static/chat.js`, where unknown events currently fall to
  `console.debug`, handles `STEP_STARTED` / `STEP_FINISHED` and tagged tool-call events. It
  renders a collapsible card per step, showing the binding name, the description, a
  running / done / error state, and the child's tool calls. Cards nest under the card whose
  `toolCallId` equals `rawEvent.parentToolCallId`.
- **History view.** It renders the same card server-side from the persisted `agent`
  `tool_use` / `tool_result` pair, and links to the child transcript route. The child is
  resolved by `(parent_session_id, parent_tool_call_id)`. When that lookup finds nothing,
  the card renders without a link. That happens for errors before spawn
  (`max_depth_exceeded`, `unknown_subagent`, `session_locked` cause (a)), and for an unlocked BO child
  that cascaded away when its target version was deleted.
- **Interrupted cards.** If an `agent` `tool_use` has no matching `tool_result` in the immediately
  following message, its card renders as "interrupted". That happens after a client
  disconnect, or when a child hit `OPENBBC_MAX_TOOL_ROUNDS` with `agent` calls still
  unanswered in its final assistant message (`orchestrator.go:387-389`). The card still
  links to the child when the `(parent_session_id, parent_tool_call_id)` lookup finds one.
  Today `view.html:89` renders `tool_use` and `tool_result` independently, so this pairing
  is new behaviour.
- **JSONL transport.** The test-only transport (`transport/jsonl/jsonl.go`) gains
  `step_started` / `step_finished` frames and a `child_session_id` key, used by the
  orchestrator tests.

### Datasets (`feedback-datasets`)

The close-draft mutation `POST /datasets/{dataset_id}/close-draft` (`api.go:350-351`; the
`…/confirm` path is the GET modal) locks the session tree in **two statements**, in this
order, inside its existing transaction (`repository/dataset.go:198-203`):
1. the existing `UPDATE chat_sessions SET locked_at = now() …` on the member roots;
2. a **separate, later** `UPDATE` that sets `locked_at` on every descendant of those roots
   (recursive CTE over `parent_session_id`).

Statement 1 takes row locks on the roots, which conflict with `CreateChildSession`'s
`FOR SHARE`. A spawn that read `locked_at = NULL` therefore commits before statement 1 can
proceed, or the spawn waits and then sees the lock. Under READ COMMITTED, statement 2 takes
a fresh snapshot and sees any child that committed while statement 1 waited. No unlocked
child can remain under a locked root.

Child sessions can't join a dataset or receive feedback. Each enforcement point is listed
under the root-only rule: the `IsChildSession` check in the assign handlers, the
`AssignSessionToDraft` filter, and feedback scoped to the path session.

### Deviations from the arch-of-record (for `/arch-review` to sync)

1. **Step names.** `c4/integrations.md`, `ddd/contexts/deployed-runtime.md` and
   `c4/data-flows.md:190` say "step name = binding `name`". Here `stepName` is `"<name>:<tool_call_id>"`, because step names
   must be unique across parallel calls to the same binding.
2. **Which event carries the result.** The same files (`c4/data-flows.md:201,205`) say
   "the root's `TOOL_CALL_END` carries the result" and that the turn ends with `TURN_END`. On the shipped wire the result
   is `TOOL_CALL_RESULT` (`agui.go:136-145`), and the turn ends with `RUN_FINISHED`
   (`agui.go:162-163`, `TurnEndEvent` → `NewRunFinishedEvent`).
   The arch and glossary event lists also omit `TOOL_CALL_RESULT`, `RUN_FINISHED`, the
   prefixed child `toolCallId` and `rawEvent.parentToolCallId`. The arch names the child
   tag `child_session_id`. On the wire it is `rawEvent.childSessionId`, camelCase like the
   `ARTIFACT_REF` `CUSTOM` payload.
3. **Configurator write routes.** `ddd/contexts/agent-lifecycle.md § Published surface`
   (lines 100-101) lists `GET/POST /agent_versions/{id}/configure/agents` with
   per-binding `/notes` and `/delete`. This spec instead puts the writes under
   `/agent_versions/{id}/architecture/agents/{toggle, (add), notes, {name}/delete}`, the same
   way the MCP tab's writes are laid out, and notes are saved in bulk by one route.
4. **New deployed child route.** `GET /deployed/{agent_id}/sessions/{root_id}/children/{child_id}`
   is not yet listed in `ddd/contexts/deployed-runtime.md § Published surface`. The BO
   route is already listed in `ddd/contexts/feedback-datasets.md`. Both routes exceed
   `api.md`'s nesting depth of 2, the same as the existing deep routes.
5. **`assign-dataset` on a child.** `ddd/contexts/feedback-datasets.md` says it returns
   `409`. Under the root-only rule it returns `404`, like every other per-session route.
6. **Close-draft route.** `ddd/contexts/feedback-datasets.md` and `bizbok/value-streams.md:48`
   name the close-draft mutation `POST /datasets/{id}/close-draft/confirm`. The shipped mutation is
   `POST /datasets/{dataset_id}/close-draft`.
7. **Deployed child `agent_id`.** The arch doesn't say which `agent_id` a deployed child
   carries. This spec fixes it to the root's.
8. **`GET /evals/{id}/export.yaml` gains a side effect.** For a gated version it moves a
   `PENDING` eval, and the `PENDING` training sessions tied to it, to `FAILED` before
   returning `409`. That is a deliberate one-shot fail-forward for the drainers, which call
   export before `/start`. A retry is harmless: once the rows are `FAILED`, it only returns
   `409` again.
9. **Delete guards.** `ddd/contexts/agent-lifecycle.md:63-64` says "versions are never
   deleted", but a version delete route already exists. The arch also does not mention the
   new `ErrVersionReferenced` guards on version and agent delete. Both need syncing.
10. **Client compatibility.** `c4/integrations.md:24-25` says SDKs that ignore steps or the
    extra field "still render a correct root-level conversation". Such clients render the
    root's conversation correctly but also show worker tool calls as extra top-level tool
    calls (see *Client compatibility*), so the sentence should be qualified.
11. **Not a deviation: the temporary eval and training gate.**
   `ddd/contexts/evaluation.md` "Evals run the real topology" stays the target state.
   `/arch-review` should **not** rewrite it to describe the gate.

## Acceptance criteria

### Migration + config

- **Migration up.** `029_agent_tool.sql` applies cleanly on a database at `028`. Existing
  sessions read as roots, and existing versions have `agent_tool_enabled = false`.
- **Migration down.** `goose down` with children present removes every former child row.
  Afterwards, no former child appears in any `GET /deployed/{agent_id}/sessions?user_id=X`
  or BO session list.
- **Boot validation.**
  - `AGENT_TOOL_MAX_DEPTH=0`, or a non-integer value, fails boot with an error naming the
    var. The same applies to `AGENT_TOOL_MAX_PARALLEL`.
  - When unset, the values default to `3` and `4`.

### Agent-tool configuration

- **Editing on DRAFT.** On a `DRAFT` version with no active eval work, an admin can enable
  the tool, add a binding to a `READY` version of another agent, edit its note, and delete
  it. Each action swaps the fragment without a full reload.
- **Locked statuses.** On a `READY` or `DEPLOYED` version the tab renders read-only. A
  direct `POST` to any write route returns `409`, and the DB is unchanged.
- **Active eval work.** On a `DRAFT` version with a `PENDING` or `IN_PROGRESS` eval, or with
  a training session in either state, every write route returns `409` and the tab shows
  the banner. Once the eval is `DONE` or `FAILED`, writes succeed again.
- **Unrunnable targets.** Binding a target in `INITIALIZING`, `PENDING`, `DRAFT` or
  `TRAINING` returns `400`. The error fragment swaps into `#agents-error`, and the bindings
  table is unchanged.
- **Unbound names.** Deleting a name that is not bound returns `404`. A note for a name
  that is not bound is ignored, and the other notes still save.
- **Cycle check.** It is defence in depth, so it is tested at the repository level on
  raw-inserted fixtures that bypass the status rules. Each of the following returns
  `ErrTopologyCycle`:
  - a self-binding;
  - A→B when B→A exists;
  - A→B when B→C→A exists.
- **Bind racing Finalize.** A binding add on an `INITIALIZING` version runs concurrently
  with that version's Finalize (`INITIALIZING → PENDING`), tested with two connections. It
  ends in exactly one of two states:
  - the binding committed and Finalize ran after it;
  - the add returned `409` and no row exists.
- **Duplicates.** A duplicate `name` or a duplicate target on the same caller returns `409`.
- **Name collision.** Enabling the tool, or adding a binding, on a version whose agent has
  an endpoint tool named `agent` returns `409`.
- **Fork copy.** SavePrompts, LandPrompts and training Complete each produce a new version
  with the same `agent_tool_enabled` and identical binding rows (name, target, note).
- **Version delete guard.** Deleting a version returns `409 ErrVersionReferenced` and
  deletes nothing when the version is:
  - a binding target;
  - the version of a deployed child session;
  - pinned by a locked BO child session, including after the binding that spawned that
    child has been removed.
- **Agent delete guard.** Deleting an agent whose version is in one of those states because
  of **another** agent returns `409` and deletes nothing.
- **Self-binding agent delete.** Deleting an agent succeeds when it has no version currently
  `DEPLOYED` (the existing `ErrAgentInUse` gate, `repository/agent.go:135-143`, still
  applies), a version that binds an older version of the same agent, and deployed children
  from an earlier deployment pinned to its own versions. It
  removes all of the agent's versions, bindings, sessions and children in one statement.
- **Agent delete cascade.** Deleting a root agent whose deployed sessions have children
  removes the roots and all their children.

### Agent tool runtime (both surfaces)

- **Invalid input.** An `agent` call with a missing `prompt`, or with unparsable input,
  returns `invalid_input: …` and creates no session.
- **Tool exposure.**
  - A version with `agent_tool_enabled = false`, or with zero bindings, exposes no `agent`
    tool.
  - A version with bindings `researcher` and `writer` exposes one `agent` tool. Its
    `subagent` enum is `["researcher","writer"]`, its description contains both notes, and
    its schema has no `artifacts` property.
- **A root delegation.** `agent(subagent="researcher", prompt=P)` from the root:
  - creates exactly one child session with `depth = 1`, the correct parent links, and the
    researcher's pinned version;
  - a deployed child also carries the root's `agent_id` and `user_id`;
  - the child's first user message is `P`, and it sees none of the root's history;
  - the root's tool result is plain text equal to the child's final assistant text.
- **Root turns from both handlers.** A root turn started from the BO turn handler, and one
  started from the deployed turn handler, each spawn a depth-1 child. In both cases
  `CreateChildSession` receives the root's id as `rootID`.
- **Nesting and the depth cap.**
  - A child's own `agent` call creates a grandchild at `depth = 2`.
  - With `AGENT_TOOL_MAX_DEPTH=2`, a call from depth 2 returns
    `max_depth_exceeded: …`, creates no session, and the calling LLM continues its turn.
- **Parallel fan-out.** With 6 `agent` calls in one assistant message and
  `AGENT_TOOL_MAX_PARALLEL=4`:
  - no more than 4 children ever run concurrently (instrumented fake runner);
  - all 6 complete;
  - the tool-role message lists results in the original `tool_use` order;
  - no `UNIQUE(session_id, seq)` violation occurs (race test under `-race`);
  - the parent's `Composite.Call` is never invoked for `agent`.
- **Unknown sub-agent.** An `agent` call whose `subagent` names no binding of the version
  returns `unknown_subagent: …`, creates no session, and the calling LLM continues its turn.
- **Child failure.** A child turn whose fake LLM returns an error yields `is_error: true`
  with `subagent_failed: …`. The child row exists, and the root turn continues.
- **Child hits the round cap.** A child that reaches `OPENBBC_MAX_TOOL_ROUNDS` yields
  `is_error: true` with `subagent_max_tool_rounds: …`, and the root turn continues.
- **Cancellation.** Cancelling the root request context:
  - cancels running children. Their `cancelled: …` results are visible only to the fake
    runner in tests and on the already-dead stream. The parent's tool message is not
    persisted after cancellation (see *Risks*);
  - leaves no goroutine running after the root turn (goleak in the orchestrator tests).
- **Tool handler cleanup.** Every turn, root or child, successful or failed, calls `Close()`
  exactly once on its tool handler; a fake MCP backend records the call.
- **Header inheritance.**
  - The child's MCP calls carry the root request's forwarded headers and routing envelope.
  - On BO they also carry session header overrides for matching backend ids.
  - A BO child row's `backend_header_overrides` equals the root's at spawn time.
- **Locked root (two-connection test).** Connection A begins `CreateChildSession` and reads
  the root's `locked_at = NULL` under `FOR SHARE`. Connection B runs close-draft on a
  dataset containing that root. Two orderings are possible:
  - A commits first. B's statement 2 then locks the new child.
  - B's statement 1 locks the root first. A then reads `locked_at` set and returns
    `session_locked: …`.

  Either way, afterwards no session under the root has `locked_at IS NULL`. A later `agent`
  call in the same in-flight turn returns `session_locked: …`.
- **Sink close.** A root turn that fails before streaming (agent not runnable) still closes
  its sink exactly once.

### Sub-agent artifact rules

- **Child tool-result artifacts.** When a child's MCP tool returns `ImageContent`:
  - the `tool_result` row is written to the child's `*_session_artifacts`;
  - the child's next LLM call renders it;
  - the root's stream carries the child's tagged `TOOL_CALL_*` events but no
    `ARTIFACT_REF`;
  - the root's `agent` tool-result message contains no `artifact_ref`.
- **Echoed refs are not trusted.** The root writes a `{store_id, uri}` from one of its own
  artifact rows into the `agent` prompt, and the child echoes it in a `tool_use` input. The
  ref reaches the tool as opaque JSON, no session-artifact row is created on the child, and
  the child's messages contain no `artifact_ref` for it.
- **Artifact routes refuse child ids.** On both surfaces, upload, pending list, pending
  remove and retrieval addressed at a child session id return `404`.
- **Child transcript rendering.** The BO child transcript renders a child `artifact_ref` as
  a filename label with no link.

### Streaming

- **Depth-1 event order.** One `agent` call whose child calls one MCP tool produces this
  deployed SSE sequence:
  1. root `TOOL_CALL_START/ARGS/END` for `agent`;
  2. `STEP_STARTED`;
  3. child `TOOL_CALL_START/ARGS/END/RESULT`, with a prefixed `toolCallId` and
     `rawEvent.childSessionId`;
  4. `STEP_FINISHED`;
  5. root `TOOL_CALL_RESULT` for `agent`;
  6. root `TEXT_MESSAGE_*`, then `RUN_FINISHED`.

  The stream has exactly one `RUN_STARTED` and one `RUN_FINISHED`, and no child
  `TEXT_MESSAGE_*`.
- **Depth-2 event order.** A root calls worker W, and W calls worker X, which calls one MCP
  tool. The stream contains:
  - W's `STEP_STARTED`, with `parentToolCallId` = the root's `agent` id;
  - X's `STEP_STARTED`, with `parentToolCallId` = `"<W child id>:<W's agent tool_use id>"`;
  - X's tool events, prefixed exactly once with X's child id;
  - X's `STEP_FINISHED`, then W's `STEP_FINISHED`.

  No `toolCallId` is prefixed twice.
- **Multiple calls.** A root with two sequential `agent` calls streams both. Two parallel
  calls to the same binding produce two distinct `stepName`s.
- **Long turns.** A deployed turn and a BO turn that each run past 30s (fake LLM with a
  delay) stream to completion. A non-streaming route still times out at the server
  `WriteTimeout`.

### BO chat surface

- **Session list.** The list for a version shows only roots, including when that version is
  itself a binding target of another version.
- **History card.** After a turn with an `agent` call, the history view renders a card with
  the binding name and description. Its link opens the read-only child transcript, which
  works at depth 2.
- **Interrupted card.** For a persisted `agent` `tool_use` with no matching `tool_result`,
  the history view renders the card as "interrupted". If a child row exists for it, the card
  links to the child; if not, it has no link.
- **Child ids return 404.** Every BO per-session route listed under the root-only rule
  returns `404` for a child id. That includes `ChatView`, title, headers GET/POST and
  assign-dataset, each with no write. `POST …/turn` with a child id creates nothing and runs
  nothing.
- **Close-draft cascade.** `POST /datasets/{dataset_id}/close-draft` on a draft containing a
  root with two children and one grandchild sets `locked_at` on all four sessions.
- **Manual QA, recorded in the code PR.** For a parallel delegation to two workers, the live
  `chat.js` cards show running then done, and a depth-2 card nests under its parent card.

- **Feedback is scoped to the path session.** Feedback GET/POST/DELETE for a child's
  `message_id` sent under its **root's** path returns `404`, and no feedback row is written.
  The same applies to a message of another root sent under this root's path.
- **Children can't join datasets.** `AssignSessionToDraft` called directly with a child
  session id returns `ErrNotFound`, and no `dataset_version_sessions` row is written.

### Deployed surface

- **Session list.** `GET /deployed/{agent_id}/sessions?user_id=X` never returns children.
- **Child read.** `GET …/sessions/{root_id}/children/{child_id}?user_id=X` returns the child.
  It returns `404` when the `user_id` is wrong, when `child_id` is not a descendant, or when
  `root_id` is itself a child.
- **Root-only routes.** `GET`, `PATCH title`, `DELETE`, `turn` and every artifact route
  addressed at a child id return `404` and change nothing; the child still exists
  afterwards. A `DELETE` naming another agent's `agent_id` with a valid root id also
  returns `404`.
- **Root delete cascade.** `DELETE …/sessions/{root_id}` removes the root, every descendant,
  their messages and their session-artifact rows.

### Eval and training gate

- **Create racing enable (two-connection test).** An eval create and an enable toggle on
  the same DRAFT version run concurrently. The outcome is exactly one of these:
  - the eval exists and the toggle returned `409`;
  - the toggle committed and the create returned `409`.
- **Creation is refused.** For an `agent_tool_enabled` version,
  `POST /agent_versions/{v}/evals` returns `409` with the sentinel text as JSON, for both
  JSON and form requests. No `evals` row is created.
- **Enabling is refused while work is active.** An eval or training session is
  `PENDING`/`IN_PROGRESS` on a DRAFT version with the tool off. Enabling the tool returns
  `409`, and no score or training run is ever recorded against a version that had the tool
  enabled while the work ran.
- **Backstop, raw-inserted state.** A raw-inserted `PENDING` eval on an
  `agent_tool_enabled` version is put through the shipped drainer flow (`run_eval.sh`:
  export, then start). Export returns `409` and leaves the eval `FAILED` with the sentinel
  text. The drainer's next cycle no longer lists the eval in
  `GET /evals.json?status=PENDING`.
- **Backstop, training sessions.** Training create and training `/start` return `409`. A
  raw-inserted `PENDING` training session moves to `FAILED` on `/start` and on its eval's
  export.
- **Buttons.** The Evaluate and Train buttons render disabled with the tooltip.
- **No regressions.** For versions without the agent tool, eval and training behaviour is
  unchanged.

## Risks & assumptions

### Assumptions

- **The ag-ui Go SDK already has what we need.** *Verified.* The version pinned in `go.mod`
  (`sdks/community/go v0.0.0-20260609155419-861d0b22880e`) exports `NewStepStartedEvent` /
  `NewStepFinishedEvent` (`pkg/core/events/run_events.go:309,367`) and a `RawEvent` field on
  the base event (`events.go:142`). No bump is needed.
- **Status only moves forward.** Shipped transitions are: `DRAFT` only through a fork;
  `INITIALIZING → PENDING → READY`; `READY ↔ DEPLOYED`; and `READY → TRAINING → READY`
  (`agent_version.go:168-192,420-436`; `agent_detail.go:565`). No status ever returns to
  `INITIALIZING` or `DRAFT`. "Acyclic by construction" depends on this. A future transition
  back to `DRAFT` would make the cycle check load-bearing, and it is already in place.
- **AG-UI verifiers tolerate overlap.** Client-side AG-UI event verifiers accept
  overlapping `STEP_STARTED` / `STEP_FINISHED` pairs and interleaved `TOOL_CALL_*`
  sequences from parallel children. The BO is not affected: `web/static/chat.js` parses the
  SSE stream itself and has no AG-UI client or verifier. The risk is in deployers' frontends
  that use the official AG-UI JS client. *To verify in the code PR* against that client's
  verifier. If a verifier rejects overlap, `childSink`
  serialises child event output per step (children still run in parallel), and the spec is
  amended.
- **Several tool calls per assistant message.** The Anthropic provider accepts several
  `tool_use` blocks in one assistant message. The orchestrator already handles N per step,
  and `disable_parallel_tool_use` is not sent.
- **One orchestrator instance per surface is enough.** Children use the same `Model`,
  `MaxTokens` and `OPENBBC_MAX_TOOL_ROUNDS` as roots.
- **The runnable check is about content, not status.** `GetWithAgent`
  (`repository/agent_version.go:96-105`) filters by id only. `Turn`'s check
  (`orchestrator.go:173-175`) requires only a non-empty architecture and prompts. Pinned
  targets are `READY`/`DEPLOYED` at bind time, so they always pass. A target that is briefly
  `TRAINING` (`READY → TRAINING → READY`) still runs, which is intended: pinning is by
  version id, and training never mutates the pinned version's prompts.
- **The BO surface stays network-trusted.** Child transcript routes follow the same trust
  model as every BO route.
- **A target version's prompts never change after `READY`.** SavePrompts forks a new
  version instead of mutating a `READY` one. Pinning relies on this.

### Risks

- **Concurrent turns on one session are not serialised server-side (pre-existing).**
  - Each surface's lock covers only the user-turn insert transaction: BO
    `lockChatSessionForTurn`, deployed `FOR KEY SHARE`
    (`repository/session_artifacts.go:270-283,357-364`).
  - A second `POST …/turn` sent while a long multi-agent turn is still streaming would run
    alongside it, competing for `NextSeq` with only `UNIQUE(session_id, seq)` as a backstop.
  - Clients are expected to send one turn at a time; `chat.js` keeps input disabled while a
    stream is open. Multi-agent turns are longer, which widens the window.
  - Server-side single-flight per session is out of scope and should be a follow-up.
- **LLM cost and latency amplification.**
  - `AGENT_TOOL_MAX_PARALLEL` caps concurrency per node, not the number of `agent` calls per
    step; excess calls queue.
  - Each node runs up to `OPENBBC_MAX_TOOL_ROUNDS` steps. The total number of child turns per
    root turn is therefore bounded only by rounds × calls-per-step at each depth, down to
    `AGENT_TOOL_MAX_DEPTH`. Even 4 calls per step to depth 3 is 4 + 16 + 64 = 84 child turns.
  - Mitigations: admin-only bindings and gateway rate limits per root turn. The per-turn
    budget is the real cap, and it is an open question in `assumptions.md`.
- **Empty locked children.** A close-draft landing between `CreateChildSession` and the
  child's first `AppendUserTurn` leaves a locked child with no messages. The caller gets
  `session_locked`. The row is harmless: it is read-only, it is excluded from lists, and the
  history card links to an empty transcript.
- **A GET with a side effect.** For gated versions, `GET /evals/{id}/export.yaml` fails
  `PENDING` rows forward (Deviations 8). It is idempotent in effect, and it only fires for
  rows the primary rule says should not exist.
- **Silent stretches on the stream.** A child spending a long time in a single LLM call
  emits no events until its next tool call or its end. Intermediaries with SSE idle
  timeouts (gateway, ingress) may cut such a stream. No keep-alive frames are added in v1;
  deployers set idle timeouts above their longest expected LLM call.
- **`Composite.Tools()` lists MCP tools with `context.Background()`.** A child's tool listing
  therefore ignores cancellation and header context. This is accepted because roots behave
  the same today, and cancellation still stops the child's LLM loop.
- **Parallel children interleave events on one SSE stream.** `childSink` serialises `Send`.
  Clients tell children apart by `rawEvent.childSessionId` and the `toolCallId` prefix.
- **Confused deputy.** A sub-agent uses its own wiring, so a caller reaches backends it is
  not wired to. This is by design (`nfrs.md § Security`). Bindings are admin-only, behind the
  BO network restriction.
- **A missed root-only filter leaks worker transcripts.** Deployed children carry the root's
  `agent_id`, so a missed filter exposes only the user's own worker transcripts within the
  same agent, never another agent's. The acceptance criteria test every list, getter and
  write query explicitly.
- **Clearing the SSE write deadline removes a slow-client backstop on stream routes.** The
  request context is still cancelled when the client disconnects, and the round and depth
  caps bound the work.
- **A dangling `tool_use` after a client disconnect (pre-existing).** When the root context
  is cancelled, the parent's `NextSeq` / `AppendToolMessage` (`orchestrator.go:455-467`) fail
  on the dead context. The persisted assistant `tool_use` is then left without a
  `tool_result`. Normal tools behave the same today; multi-agent turns only widen the window.
  The history view renders such an `agent` card as interrupted. Persisting with
  `context.WithoutCancel` is left to a follow-up.
- **Test fakes must use unique `tool_use` ids.** The UNIQUE parent-link index rejects a
  second child for the same `(parent_session_id, parent_tool_call_id)`. Orchestrator test
  fakes must therefore generate a unique `tool_use` id per call, not a fixed `"tu_1"`
  reused across turns of one session.
- **Integration-test TRUNCATE lists are hand-maintained.** They live in
  `repository/integration_helper_test.go:41-46,74-80`, `handler/testhelpers_test.go:40`,
  `handler/backends_test.go:38` and `handler/configurator_test.go:681`.
  `agent_version_subagent` must be added to them, or state leaks between tests.
- **Multi-agent versions can't be evaluated or trained until `multiagent-eval` ships.**
  Admins can still evaluate each worker version on its own, because workers are ordinary
  versions.

---

## Cross-references

- **Arch-of-record (merged):** `docs/architecture/logs/2026-09-30-multiagent-tools/README.md`
  (PR #9), narrowed by `docs/architecture/logs/2026-10-01-sync-deployed-runtime-artifacts/README.md`
  (PR #12, SCOPE_MOVE H).
- **Modularity node:** `docs/architecture/current/modularity/open-bbcd/tool-runtime/README.md`
  (primary). Siblings: `agent-lifecycle`, `feedback-datasets`, `deployed-runtime`,
  `evaluation`, `training`, `artifacts`.
- **Arch invariants referenced:**
  - `ddd/contexts/agent-lifecycle.md § Invariants`: pinned targets, DAG, version-scoped
    config, verbatim inheritance.
  - `ddd/contexts/feedback-datasets.md § Invariants`: root-only datasets, lock cascade,
    child artifact scope.
  - `ddd/contexts/deployed-runtime.md § Invariants`: hidden children, `STEP_*` progress,
    children not addressable by artifact routes.
  - `constraints.md § Hard technical limits`: caps, DAG, target status.
  - `nfrs.md § Security`: the sub-agent trust model.
  - `c4/integrations.md § Contracts`: AG-UI step events, the agent tool.
- **Related spec:** `docs/superpowers/specs/2026-09-30-deployed-runtime-artifacts-design.md`
  — its § *Sub-agents* and its *Sub-agents* acceptance criteria are implemented here.
- **Follow-on spec anticipated:** `multiagent-eval`. It covers the transitive
  `eval-input.yaml` export, the in-process topology in aikdm `evaluate` / `train-agent`
  (using the same text-only `agent` contract), and lifting the gate.
