# eval-mcp-tools — MCP backend tools in `aikdm evaluate` and `train-agent`

**Date:** 2026-11-21
**Status:** Draft — pending `/spec-review`
**Modularity attachment:** `docs/architecture/current/modularity/open-bbcd/evaluation/` (L2).
Also affects [`aikdm/eval-scoring`](../../architecture/current/modularity/aikdm/eval-scoring/README.md)
(the new MCP tool caller and mock wiring) and reuses the MCP client from
[`open-bbcd/tool-runtime`](../../architecture/current/modularity/open-bbcd/tool-runtime/README.md)
at export time, without changing it. [`aikdm/training-loop`](../../architecture/current/modularity/aikdm/training-loop/README.md)
gets the feature without code changes, because it reuses the eval export and `run_eval`.

**Origin:** handoff from the ai-sales-copilot project (an OpenBBC consumer), Item 1
("MCP tools never reach evals or training"). Item 2 of the same handoff (the `Skill` tool resolving
from the version's `skill_prompts`) is a separate spec.

**Grounded against:** OpenBBC `main` @ `a56dfd7`. Paths are relative to `open-bbcd/` or
`aikdm/aikdm/` as stated.

## Business value / Why

Evals are how a PL decides whether a version is better than its parent, and they are the reward
signal for `train-agent`. Today an agent whose capabilities come from an attached MCP backend is
evaluated and trained **without those tools**:

- The consumer's `sales-copilot` agent has one MCP backend (`kb`, 18 tools: search, project
  reads, change proposals, RFP extraction). In BO and deployed chats these tools work. In evals and
  training the model is offered no tools, so every tool-dependent scenario fails or the agent
  answers from nothing.
- Training therefore optimises prompts for an agent that does not exist in production: **what we
  train is not what runs.**
- The consumer works around it out of tree (an HTTP→MCP bridge, a script rewriting
  `eval-input.yaml`, and a signing proxy for its per-call identity). Every MCP-backed consumer would
  have to rebuild that.

Root causes in code:

1. **The export drops MCP backends.** `internal/handler/eval_store_adapter.go` `GetToolBackends`
   (`:105-135`) selects only `tb.kind = 'http_endpoint'` and never reads
   `agent_version_mcp_backend`.
2. **The export has no MCP tool list.** `internal/eval/export.go` `InputPayload` (`:22-30`) carries
   only the bundle's agent-defined `tools`. Nothing runs `tools/list` against attached backends.
3. **aikdm has no MCP client.** `eval/tool_real.py` only does HTTP POSTs; `pyproject.toml` has no
   `mcp` dependency. `eval/target.py` `build_tool_specs` (`:29-50`) declares only `bundle.tools`.
4. **The mock can't cover it.** `eval/tool_mock.py` would replay recorded `kb__*` calls by
   `(name, args)`, but the model is never offered those names.
5. **No per-backend identity for evals.** `evals.header_overrides` (migration 023) is a flat
   `{header: value}` map applied to HTTP endpoint tools only. Deployed turns get per-backend
   headers through the `X-Openbbcd-Mcp-Headers` envelope; evals have no equivalent, so an MCP
   server that authenticates each call can't be evaluated at all.

Success is judged by: an eval and a training run of a version with an MCP backend attached offer
the model the same MCP tools, with the same names and schemas, as a deployed turn; real-mode runs
call them live and the judge sees the real results; and the consumer can delete its bridge,
inject script and signing proxy.

## Change level

**C2.** It touches:
- **DB:** new column `evals.mcp_header_overrides` (migration).
- **API:** new optional field on `POST /agent_versions/{id}/evals`; new error response on
  `GET /evals/{id}/export.yaml`.
- **Cross-service contract:** `eval-input-v1` (open-bbcd → aikdm) gains the `mcp_backends` key.
- **Security-relevant data:** identity headers stored on the eval row and written to the export.
- **Dependency:** aikdm adds the official Python `mcp` SDK.

All changes are additive and backward-compatible (see Contracts).

## Scope

### In

1. **Per-backend MCP headers on evals (open-bbcd, evaluation).**
   - Migration `030`: `evals.mcp_header_overrides JSONB NOT NULL DEFAULT '{}'`, with a CHECK that it
     is a JSON object. Existing rows get `{}`.
   - Shape: `{"<backend name>": {"<Header>": "<value>"}}`. This is the deployed
     `X-Openbbcd-Mcp-Headers` envelope without `_all`, which has no meaning in evals: there is no
     live frontend request whose headers could be copied.
   - Keys are backend names (`tool_backends.name`), as in the envelope.
   - The existing flat `header_overrides` is unchanged and keeps applying only to HTTP endpoint
     tools.
