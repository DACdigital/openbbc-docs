# bifrost — embedded Bifrost Go SDK as a second `llm.LLM` adapter in open-bbcd

**Date:** 2026-10-07
**Status:** Draft — pending `/spec-review`
**Modularity attachment:** `docs/architecture/current/modularity/open-bbcd/` (L1). No L2 owns
the LLM adapter layer: `internal/llm` sits under both orchestrators, the BO chat orchestrator
in `feedback-datasets` and the production one in `deployed-runtime`.

**Arch-of-record:**
- Locked decision "Multi-provider LLM access in `open-bbcd` goes through the embedded
  Bifrost Go SDK" in `assumptions.md`.
- `OPENBBC_LLM_ADAPTER`, `<provider>/<model>` and CGO-off embedding constraints in
  `constraints.md`.
- LLM providers row in `c4/integrations.md`.
- Glossary term "LLM adapter".
- Introduced by `docs/architecture/logs/2026-10-07-bifrost/` (PR #15).

**Grounded against:** OpenBBC `main` @ `96e4983`; Bifrost `github.com/maximhq/bifrost/core`
`v1.9.1`. File:line references are relative to `open-bbcd/` at that commit unless they start
with `deploy/`, which is relative to the OpenBBC repo root.

## Business value / Why

`open-bbcd` reaches models through one provider-agnostic interface, `llm.LLM`
(`internal/llm/llm.go`), but it has exactly one implementation:
`internal/llm/anthropic` (433 lines plus about 800 lines of tests). Today the only way a
deployer can use OpenAI, Gemini, Mistral, Groq, or a cheaper or self-chosen model is for us
to write another adapter. Each one means request mapping, stream normalisation, tool-call
delta tracking and multimodal rendering. That cost grows with every provider, and every
adapter is a place for the orchestrator's stop-reason and tool-use contract to drift.

Bifrost's Go SDK already normalises 20+ providers behind one OpenAI-shaped chat-completions
surface. Writing **one** adapter against it makes "support provider X" a config change for
every key-only provider Bifrost supports, and keeps provider churn out of our code.

Success is judged by:
- A deployer can run BO chat, the deployed runtime and multi-agent turns on a non-Anthropic
  model (e.g. `openai/gpt-4o`) by setting env vars only, with no code change.
- With `OPENBBC_LLM_ADAPTER` unset, behaviour is byte-for-byte unchanged (direct Anthropic).
- A tool-using turn behaves identically on both adapters from the orchestrator's point of
  view: same event sequence shape, same `tool_use` / `end_turn` / `max_tokens` stop reasons.

## Change level

**C2.**
- It adds a new external-integration contract: the env contract for adapter, provider,
  model, keys and base URLs.
- It takes a new core third-party dependency.
- It forces a Go toolchain upgrade from `go 1.26` to `go 1.27`.

A dependency or toolchain upgrade puts this at C1 or higher under the hard rules. The
contract surface (env, boot-failure semantics) makes it ambiguous between C1 and C2, and
the hard rule for that case is to pick the higher level.

## Scope

### In scope

1. **Adapter selection.** `OPENBBC_LLM_ADAPTER` ∈ {`anthropic` (default), `bifrost`},
   validated in `config.Load`. A `newLLM(cfg)` factory replaces the hard-coded
   `anthropic.New(cfg.Anthropic)` at `internal/handler/api.go:73`. One adapter instance
   serves both orchestrators and every sub-agent.
2. **Bifrost adapter.** New package `internal/llm/bifrost` implementing `llm.LLM` and
   `llm.MultimodalRenderer`. It is built on Bifrost's `ChatCompletionStreamRequest`
   (chat-completions API).
3. **Provider/model parsing.** On `bifrost`, `OPENBBC_DEFAULT_MODEL` = `<provider>/<model>`,
   split on the **first** `/` (so `openrouter/meta-llama/llama-3.1-70b` keeps its model id).
4. **v1 provider allow-list:** `anthropic, openai, gemini, mistral, groq, cohere,
   openrouter, deepseek, xai, cerebras`. These are key-only providers.
5. **Credentials.** An in-memory Bifrost `Account` built at boot with exactly one provider
   and one key, `<PROVIDER>_API_KEY`. The env name is the provider id upper-cased with `-`
   turned into `_`. Optional `<PROVIDER>_BASE_URL` for `openai, anthropic, cohere, mistral`
   only.
6. **Request/stream translation**, including finish-reason normalisation to the vocabulary
   the orchestrator branches on (`internal/chat/orchestrator.go:504`).
7. **Native image rendering** for `image/{png,jpeg,gif,webp}`. Every other MIME falls back
   to a text surrogate through the existing `TextSurrogate` path in `internal/chat/artifacts.go`.
8. **Shared fetch helper.** `fetchBytes` moves out of `internal/llm/anthropic` into
   `internal/llm` so both adapters use it. No behaviour change beyond a neutral `llm:`
   error prefix.
9. **Lifecycle.** Bifrost `Init` happens at boot, from `main`, through `handler.NewLLM`.
   `Shutdown` is called after the server's graceful shutdown (`cmd/open-bbcd/main.go:94-97`). Bifrost's logger is bridged to `slog`.
10. **Toolchain.** `go.mod` moves to `go 1.27`, and the Dockerfile builder moves from
    `golang:1.26` to `golang:1.27`. `.github/workflows/ci.yml:35` `actions/setup-go`
    `go-version` moves from `'1.22'` to `'1.27'`, so CI builds with the declared toolchain
    instead of relying on `GOTOOLCHAIN` auto-download.
11. **Docs.** The env table in the OpenBBC README / PRODUCTION docs. A Helm `values.yaml`
    comment describing how to pass adapter, model and keys through the existing
    `openbbcd.extraEnv` and `openbbcd.envFromSecret` (`deploy/helm/openbbc/values.yaml:36-43`).
    No new chart values.

### Out of scope

- **`aikdm` / `aikdm-runner`.** They keep Google ADK + LiteLLM. Runtime-vs-eval model
  parity is an open question in `assumptions.md`.
- **Cloud-credential providers.** Azure OpenAI, Bedrock and Vertex need endpoints,
  deployments, regions or cloud identity. They are refused at boot.
- **Keyless self-hosted providers** (ollama, vllm, sgl, OpenAI-compatible local endpoints).
  Refused at boot.
- **Bifrost fallbacks, load-balancing, multi-key weighting, semantic cache, governance and
  plugins.** None are configured.
- **Per-version or per-sub-agent model selection.** The model stays deployment-global.
- **Native PDF / file rendering** on the Bifrost adapter. PDFs become text surrogates.
- **Removing the direct Anthropic adapter.**
- **Bifrost's HTTP gateway / transport.** Only `core` is imported.
- **Any DB schema, REST, AG-UI or event change.**

## Contracts

### Env vars (read in `internal/config/config.go`)

| Var | Adapter | Required | Default | Validation (fails boot unless noted) |
|---|---|---|---|---|
| `OPENBBC_LLM_ADAPTER` | both | no | `anthropic` | ∈ {`anthropic`, `bifrost`} |
| `OPENBBC_DEFAULT_MODEL` | `anthropic` | no | `claude-sonnet-4-6` | unchanged (bare Anthropic id) |
| `OPENBBC_DEFAULT_MODEL` | `bifrost` | **yes** | — | `<provider>/<model>` with both parts non-empty; `<provider>` in the v1 allow-list. Unset (the `anthropic` default has no `/`), no `/`, empty part or non-allow-listed provider → boot error naming the variable and listing allowed providers |
| `OPENBBC_MAX_TOKENS` | both | no | `4096` | unchanged |
| `ANTHROPIC_API_KEY` | `anthropic` | no | — | unchanged: missing key fails lazily at the first LLM call |
| `<PROVIDER>_API_KEY` | `bifrost` | no | — | read only for the selected provider. Missing → no boot error; the first `Generate` yields `bifrost: <PROVIDER>_API_KEY not configured` (same lazy-fail policy as Anthropic) |
| `<PROVIDER>_BASE_URL` | `bifrost` | no | — | allowed only for `openai, anthropic, cohere, mistral` (the providers for which Bifrost honours `NetworkConfig.BaseURL`). Set for any other selected provider → boot error. Must parse as an absolute URL. The
scheme must be `https`, except that `http` is allowed only when the host is a loopback
address (`localhost`, `127.0.0.0/8`, `::1`) for local proxies and the `httptest` harness.
Plain `http` to any other host → boot error, because the provider key would travel
unencrypted |

Env names derived for the allow-list are `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`,
`GEMINI_API_KEY`, `MISTRAL_API_KEY`, `GROQ_API_KEY`, `COHERE_API_KEY`, `OPENROUTER_API_KEY`,
`DEEPSEEK_API_KEY`, `XAI_API_KEY` and `CEREBRAS_API_KEY`. `ANTHROPIC_API_KEY` and
`GEMINI_API_KEY` are deliberately the same names the direct adapter and `aikdm` already use.

`<PROVIDER>_*` variables for providers other than the selected one are ignored. They are
not read and not validated.

### Go internal contracts

**Config.** A new `config.LLMConfig { Adapter string; Provider string; Model string;
APIKey string; BaseURL string }` is populated by `Load`. `Provider` and `Model` are filled
only on `bifrost`. On `anthropic`, `Model` mirrors `AnthropicConfig.DefaultModel`.
`AnthropicConfig` is kept unchanged for the direct adapter.

**Factory and boot wiring** (`internal/handler`, `cmd/open-bbcd`):

```go
// NewLLM returns the boot-selected adapter and its shutdown func (no-op for anthropic).
func NewLLM(cfg *config.Config, logger *slog.Logger) (llm.LLM, func(), error)

// NewAPIWithLLM builds the API around an already-constructed adapter.
func NewAPIWithLLM(db *sql.DB, cfg *config.Config, logger *slog.Logger, client llm.LLM) http.Handler
```

- `cmd/open-bbcd/main.go` calls `handler.NewLLM` before building the `http.Server`. An
  error is returned from `run()` (`cmd/open-bbcd/main.go:46`) as `fmt.Errorf("init llm: %w", err)`. That is the same
  exit path as a failed DB connect or migration, so it gives a non-zero exit and a logged
  error with no `os.Exit` inside the handler.
- `main` passes the client to `handler.NewAPIWithLLM`. After `server.Shutdown(ctx)` returns,
  it calls the shutdown func, then returns the `Shutdown` error.
- `NewAPIWithLLM` delegates to the existing `newAPI(db, cfg, logger, llmClient)` test seam
  (`api.go:76-78`), which is unchanged.
- `handler.NewAPI(db, cfg, logger)` is kept with its current signature and behaviour (it
  constructs `anthropic.New(cfg.Anthropic)`). The existing integration tests that call it
  (`artifacts_integration_test.go:381`, `multiagent_integration_test.go:38`) are not
  touched. `main` no longer calls it.
- The orchestrators' `Model` field (`api.go:194`, `api.go:246`) is set from `cfg.LLM.Model`,
  which is the model part only on `bifrost`.

**Adapter** (`internal/llm/bifrost`):

```go
func New(ctx context.Context, cfg config.LLMConfig, logger *slog.Logger) (*LLM, error)
func (l *LLM) Name() string // "bifrost:<provider>"
func (l *LLM) Generate(ctx context.Context, req llm.Request) iter.Seq2[llm.Event, error]
func (l *LLM) Shutdown()
// llm.MultimodalRenderer
func (l *LLM) SupportsNative(ref llm.ArtifactRefBlock) bool
func (l *LLM) NativeRenderBudget() llm.RenderBudget
func (l *LLM) RenderArtifactAsBlock(ctx context.Context, ref llm.ArtifactRefBlock, fetch llm.ArtifactFetcher) (llm.Block, error)
```

`New` calls `bifrost.Init` with an `Account` whose methods behave as follows:
- `GetConfiguredProviders` returns `[selected]`.
- `GetKeysForProvider` returns one key (the env value, weight 1, no model restriction), or
  none when the key is empty.
- `GetConfigForProvider` returns default network/concurrency config plus `BaseURL` when set.

`New` fails only on `Init` error.

**Request mapping** (`llm.Request` → `schemas.BifrostChatRequest`):

| `llm` | Bifrost chat-completions |
|---|---|
| `Request.System` (non-empty) | first message, role `system`, text |
| `RoleUser` message | role `user`. `TextBlock` → text part; `InlineMediaBlock` (image only) → `image_url` part with `data:<mime>;base64,<b64>` |
| `RoleAssistant` message | role `assistant`. Concatenated `TextBlock`s → content; each `ToolUseBlock` → `tool_calls[]` `{id, type:"function", function:{name, arguments: string(Input)}}` (`"{}"` when `Input` is empty) |
| `RoleTool` message, each `ToolResultBlock` | role `tool`, `tool_call_id = ToolUseID`. Content = the JSON-string result unwrapped to plain text (same rule as `anthropic.convertMessage`), else the raw JSON. `IsError` prepends `"Error: "` |
| `RoleTool` message, non-result blocks (`InlineMediaBlock`, surrogate `TextBlock`) | collected into **one synthetic `user` message** placed directly after that message's `tool` messages. It starts with the text part `Attachments returned by tool call(s) <ids>:`, followed by the collected parts in order. It is omitted when there are none |
| `Request.Tools` | `tools[]` `{type:"function", function:{name, description, parameters: InputSchema}}` (permissive `{"type":"object"}` when `InputSchema` is empty) |
| `Request.MaxTokens` > 0 | `max_completion_tokens` |
| `Request.Temperature` > 0 | `temperature` |
| — | stream usage requested (`stream_options.include_usage = true`) |

Provider = `cfg.Provider`. Model = `Request.Model`, falling back to `cfg.Model` when empty.

**Stream translation** (`chan *schemas.BifrostStreamChunk` → `llm.Event`). Events are
emitted in this order:
1. A text content delta → `TextDeltaEvent{Delta}`.
2. A tool-call delta at `index i`, the first time `i` is seen → `ToolUseStartEvent{ID, Name}`.
   ID or Name may arrive on a later delta for the same index; start is emitted as soon as
   both are known.
3. A tool-call `arguments` fragment at `index i` → `ToolUseInputEvent{ID, JSONFragment}`.
   Fragments that arrive before start are buffered and flushed right after it.
4. A chunk with `finish_reason` → `ToolUseEndEvent{ID}` for every open tool call in index
   order, then `MessageStopEvent{StopReason}` per the table below. `ToolUseEndEvent`s are
   emitted on `length` too, so the orchestrator sees complete, if truncated, tool-call
   records to discard.
5. Usage (on any chunk, normally the last) → `UsageEvent{InputTokens: prompt_tokens,
   OutputTokens: completion_tokens}`.

Finish-reason normalisation:

Rules apply top to bottom; the first match wins:

| Bifrost `finish_reason` | `MessageStopEvent.StopReason` |
|---|---|
| `length` (whether or not a tool call is open) | `max_tokens` |
| `tool_calls` | `tool_use` |
| any other value, when ≥1 tool call was opened in this stream | `tool_use` |
| `stop` | `end_turn` |
| anything else | passed through verbatim |

`length` deliberately takes precedence over the tool-call override. A turn truncated in the
middle of a tool call must reach the orchestrator as `max_tokens`. That way the existing
"round did not stop for tool_use" branch (`internal/chat/orchestrator.go:461-467`) drops the
truncated `tool_use` and ends the turn, exactly as on the Anthropic adapter. It must not be
dispatched as a tool round with invalid input.

**A stream that ends without a `finish_reason` is an error.** If the channel closes before
any chunk carried a `finish_reason` and no `BifrostError` was seen, the adapter yields
`bifrost: <provider>: stream ended without finish_reason` and emits no `ToolUseEndEvent` or
`MessageStopEvent`. A cut-off stream therefore fails the turn, and is never mistaken for a
clean finish or a complete tool call.

**Errors.**
- A `*schemas.BifrostError` from the call, or carried on a chunk, yields
  `fmt.Errorf("bifrost: %s: %s (status %d)", provider, message, status)` (status omitted
  when absent), and translation stops.
- On ctx cancellation the adapter stops yielding, drains the channel in a goroutine until
  it closes, and yields `ctx.Err()`.
- No adapter-level retries.

**Multimodal.**
- `SupportsNative`: MIME ∈ `image/{png,jpeg,gif,webp}` and `ceil(size/3)*4 ≤ 5_000_000`.
- `NativeRenderBudget`: `{MaxBytes: 24<<20, MaxBlocks: 20}`, the same values as
  `internal/llm/anthropic` (`anthropic.go:333-335`).
- `RenderArtifactAsBlock`: fetch through the shared `llm.FetchBytes`, re-check the size on
  the actual bytes, return `llm.InlineMediaBlock`; otherwise `llm.ErrUnsupported`.

**Shared helper.** `internal/llm.FetchBytes(ctx, uri, fetch ArtifactFetcher) ([]byte,
error)` is moved from `anthropic.fetchBytes` (Get for bytes delivery, Sign + 60s-TTL follow
for signed-URL delivery). The anthropic adapter calls it. The only permitted change is the
error prefix: `"anthropic: signed URL fetch returned status …"` (`anthropic.go:427`)
becomes `"llm: signed URL fetch returned status …"`, so errors are not mislabelled when the
Bifrost adapter is in use. Behaviour is otherwise identical.

### Unchanged contracts

- DB schema, REST routes, the AG-UI wire, the BO stream, the `eval-input.yaml` export,
  `llm.LLM`, `llm.Event` and `llm.Request` are all untouched.
- The orchestrator still sees `tool_use` / `end_turn` / `max_tokens`.

### Deviations from the arch-of-record (for `/arch-review` to sync)

- `current/` says the Bifrost adapter reaches "any Bifrost-supported provider" (or
  equivalent) in:
  - `c4/integrations.md` (LLM providers row)
  - `glossary.md` ("LLM adapter")
  - `c4/containers.md:87`
  - `c4/context.md:53-54`
  - `c4/deployment.md:44-45`
  - `modularity/open-bbcd/README.md:21-23`

  This spec narrows v1 to a 10-provider key-only allow-list, and refuses cloud-credential
  and keyless providers at boot. `constraints.md`'s "unknown provider fails boot" should
  read "provider outside the v1 allow-list fails boot".
- `current/` does not record `<PROVIDER>_BASE_URL`. This spec adds it, restricted to the
  four providers where Bifrost honours it.
- The stated Go version becomes stale. Bifrost core v1.9.x requires Go 1.27, but
  `current/` still says:
  - "Go 1.22+" in `constraints.md`
  - "Go 1.22 plus" in `c4/containers.md:18`
  - "Go 1.22+" in `c4/containers.md:92` (open-bbcd Tech stack)
  - the `golang:1.26` builder in `c4/containers.md:95` (open-bbcd Tech stack)

  The toolchain is an implementation detail, but these stated values must be synced.

## Acceptance criteria

### Config + boot

- [ ] With no new env vars set, `open-bbcd` boots on the direct Anthropic adapter and the
      existing `internal/llm/anthropic` tests pass unchanged.
- [ ] `OPENBBC_LLM_ADAPTER=foo` fails boot with an error naming `OPENBBC_LLM_ADAPTER`.
- [ ] Each of the following fails boot with an error naming `OPENBBC_DEFAULT_MODEL`:
      - `OPENBBC_LLM_ADAPTER=bifrost` with `OPENBBC_DEFAULT_MODEL` unset
      - `OPENBBC_DEFAULT_MODEL=gpt-4o` (no `/`)
      - `OPENBBC_DEFAULT_MODEL=/gpt-4o` or `openai/` (empty part)
      - `OPENBBC_DEFAULT_MODEL=bedrock/…` (not on the allow-list)

      The error lists the allowed providers.
- [ ] `OPENBBC_DEFAULT_MODEL=openrouter/meta-llama/llama-3.1-70b` parses to provider
      `openrouter`, model `meta-llama/llama-3.1-70b`.
- [ ] `GROQ_BASE_URL` set with provider `groq` fails boot. `OPENAI_BASE_URL=not a url`
      fails boot. `OPENAI_BASE_URL=http://proxy.internal:8080` fails boot (plain `http` to
      a non-loopback host). `OPENAI_BASE_URL=https://proxy.example/v1` and
      `OPENAI_BASE_URL=http://127.0.0.1:9999` both boot with provider `openai`.
- [ ] On `bifrost` with the provider key unset, boot succeeds, and the first BO chat turn
      fails with `bifrost: <PROVIDER>_API_KEY not configured` surfaced exactly as today's
      missing-Anthropic-key error.
- [ ] `go.mod` declares `go 1.27`, and CI `setup-go` uses `'1.27'`. The Docker image builds with `CGO_ENABLED=0` on the
      `gcr.io/distroless/static-debian12:nonroot` runtime for amd64 and arm64.
- [ ] SIGTERM calls the Bifrost `Shutdown` after `server.Shutdown`, and the process exits
      within `ShutdownTimeout`.
- [ ] A `bifrost.Init` failure makes `open-bbcd serve` exit non-zero with `init llm: …`
      logged, before the listener opens.
- [ ] `handler.NewAPI` keeps its signature, and the existing handler integration tests
      compile and pass unchanged.

### Translation (unit tests on fabricated chunks / requests)

- [ ] A text-only stream yields `TextDeltaEvent`s, then `MessageStopEvent{end_turn}` and a
      `UsageEvent` carrying the final chunk's token counts.
- [ ] A single tool call yields `ToolUseStartEvent`, one or more `ToolUseInputEvent`s whose
      concatenation is the full JSON input, `ToolUseEndEvent`, then
      `MessageStopEvent{tool_use}`.
- [ ] Two parallel tool calls with interleaved index deltas are attributed to the correct
      IDs, and end events are emitted in index order.
- [ ] `finish_reason=stop` with a tool call opened yields `tool_use`. An unknown reason
      passes through verbatim.
- [ ] `finish_reason=length` with **no** tool call open yields `max_tokens`.
- [ ] `finish_reason=length` **with** a tool call open yields `ToolUseEndEvent`, then
      `MessageStopEvent{max_tokens}`, never `tool_use`. In a DB-backed orchestrator test on
      the `bifrost` adapter, that turn ends after one LLM round: the truncated `tool_use` is
      dropped, no tool is dispatched, and no follow-up LLM call is made. This matches the same
      scripted scenario on the anthropic adapter.
- [ ] A channel that closes with no `finish_reason` and no `BifrostError` yields exactly one
      error `bifrost: <provider>: stream ended without finish_reason`. It emits no
      `ToolUseEndEvent` and no `MessageStopEvent`, and the BO turn fails the same way as on
      any other provider error.
- [ ] A `BifrostError` on a chunk yields one error carrying provider, message and status,
      and no further events.
- [ ] Cancelling ctx mid-stream returns `ctx.Err()`, and the goroutine leak check
      (`goleak`, or a counting test) shows no leaked goroutines.
- [ ] Request mapping:
      - the system message is first, and absent when `System` is empty
      - an assistant `ToolUseBlock` round-trips into `tool_calls` (empty input → `"{}"`)
      - a JSON-string tool result is unwrapped, and `IsError` is prefixed
      - a tool message carrying an image yields the `tool` message(s) followed by one
        synthetic `user` message with the attachments text, then the `image_url` data URI
      - tools map to function tools with the schema passed through

### Multimodal

- [ ] `SupportsNative` is true for a 1 MB PNG and false for a 4 MB PNG (base64 > 5 MB),
      for `application/pdf` and for `text/csv`.
- [ ] A BO chat turn on `bifrost` with an attached PNG sends an `image_url` part. The same
      turn with a PDF sends the `[Attachment: …]` surrogate text.
- [ ] After the `FetchBytes` move, the anthropic artifact tests pass. The only assertion
      allowed to change is the signed-URL non-2xx error text (`anthropic:` → `llm:` prefix).

### End to end (CI, no live provider)

- [ ] An integration test boots a real `bifrost.Init` with provider `openai` and
      `OPENAI_BASE_URL` pointed at an `httptest` server replaying recorded chat-completions
      SSE. It drives one BO-orchestrator turn through one tool round (tool call →
      tool result → final text) and asserts the persisted `chat_messages` match the same
      scenario on the anthropic adapter (roles, block types, tool IDs).
- [ ] The same harness drives one deployed turn with a sub-agent call and asserts the AG-UI
      event sequence (`TOOL_CALL_*`, `STEP_STARTED/FINISHED`, `RUN_FINISHED`) has the same
      shape as on the anthropic adapter.

### Live smoke (manual / opt-in)

- [ ] `OPENBBC_LIVE_LLM_TEST=1 go test ./internal/llm/bifrost -run Live` runs two
      scenarios against each provider whose key is present, and is skipped otherwise:
      1. one tool-using turn (tool call → tool result → final text)
      2. one **image-returning tool** turn: the tool result carries a PNG, which produces the
         synthetic attachments `user` message, and the provider accepts the request and
         returns a final text answer that references the image.

      Both scenarios have been run at least once against `openai` and `anthropic` before
      merge.

### Docs

- [ ] The OpenBBC env table documents every variable in **Contracts › Env vars**, including
      the allow-list and the base-URL rule.
- [ ] `deploy/helm/openbbc/values.yaml` comments show setting `OPENBBC_LLM_ADAPTER` and
      `OPENBBC_DEFAULT_MODEL` through `openbbcd.extraEnv`, and provider keys through
      `openbbcd.envFromSecret`. No new chart values are added.

## Risks & assumptions

### Assumptions

- **Bifrost `core` v1.9.x normalises streaming tool calls consistently** across the
  allow-listed providers into OpenAI-shaped `tool_calls` deltas with stable `index`. This
  has been verified in source for the chat-completions schema, not yet live per provider.
  The live smoke test is the check.
- **`stream_options.include_usage` is honoured or harmlessly ignored** per provider. Where
  a provider sends no usage, no `UsageEvent` is emitted. The orchestrator already tolerates
  that, because usage is informational.
- **`image_url` data URIs are accepted** by every allow-listed provider that has vision
  models. A model without vision rejects the request, and that is a deployer
  model-choice error, surfaced as a provider error.
- **Building with CGO off** was verified locally (`CGO_ENABLED=0 go build ./...` on
  `bifrost/core` succeeds).
- **Deployer keys.** A deployer who selects a provider has a key for it, and mounts only
  that key into `open-bbcd`.

### Risks

- **Toolchain bump.** `go 1.27` is forced on the whole `open-bbcd` module. CI currently pins
  `setup-go` to `'1.22'` (`ci.yml:35`) and silently relies on `GOTOOLCHAIN` auto-download
  for `go 1.26.2`. That pin moves to `'1.27'` in this change. *Mitigation:* the bump lands as the first commit of
  the implementation PR, with CI green before adapter work starts.
- **Dependency weight.** Bifrost core pulls in the AWS, Azure and GCP SDKs, sonic,
  fasthttp, mcp-go and others, even when only key-only providers are used. The binary and
  image grow, and the supply-chain surface widens. *Mitigation:* record the image-size
  delta in the PR, and pin the exact `core` version in `go.mod`.
- **Stop-reason semantics drift.** Some providers report `stop` alongside tool calls, or
  `length` mid-tool-call. *Mitigation:* the override "tool call opened → `tool_use`" covers the
  first case. `length` takes precedence over that override and maps to `max_tokens`, so the
  orchestrator's existing mid-call truncation handling (`orchestrator.go:461-467`) drops the
  truncated call. Both cases are pinned by ACs.
- **Missing `finish_reason` treated as an error.** Some providers or proxies might close a
  healthy stream without a final `finish_reason`, and those turns would then fail instead of
  completing. This trade-off is deliberate: a false failure is visible and retryable, while
  a truncated stream mistaken for success silently persists partial text or dispatches a
  truncated tool call. If the live smoke test shows an allow-listed provider omitting
  `finish_reason` on healthy streams, that provider is removed from the allow-list rather
  than the rule being relaxed.
- **Synthetic attachments message.** Moving tool-returned images into a following `user`
  message is a translation the model sees, and Anthropic does not. Some providers reject
  two consecutive `user`-like turns or a `user` message between `tool` messages and the
  next assistant turn. *Mitigation:* it is emitted only when a tool result carries media,
  and covered by request-mapping tests and the live smoke test with an image-returning
  tool on `openai`.
- **Bifrost API churn.** `core` is pre-2.0, so schema field names may move between minor
  versions. *Mitigation:* all Bifrost types are confined to `internal/llm/bifrost`, and
  upgrades are deliberate version bumps with the translator tests as the guard.
- **Error-message fidelity.** Bifrost wraps provider errors, so BO/AG-UI error text will
  differ from the direct Anthropic adapter's. This is acceptable because no client
  contract parses error text.
- **Eval mismatch.** An agent tuned on a non-Anthropic Bifrost model is still evaluated by
  `aikdm` through LiteLLM with its own model config. This is tracked as an open question in
  `assumptions.md` and explicitly out of scope here.

## Cross-references

- Arch log: `docs/architecture/logs/2026-10-07-bifrost/README.md`
- `docs/architecture/current/assumptions.md` — Bifrost locked decision + open questions
- `docs/architecture/current/constraints.md` — `OPENBBC_LLM_ADAPTER`, `<provider>/<model>`,
  CGO-off
- `docs/architecture/current/c4/integrations.md` — LLM providers row
- Bifrost Go SDK: https://docs.getbifrost.ai/quickstart/go-sdk/setting-up
