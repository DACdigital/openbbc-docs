# client-context — client-supplied, LLM-visible `client_context` block on turns

**Date:** 2026-11-17
**Status:** Draft — pending `/spec-review`
**Modularity attachment:** primary
[`open-bbcd/deployed-runtime`](../../architecture/current/modularity/open-bbcd/deployed-runtime/README.md)
(L2), which owns the deployed turn route, `deployed_messages` and session read. Also touches:
- [`open-bbcd/feedback-datasets`](../../architecture/current/modularity/open-bbcd/feedback-datasets/README.md):
  the BO chat turn route, `chat_messages` and the BO chat view;
- [`open-bbcd/evaluation`](../../architecture/current/modularity/open-bbcd/evaluation/README.md):
  `GET /evals/{id}/export.yaml`;
- [`aikdm/eval-scoring`](../../architecture/current/modularity/aikdm/eval-scoring/README.md):
  replaying the block in eval.

The shared chat orchestrator (`internal/chat`) is under both open-bbcd L2s.

**Origin:** handoff from the ai-sales-copilot project (an OpenBBC consumer), Item 2, plus the
"Interaction" section that joins it to Item 1.

**Depends on:** [`eval-turn-replay`](2026-11-17-eval-turn-replay-design.md) (openbbc-docs PR #19).
Scope → In → 7 changes that spec's `UserStep` / `render_user_turn`. The open-bbcd part
(Scope → In → 1–6) does not depend on it and can ship first.

**Grounded against:** OpenBBC `main` @ `688a3c3`. Paths are relative to `open-bbcd/` unless they
start with `aikdm/`.

## Business value / Why

The consumer runs two OpenBBC agents behind one frontend. Both need deterministic facts from the
client that the model can see, but that are not part of what the user typed:

- **Main agent:** the screen the user is on, the selected Project `{id, name}`, the mode, and
  the user's decisions on change proposals since the last turn (`{proposal_id, outcome}`).
- **Sparring agent** (no tools): the full Project JSON (tens of KB at most). It role-plays the
  customer from that JSON.

Sessions are not bound to a Project (a conversation can move between Projects), so these facts
can't be session metadata. The workarounds all fail:

- **In the user's text:** the UI shows it, it mixes client facts with what the user typed, the
  eval simulator paraphrases it, and the frontend can't read it back as data.
- **In an MCP header envelope:** the LLM never sees it.
- **As session-sticky context:** only the latest survives, history no longer says what the model
  knew at each turn, and changing it rewrites the prompt prefix.

Success is judged by:
- A client attaches a JSON object to a turn. The model sees it on that turn and on every later
  replay of that turn, exactly as sent and before the user's text.
- After a reload, the client can read it back from session read as structured data, clearly
  marked so the UI can hide it.
- An eval built from such a session replays the same bytes to the target that production sent.
- Clients that don't send the field see no change at all.

## Change level

**C2.** It adds a field to two public turn APIs, adds a new stored content-block shape in
`chat_messages.content` / `deployed_messages.content`, changes the LLM-context rendering of
history, changes what session read returns, and changes the eval export contract
(`eval-input-v1`). There is no DB migration and no event change.

## Scope

### In scope

1. **Turn request field.** Both turn routes accept an optional top-level `client_context`:
   - `POST /deployed/{agent_id}/sessions/{session_id}/turn` (`internal/handler/deployed.go`,
     `turnRequest`, lines 272–278);
   - BO `POST /agent_versions/{v}/chat/{s}/turn` (`internal/handler/chat.go`, `TurnRequest`,
     lines 858–860).

   The value must be a JSON object. OpenBBC treats its content as opaque: no schema, no diffing,
   no merging with earlier blocks. The client sends it only when it changed; OpenBBC does not
   repeat earlier blocks.

   Validation runs after decoding and before the empty-turn check. Every failure is a 400
   returned before the stream opens. The response goes through the same plain-text path as
   `empty_turn` today (`http.Error` with the sentinel's text).
   - **`null` or absent:** no context; the behaviour is exactly as today.
   - **Not an object** (array, string, number, bool): 400 with body
     `"invalid client_context: must be a JSON object"` (new `types.ErrClientContextInvalid`).
   - **Contains a NUL character** (`\u0000`) anywhere in a key or string. Postgres JSONB rejects
     it, which today would fail only after the stream opens. The response is the same 400
     `invalid client_context…`, with the message suffix `: contains \u0000`.
   - **Too large:** the canonical string (Scope → In → 3) is bigger than
     `OPENBBC_CLIENT_CONTEXT_MAX_BYTES` (default `65536`). 400 with body
     `"client_context exceeds OPENBBC_CLIENT_CONTEXT_MAX_BYTES"` (new
     `types.ErrClientContextTooLarge`).
   - **Malformed JSON:** the existing decode error path, unchanged.

   The field is decoded as `json.RawMessage`. The handler checks that its first non-whitespace
   byte is `{`, then canonicalizes it (Scope → In → 3).

   **Empty-turn rule unchanged.** A turn with a `client_context` but no text and no pending
   artifacts is still a 400 `empty_turn`. The block never counts as user content for this rule,
   in the handler or in the transactional re-check (`decodeUserContent`).

   `OPENBBC_CLIENT_CONTEXT_MAX_BYTES` lives on `ChatConfig` (`internal/config/config.go`) and is
   parsed with `positiveIntEnv`, so a value below 1 fails `config.Load`.

2. **Stored block.** On a turn that carries a context, the user message's content array starts
   with one new block:

   ```json
   {"type": "client_context", "canonical": "<canonical JSON text>", "context": { ... }}
   ```

   It is always first, before the `text` block(s) and any `artifact_ref` blocks that
   `appendUserTurn` adds. The two fields have different jobs:
   - **`canonical`** is a JSON **string** holding the canonical JSON text (Scope → In → 3). It is
     the **only** source for rendering, the size cap and the export's `rendered`. Postgres JSONB
     stores strings byte-for-byte, so this text never changes after ingest.
   - **`context`** is the same object as a JSON value, for clients that read the session back as
     structured data. It is **informational only**: JSONB normalizes its numbers (`1e3` →
     `1000`) and its key order, so it is never used for rendering or measuring.

   In Go:
   - A new `llm.ClientContextBlock{Canonical string}` is added in `internal/llm/llm.go` and to
     the sealed `Block` interface.
   - `blocksToJSON` (`internal/chat/orchestrator.go:710`) writes both fields from `Canonical`.
   - `parseBlocks` (`orchestrator.go:664`) reads `canonical` back into a `ClientContextBlock`,
     which is a true round-trip; it ignores `context`.
   - The orchestrator's text-only input filter (`orchestrator.go:296–301`) keeps a
     `ClientContextBlock` and passes it through.
   - The test fake (`internal/chat/fakes_test.go:176–193`) round-trips the block the same way,
     and does not count it as text for the empty-turn rule.
   - Turns without the field produce exactly today's content. Stored history without the block is
     unchanged, and no migration is needed: the only CHECK constraints are on `role`.

3. **One rendering rule.** `chat.CanonicalizeClientContext(raw json.RawMessage) (string, error)`
   runs once, at ingest, in the handler. `chat.RenderClientContext(canonical string) string`
   returns `"<client_context>" + canonical + "</client_context>"`. The canonical text is the
   object re-encoded deterministically:
   - Object keys are sorted (Go `map` marshalling sorts them), at every depth.
   - There is no insignificant whitespace.
   - There is no HTML escaping (`json.Encoder.SetEscapeHTML(false)`), so `<`, `>` and `&` stay
     literal. The trailing newline that `Encoder` adds is trimmed.
   - Numbers are kept as their original literal (`UseNumber` on decode, `json.Number` on encode).
   - Strings are re-encoded by Go's encoder. That is deterministic but not always byte-for-byte
     what the client sent: U+2028/U+2029 are escaped as `\u2028`/`\u2029`, and invalid UTF-8
     becomes U+FFFD.

   Canonicalization happens **once, at ingest**. Its output string is what the size cap
   measures, what is stored in `canonical`, and what every render (live turn, every replay,
   export) wraps. Nothing re-canonicalizes later, and nothing reads the JSONB-normalized
   `context` for rendering. Replay is therefore byte-identical, and the cap measures exactly the
   bytes the model will see on every turn.

   This function is the **only** renderer of the block. aikdm does not re-implement it (see
   Scope → In → 6 and 7).

4. **LLM context: one conversion point.** `req.Messages` keeps the block form
   (`ClientContextBlock`) everywhere: in history from `parseBlocks`, and in the current turn's
   in-memory user message (`orchestrator.go:332–337`, which reuses the filtered input).

   The conversion to model-visible text happens in **exactly one place**: the per-LLM-call
   render pass that already builds a fresh LLM-bound copy of `req.Messages` before every
   `o.llm.Generate` call (`renderArtifactsForLLM`, `internal/chat/artifacts.go:79`, called at
   `orchestrator.go:381`).
   - That pass (or a sibling it calls) replaces each `ClientContextBlock` with
     `llm.TextBlock{Text: RenderClientContext(b.Canonical)}`, in its position, which is first.
   - It runs on every round of every turn and covers history and the current turn alike, so the
     model sees the context on the turn it arrives and on every later replay, in order.
   - Neither LLM adapter changes. Because both adapters silently drop unknown block types, a
     `ClientContextBlock` must never reach them. An orchestrator test asserts that no
     `ClientContextBlock` exists in the LLM-bound request.

   No system-prompt line is added. Agents that use the block explain it in their own prompt.

5. **Session read and BO view.**
   - `GET /deployed/{agent_id}/sessions/{session_id}` (`deployed.go:136–169`) already returns the
     raw stored `content`. The block appears there as
     `{"type":"client_context","canonical":"…","context":{…}}` on its user message, so clients can hide it by `type` and rebuild state after a reload.
     There is no code change on this route beyond what Scope → In → 2 stores.
   - `GET .../children/...` (child transcripts) never contains the block (Scope → In → 8).
   - The BO chat view (`internal/handler/chat.go` `decodeBlock`, lines 530–596; template
     `web/templates/chat/bubble.html`) gets a new `blockView.Kind` `client_context`. It renders
     as a collapsed `<details>` labelled "client context" above the user's text, showing the
     `canonical` JSON pretty-printed. The read-only child transcript view uses the same builder.

6. **Eval export.** `GET /evals/{id}/export.yaml` (`internal/eval/export.go:196–206`) already
   passes user-message content blocks through. For each `client_context` block it additionally
   emits `rendered: <RenderClientContext(canonical)>`, so the exported block is
   `{type: client_context, canonical: "…", context: {...}, rendered: "<client_context>…</client_context>"}`.
   - `rendered` is the **authority** for replay. `context` in the export is informational only:
     the export decodes content into `any`, so its numbers become float64 and may lose precision.
   - `rendered` exists only in the export. It is never stored and never returned on session
     read.
   - The export works the same for locked and closed sessions.

7. **aikdm replay** (amends eval-turn-replay's `UserStep`):
   - `render_user_turn(ref_msg)` puts each `client_context` block's `rendered` string first,
     **verbatim**, joined to the text blocks with `"\n\n"`, matching the open-bbcd block order.
     The two texts are separate content parts in open-bbcd and one joined string in aikdm (the
     target receives a single string per user message). The joining separator is the only
     difference, and it is accepted (see Risks).
   - In `paraphrase` mode the simulator writes only the user text. The runner attaches the
     rendered block before it, never through the simulator. The `client_context` text is removed
     from the `<reference_transcript>` and the explicit intent given to the simulator, so it
     cannot be paraphrased or dropped.
   - In `reference` history mode, earlier reference user turns are rendered by the same
     `render_user_turn`, so they carry their blocks too.
   - A `client_context` block without `rendered` (an export from before this change, or a
     hand-written dataset) is skipped with a warning logged per session. aikdm never
     re-serializes `context`.
   - The judge sees user messages as rendered, so it can tell the agent's use of the context
     apart from the user's text.

8. **Sub-agents.** Child sessions never receive the parent's block. A child's first user message
   is built only from the `agent` tool call's `prompt` (`internal/chat/subagent.go:173`), and
   that stays so. If the parent model chooses to quote the context in the prompt, that is model
   behaviour, not plumbing.

9. **AG-UI.** No new event. The block belongs to the user message, which the client already
   knows it sent. Today no event echoes user input (`internal/transport/agui/agui.go`).

10. **Tests.**
    - **Handler** (both routes):
      - an object is accepted;
      - `null` or absent behaves as today;
      - array, string and number give 400 `invalid client_context…`;
      - an over-cap object gives 400 `client_context exceeds…`;
      - context with no text and no pending artifacts gives 400 `empty_turn`;
      - none of the 400s writes SSE headers.
    - **Canonicalization:**
      - key order is sorted at every depth;
      - no whitespace;
      - `<>&` stay literal;
      - number literals such as `1.0`, `1e3` and big integers are preserved in `canonical`;
      - U+2028 is escaped deterministically;
      - `\u0000` in a key or a value is rejected with 400 before the stream opens;
      - the size cap is measured on the canonical string.
    - **Orchestrator** (fake LLM):
      - **the turn-1 LLM request already contains** `<client_context>…</client_context>` as
        the first text part of the current user message;
      - no `ClientContextBlock` ever reaches the fake LLM; only `TextBlock`s do;
      - turn 1 with a context, turn 2 without, turn 3 with a different context: the LLM request
        on turn 3 has exactly two `<client_context>` texts, each first in its own user message,
        in order;
      - turn 2's user message has none;
      - the bytes match `RenderClientContext`.
    - **Persistence:**
      - `blocksToJSON` / `parseBlocks` round-trip `ClientContextBlock`, and the block is stored
        first, before text and `artifact_ref`.
      - A **repository test against real Postgres** (both `chat_messages` and
        `deployed_messages`) stores a context containing `1e3`, `1.0`, a 30-digit integer and
        `-0`. Its read-back `canonical` is byte-identical to the input canonical string, while
        `context` is allowed to differ. A fake repo can't catch JSONB normalization.
    - **Session read:** the deployed read returns the block raw. The BO view renders the
      collapsed bubble, and an older session without the block renders unchanged.
    - **Export:** a session with the block exports `context` and `rendered`, and `rendered`
      equals `RenderClientContext`.
    - **aikdm:**
      - `render_user_turn` puts `rendered` first, verbatim;
      - paraphrase mode never sends the block to the simulator and still attaches it;
      - a block without `rendered` is skipped with a warning;
      - a cross-language golden test (mechanics below) passes.
    - **Golden fixture mechanics.**
      - A Go test in `internal/eval` (`TestClientContextGoldenExport`) builds an export for a
        session with two contexts that cover the edge cases above. It compares the YAML against
        the committed fixture `aikdm/tests/fixtures/client_context_export.yaml`.
      - It fails when they differ, unless it runs with `-update`, which rewrites the fixture.
      - The aikdm test loads that same fixture, runs `render_user_turn` on each user message, and
        asserts the exact expected strings committed next to it
        (`client_context_expected.json`, also written by the Go test with `-update`).
      - So a renderer change in Go fails the Go test until the fixture is regenerated, and a
        regenerated fixture fails the aikdm test until aikdm agrees. Neither side can drift
        silently.

### Out of scope

- **String-valued `client_context`.** The field is an object only.
- **A system-prompt explainer line** in OpenBBC, globally or per agent. Agent prompts own it.
  aikdm bundle generation may add such a line later; that is a separate change.
- **Server-side diffing, merging or deduplication** of contexts. The client sends a block when
  it changed.
- **Session-level or sticky context**, which the handoff rejected.
- **A context-only turn** (no text). It keeps returning 400 `empty_turn`.
- **Forwarding the block to sub-agents.**
- **An AG-UI event for the block.**
- **A general request-body size limit** on turn routes. Only `client_context` is capped here.
- **A JSON Schema for the context**, or validation of its content.
- **Message-level prompt-cache breakpoints.** None exist today. The deterministic rendering
  keeps old turns byte-stable, so they can be added later.
- **The handoff's related findings** (app context from the discovery zip, a current-date line,
  promotion under noise). Each needs its own ticket.

## Contracts

- **API: turn requests** (both routes). New optional body field
  `client_context: object | null`. The new 400 bodies are:
  - `invalid client_context: must be a JSON object`
  - `invalid client_context: contains \u0000`
  - `client_context exceeds OPENBBC_CLIENT_CONTEXT_MAX_BYTES`

  All are plain text, the same as the existing `empty_turn` on these routes (`deployed.go:334`,
  `chat.go:933`). That deliberately deviates from the `{error:{code,message}}` envelope in
  `docs/conventions/api.md`, for consistency within the turn routes. The change is additive: unknown
  fields are already ignored by `DecodeJSON`, so older clients are unaffected, and today's
  servers silently ignore the field from newer clients.
- **API: session read** (`GET /deployed/{agent_id}/sessions/{session_id}`). User-message
  `content` may now start with `{"type":"client_context","canonical":"…","context":{…}}`.
  `canonical` is the exact text the model saw; `context` is a convenience copy whose numbers
  and key order may be normalized. Clients that switch on
  block `type` must ignore unknown types. This is already required for `artifact_ref`. The post-merge
  `/arch-review` should state it in the `deployed-runtime` L2.
- **Data:** a new content-block shape in `chat_messages.content` and `deployed_messages.content`
  (JSONB arrays). No migration, no new column, no index.
- **Eval export (`eval-input-v1`):** `InputMessage.content` for user messages may contain
  `{type: client_context, canonical: string, context: {...}, rendered: string}`. `rendered` is
  authoritative, and `context` is informational. The change is additive; aikdm
  already loads `content` as opaque (`Any`). The schema version is not bumped.
- **LLM context:** for every user message with a block, the model sees
  `<client_context>{canonical json}</client_context>` as the first text part of that message.
- **Config:** new env `OPENBBC_CLIENT_CONTEXT_MAX_BYTES` (positive int, default `65536`).
- **Events:** none.

## Acceptance criteria

1. A turn with `client_context` stores it as the first block of its user message. The LLM
   request for that turn (including its very first LLM round) and for every later turn contains
   `<client_context>{canonical}</client_context>` as the first text part of that user message,
   in conversation order.
2. A turn without `client_context` stores and sends exactly what it does today. Earlier blocks
   are never repeated onto it.
3. The rendered bytes for a given block are identical on the live turn, on every replay and in
   the export, including after a Postgres round-trip of numbers like `1e3` and big integers.
   The canonical form has sorted keys, no whitespace, no HTML escaping and preserved number
   literals. The size cap measures those exact bytes.
4. Session read returns the block raw, marked by `type: client_context`. The BO chat view shows
   it as a collapsed "client context" section.
5. An oversized context, one that isn't an object, or one containing `\u0000`, returns 400 with
   the specified body, and no SSE stream opens. A context-only turn without text returns 400 `empty_turn`.
6. Clients that don't send the field, and stored history without the block, behave exactly as
   before. The existing open-bbcd test suite passes unchanged.
7. `GET /evals/{id}/export.yaml` emits `context` and `rendered` for every block, including on
   locked and closed sessions.
8. In aikdm, every replayed user turn whose reference carries a block starts with its `rendered`
   text, verbatim, in every eval-turn-replay mode. The simulator never sees or produces the
   block. The Go-to-Python golden fixture tests pass on both sides.
9. Child sessions never contain the parent's block.
10. `go test ./...` in `open-bbcd/` and `uv run pytest` and `ruff` in `aikdm/` pass.

## Risks & assumptions

- **Assumption: clients ignore unknown block types on session read.** A strict client that
  fails on an unknown `type` would break as soon as a session has a context. This is mitigated
  because only clients that send `client_context` get the block, and the rule is made explicit
  in the contract.
- **Risk: token cost.** Every block is replayed on every later turn, and the sparring agent
  sends tens of KB. The design accepts this: history must say what the model knew at each turn.
  The client controls it by sending only on change. The 64 KB cap bounds the exact bytes of a
  single block, because it is measured on the stored canonical string. **Non-goal:** there is
  no session-wide limit, log or metric on the total. Over a long session the sum can grow, and
  context-window limits apply as they do to long text today.
- **Risk: aikdm and open-bbcd text joining differ.** In open-bbcd the block and the user's text
  are separate text parts of one message. In aikdm they are one string joined with `"\n\n"`,
  because the target sends string content. Providers concatenate adjacent text parts, so the
  model-visible difference is at most a separator. The `<client_context>` bytes themselves are
  identical because they come from one Go renderer.
- **Risk: prompt injection through the context.** The context is client-controlled, the same as
  user text, and is shown to the model as data inside a tag. It grants no new capability. Agent
  prompts should treat it as facts from the client, not as instructions. This trust level is the
  same as the user's own text, and is no worse than the workaround the consumer uses today.

  Because HTML escaping is off, a string value containing `</client_context>` closes the
  wrapper early, and the text after it reads as if it were outside the block. This is accepted
  deliberately. The client already controls the user text next to it, so the break-out grants
  nothing new, and keeping `<`/`>` readable matters for the JSON the model reads (e.g. HTML
  snippets in Project data).
- **Assumption: there is no message-level caching today** (`anthropic.go` sets `cache_control`
  only on the system prompt and the last tool). If it is added later, byte-stable rendering of
  old turns keeps those prefixes cacheable.
- **Sequencing:** Scope → In → 7 changes code that eval-turn-replay creates. The aikdm part
  lands after eval-turn-replay's implementation, or in the same PR stack. The open-bbcd parts
  can ship first. The names this spec uses from eval-turn-replay (`UserStep`,
  `render_user_turn`, `paraphrase` user-turn mode, `reference` history mode,
  `<reference_transcript>`) match PR #19 as of commit `2f84cd0`. Until aikdm learns about `rendered`, today's aikdm sends the raw block to the
  simulator inside the reference-transcript JSON, like any other content. Nothing breaks, but
  the context is not reliably replayed until Scope → In → 7 lands.
- **Follow-up after merge:** run `/arch-review client-context` to sync:
  - the `deployed-runtime`, `feedback-datasets` and `evaluation` L2 READMEs (block shape;
    clients must ignore unknown block types);
  - `bizbok/information-map.md` (Chat message and Deployed message rows: content now includes
    `client_context`);
  - `glossary.md` (a "client context" term);
  - `ddd/context-map.md` (the `feedback-datasets` → `evaluation` export now carries `rendered`);
  - `c4/integrations.md` (turn request body);
  - wherever deployment config is documented (`OPENBBC_CLIENT_CONTEXT_MAX_BYTES`).