2. **Create API and BO.**
   - `POST /agent_versions/{version_id}/evals` accepts `mcp_header_overrides` (JSON) or the form
     field `mcp_header_overrides_json`.
   - Validation → 400 when the value is not an object of string-valued objects, or when it names a
     backend that is not attached to the version via `agent_version_mcp_backend`.
   - The BO "new eval" modal (`web/templates/eval/new_modal.html`) gets a per-backend header editor,
     one block per attached MCP backend. It is shown in **both** mock and real mode, because the
     headers are also used for discovery at export time (In → 3).
   - The eval detail page shows which backends have overrides (header names only, never values).
3. **MCP tool discovery at export (open-bbcd, evaluation).**
   - `GET /evals/{id}/export.yaml` — for each backend attached to the evaluated version via
     `agent_version_mcp_backend` with `kind = 'mcp_client'`:
     - It builds the **same** Go MCP client the runtime uses (`internal/llm/tools`
       `mcpClientBackend`), so naming (`{backend}__{tool}`), the missing-`inputSchema` fallback
       (`{"type":"object","additionalProperties":true}`) and transport behaviour are shared code,
       not re-implemented.
     - Headers are applied in runtime precedence: the backend's `config.default_headers`, then
       `mcp_header_overrides[<backend name>]` (overrides win per header).
     - It runs `tools/list` and writes the result into the new `mcp_backends` key (Contracts → 2).
   - Discovery runs in **both** modes. Mocked tools still need names and schemas.
   - Discovery uses a bounded timeout per backend (default 15s, env
     `OPENBBC_EVAL_MCP_DISCOVERY_TIMEOUT`).
   - Versions with no MCP backend attached → no `mcp_backends` key, so the export is byte-identical
     to today's.
4. **Discovery failure fails loudly.**
   - If any backend cannot be connected or its `tools/list` fails, the export returns **502** with a
     JSON error body (Contracts → 3) naming the backend. There is no partial tool set.
   - When the eval is `PENDING`, the handler also fails it forward to `FAILED` with that message,
     following the existing multi-agent gate pattern (`FailIfMultiAgent` /
     `failEvalMultiAgentTx`).
   - Evals in other states (e.g. a `DONE` eval used as a training source) are not touched.
   - `scripts/train_from_session.sh` treats a failed export as a session failure: it calls its
     existing `fail_session` with the error body. Today `set -e` exits before that, leaving the
     session `PENDING`.
   - `scripts/run_eval.sh` needs no change, because the handler already moved the eval to FAILED.
5. **aikdm declares MCP tools to the model (eval-scoring).**
   - `target.build_tool_specs` appends every tool in `mcp_backends[*].tools` after the bundle's
     tools, using `name`, `description` and `input_schema` exactly as exported. There is no renaming
     or sanitizing.
   - Tool **order** is not a parity requirement.
6. **Real MCP calls (`mock_mcp_tools=false`).**
   - New `eval/tool_mcp.py` `MCPToolCaller`, using the official Python `mcp` SDK streamable-HTTP
     client.
   - A small router in `eval/orchestrator.py` sends names `<backend>__*` whose `<backend>` is a key
     of `mcp_backends` to `MCPToolCaller`. Every other name goes to the existing
     `RealHTTPToolCaller`, unchanged.
   - Connection:
     - One client per backend per simulated session, connected lazily on first call with the
       exported `headers`.
     - Reused across turns and closed when the session ends.
     - A connect failure is sticky for that session (runtime: sticky for the turn).
   - Call: strip the `<backend>__` prefix and run `tools/call` with the model's arguments.
   - Result: mirrors Go `flattenMCPContent` (`internal/llm/tools/mcp_client_backend.go:210-226`):
     - Success: `{"content": [<every raw content block>], "text": "<TextContent texts joined by \n>"}`.
       `text` is omitted when there are no text blocks.
     - `isError: true` from the server: the same object, marked as an error.
     - Transport, connect and argument errors: `{"error": "<message>"}`, marked as an error.
   - The model sees `json.dumps` of that object, as for every other tool today (`target.py:114`).
7. **Mock (`mock_mcp_tools=true`).**
   - MCP tools are added to `ToolMock`'s schema table, using the backend tool's `output_schema` (when
     the server declared one) as `response_shape`.
   - Existing behaviour then applies unchanged:
     - Recorded `<backend>__<tool>` calls in reference transcripts replay on exact `(name, args)`.
     - Otherwise a result is synthesized from `output_schema`, or the generic success stub when
       there is none.
8. **Training.** `train-agent` calls the same `run_eval` for the baseline and each candidate, so it
   gets In → 5–7 with no training code changes. The only training change is the script fix in
   In → 4.
9. **Parity golden test (Go → Python), same pattern as the client_context golden test.**
   - A Go test runs a fake in-process MCP server through the runtime `mcpClientBackend` and writes
     `aikdm/tests/fixtures/mcp_parity.json` with:
     - the tool definitions the runtime offers the model, and the tool definitions written by the
       export, which the test asserts are equal;
     - for each case (single text; multiple text blocks; text + image; `isError`; transport error),
       the raw `tools/call` response and the runtime's model-visible result.
   - A Python test feeds the same raw responses through `MCPToolCaller`'s flattening and asserts the
     results equal the Go ones **as parsed JSON**. Whitespace and key order are not compared.
10. **Integration tests.**
    - Go: an export with an attached fake MCP backend writes `mcp_backends` with merged headers. An
      unreachable backend gives 502 and a `PENDING` eval moves to `FAILED`. A version with no MCP
      backend exports byte-identically to the pre-change golden.
    - Python: against a real in-process MCP server, it checks:
      - headers arrive;
      - the prefix is stripped;
      - a live result reaches the transcript;
      - the sticky connect failure;
      - mock replay of a recorded `kb__*` call;
      - an eval input without `mcp_backends` behaves exactly as before.
11. **Docs.**
    - The aikdm README gets a section on evaluating MCP-backed agents.
    - The BO help text explains the per-backend headers and recommends a dedicated long-lived test
      identity rather than a real user's credentials.

### Out

- **Multi-agent versions.** Their export still returns 409 (`FailIfMultiAgent`); MCP tools of
  subagents are not exported.
- **Artifacts.** At runtime, `normaliseToolResult` moves image and embedded-resource blocks into the
  artifact store and appends `artifact_ref` blocks. Evals keep those blocks raw in `content`, the
  same as the existing export, which passes `artifact_ref` through untouched.
- **SSE transport.** Only streamable HTTP is supported, as at runtime.
- **Per-tool allow-lists.** All tools of an attached backend are exported, matching runtime, which
  has no per-version tool filter.
- **Snapshotting tools on the eval row.** Discovery runs on every export. A training run therefore
  sees the backend's tool list at training time, not at the source eval's time (see Risks).
- **The routing envelope's `_all` forwarding** and BO chat-session overrides. Neither has a source
  in evals.
- **Fixing the HTTP caller's lookup key.** `RealHTTPToolCaller` looks backends up by tool name,
  while the export keys `tool_backends` by `endpoint_id`. That is a separate bug (see Risks).
- **Carrying the error flag to the model in aikdm.** LiteLLM tool messages have no slot for it
  (see Risks).

## Contracts

1. **DB: `evals.mcp_header_overrides`** — migration
   `open-bbcd/migrations/030_eval_mcp_header_overrides.sql`:
   ```sql
   ALTER TABLE evals
     ADD COLUMN mcp_header_overrides JSONB NOT NULL DEFAULT '{}'::jsonb,
     ADD CONSTRAINT evals_mcp_header_overrides_object
       CHECK (jsonb_typeof(mcp_header_overrides) = 'object');
   ```
   It is additive. Older code ignores the column, and the default covers existing and new rows
   written by the previous app version during rollout.

2. **`eval-input-v1` (open-bbcd → aikdm): new optional top-level key `mcp_backends`.**
   `schema_version` stays `eval-input-v1`. The key is additive and omitted when empty, and an older
   aikdm ignores unknown keys and behaves as today (`EvalInput` is `extra="ignore"`,
   `aikdm/eval/schemas.py:64`).
   ```yaml
   mcp_backends:                     # keyed by tool_backends.name
     kb:
       url: http://172.17.0.1:8012/mcp
       transport: streamable_http
       headers:                      # default_headers ∪ mcp_header_overrides["kb"]; overrides win
         X-Identity: "…"
       tools:
         - name: kb__search          # exactly as offered to the model at runtime
           description: "…"
           input_schema: {…}         # runtime fallback applied when the server omits it
           output_schema: {…}        # omitted when the server declares none
   ```
   - Present in both mock and real mode. aikdm only connects to `url` in real mode.
   - `header_overrides` and `tool_backends` (HTTP endpoint tools) are unchanged.

3. **`GET /evals/{id}/export.yaml`: new error response.** Status **502**,
   `Content-Type: application/json`:
   ```json
   {"error": "mcp_discovery_failed", "backend": "kb", "details": "<connect or tools/list error>"}
   ```
   Side effect: a `PENDING` eval moves to `FAILED` with
   `error_message = "mcp_discovery_failed: kb: <details>"`. Other statuses are unchanged.

4. **`POST /agent_versions/{version_id}/evals`: new optional field.**
   - `mcp_header_overrides` (JSON object), or the form field `mcp_header_overrides_json`.
   - Absent → `{}`.
   - 400 on a wrong shape or an unattached backend name.
   - Existing fields are unchanged.

5. **Tool result as seen by the model in aikdm for MCP tools:** as in Scope → In → 6, identical (as
   parsed JSON) to the runtime's `flattenMCPContent` output. It is pinned by
   `aikdm/tests/fixtures/mcp_parity.json`.

6. **Unchanged:** the `eval-result-v1` result contract, events (none), the deployed and BO chat tool
   paths, and the `X-Openbbcd-Mcp-Headers` envelope.

## Acceptance criteria

1. Exporting an eval of a version with the `kb` MCP backend attached yields `mcp_backends.kb` with
   all of the backend's tools. Their names, descriptions and input schemas equal those the runtime
   offers the model, as asserted by the Go golden test.
2. With `mock_mcp_tools=false`, `aikdm evaluate` offers the model the `kb__*` tools, calls the live
   MCP server with the exported headers, and the transcript and judge see the real results.
3. `aikdm train-agent` on a training session whose source eval has the backend attached does the
   same for the baseline and every candidate.
4. The Python parity test passes against `mcp_parity.json` for all five cases (single text,
   multiple text, text + image, `isError`, transport error).
5. With `mock_mcp_tools=true`, the `kb__*` tools are offered. A call matching a recorded reference
   call returns the recorded result; other calls return a result synthesized from `output_schema`
   or the generic stub. No connection to the MCP server is made during the run.
6. `mcp_header_overrides` set at eval creation reach the MCP server on both discovery and
   `tools/call`, and override `default_headers` per header.
7. Creating an eval with `mcp_header_overrides` naming an unattached backend → 400.
8. An unreachable backend at export → 502 with the documented body. A `PENDING` eval ends `FAILED`
   with `mcp_discovery_failed: <backend>: …`. A training session whose source-eval export fails ends
   `FAILED` with the same message rather than staying `PENDING`.
9. Versions without MCP backends: the export is byte-identical to a pre-change golden, and aikdm
   results are unchanged, as covered by the existing eval and training test suites passing.
10. The BO eval modal shows a header editor per attached MCP backend in both modes. The eval detail
    page lists header names but never values.

## Risks & assumptions

- **Secrets at rest and in transit.**
  - Identity header values are stored in plaintext on `evals.mcp_header_overrides` and written into
    `export.yaml`, which the drainer fetches over the cluster network.
  - This is the same exposure the existing `header_overrides` and `tool_backends.default_headers`
    already have; this spec does not make it worse in kind, only in use.
  - Mitigation: the docs and BO help text recommend a dedicated, scoped test identity, and the BO
    never echoes values back.
  - Encrypting header values at rest is a possible follow-up across all three stores, not in scope.
- **The error flag doesn't reach the model in aikdm.** At runtime an MCP error reaches the model as
  `tool_result` with `is_error: true`. aikdm's LiteLLM tool messages have no such flag, so the model
  sees the same `{"error": …}` payload but not the flag. Behaviour on errors may differ slightly
  between eval and runtime. This is already true for HTTP endpoint tools.
- **Runtime header caveat (verified, unchanged here).** At runtime, `Composite.Tools` runs
  `tools/list` with `context.Background()`, so discovery sends only `default_headers`, not envelope
  headers. The export instead sends overrides on discovery too. A server whose `tools/list` output
  depends on identity could therefore show evals a different tool list than deployed turns. This is
  assumed not to matter for current consumers. If it does, the runtime should be fixed (tool-runtime
  L2), not the export degraded.
- **Training sees today's tools.** Discovery runs on each export, so a training run uses the
  backend's tool list at training time, which may differ from the source eval's. This is accepted,
  because the tools that will run in production are the current ones.
- **Live backends make evals non-hermetic.** Real-mode evals depend on the MCP server's data and
  availability, and side-effecting tools (e.g. change proposals) really execute. This is unchanged
  in kind from real-mode HTTP tools. Consumers should point evals at a seeded or test backend, which
  the docs will say.
- **Python SDK parity.** We assume the Python `mcp` SDK serialises content blocks (`type`, `text`,
  `data`, `mimeType`, …) with the same field names as the Go SDK. The golden test catches drift. Key
  order and whitespace are allowed to differ, which we assume does not affect model behaviour
  materially.
- **Existing HTTP caller bug (observed, not fixed here).** `RealHTTPToolCaller` looks
  `tool_backends` up by tool name, while the export keys them by `endpoint_id`. HTTP endpoint tools
  in real mode only work when the two coincide. It is worth its own C1 fix; this spec does not
  depend on it.
- **Export latency.** Discovery adds up to one `tools/list` round-trip per backend to the export,
  bounded by the timeout. Acceptable for a drainer-driven, once-per-run fetch.
