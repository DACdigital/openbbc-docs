# skill-tool-version-prompts — `Skill` resolves from the version's prompts, at runtime and in evals

**Date:** 2026-11-21
**Status:** Draft — pending `/spec-review`
**Modularity attachment:** `docs/architecture/current/modularity/open-bbcd/tool-runtime/` (L2).
Callers [`deployed-runtime`](../../architecture/current/modularity/open-bbcd/deployed-runtime/README.md)
and [`feedback-datasets`](../../architecture/current/modularity/open-bbcd/feedback-datasets/README.md)
(BO chat) get the fix through the shared orchestrator; neither changes its own contracts.
Also affects [`evaluation`](../../architecture/current/modularity/open-bbcd/evaluation/README.md),
whose export switches to the shared skill-join helper with unchanged output, and
[`aikdm/eval-scoring`](../../architecture/current/modularity/aikdm/eval-scoring/README.md), where
skills move from inlined prompt text to the `Skill` tool. [`aikdm/training-loop`](../../architecture/current/modularity/aikdm/training-loop/README.md)
inherits that through `run_eval`, with no training code change.

**Origin:** handoff from the ai-sales-copilot project (an OpenBBC consumer), Item 2
("the `Skill` tool always answers 'unknown skill' in BO and deployed chats"). Item 1 of the same
handoff is the separate spec `2026-11-21-eval-mcp-tools-design.md`.

**Grounded against:** OpenBBC `main` @ `a56dfd7`. Paths are relative to `open-bbcd/` or
`aikdm/aikdm/` as stated.

## Business value / Why

Skills are how an agent keeps task-specific rules out of its main prompt and loads them on demand.
Today they work in evals and **never** at runtime:

- In BO and deployed chats, every `Skill` call with a valid name returns
  `{"error": "unknown skill: <name>"}`. The consumer had to copy every runtime-critical rule from its
  skills into the main prompt, which defeats the point of skills.
- Evals and training don't use the `Skill` tool at all. aikdm inlines every skill prompt into the
  system prompt, so evals never test whether the agent loads the right skill at the right time,
  which is exactly what production depends on.

Together these make "what we train" and "what runs" diverge in both directions.

Root causes in code:

1. **Runtime reads a key that never exists.**
   - `internal/chat/orchestrator.go` passes `agent.Architecture` to the tool handler as the bundle:
     `toolHandler.Tools(agent.Architecture)` (`:264`) and
     `toolHandler.Call(ctx, agent.Architecture, …)` (`:551`).
   - `internal/llm/tools/composite.go` (`:97-154`) parses `bundle.skills[].{name,prompt}` from it.
   - `agents.architecture` never has `skills`. `types.SplitBundle` (`internal/types/bundle.go:61-94`)
     and `scripts/seed_bundle.py` write only `skills_meta = [{name, description}]`, and the prompts
     go to `agent_versions.prompts.skill_prompts` (`map[name]prompt`).
   - So the tool is always offered with `"enum": []`, which no name can satisfy, and every call
     misses.
2. **The runtime never reads `skill_prompts`.** The orchestrator decodes only `main_prompt` from
   `version.Prompts` (`orchestrator.go:243-248`).
3. **Evals use a different mode.** `aikdm/eval/target.py` `build_system_prompt` (`:20-26`) appends
   every skill as `<skill name="…">prompt</skill>` after `main_prompt`, and offers no `Skill` tool.
4. **The skill join is duplicated.** The `skills_meta` + `skill_prompts` join is written inline in
   `handler/eval_store_adapter.go:75-81` (export) and `handler/configurator.go:328-336`, and is
   absent at runtime.
5. **The existing test hides the bug.** `composite_test.go:76` uses the fixture
   `{"skills":[{"name":"foo","prompt":"PROMPT"}]}`, a shape that never occurs in production.

Success is judged by:
- A BO or deployed chat calling `Skill` with a name from `skills_index` gets that version's prompt.
- Evals and training offer the agent the same `Skill` tool, with the same definition and the same
  results, so a skill is loadable in exactly the same cases in both places.
- The consumer can move skill-specific rules back out of its main prompt.

## Change level

**C2.**
- The LLM-facing tool contract changes at runtime: the `Skill` tool's `enum` is populated, and the
  tool is omitted when no skill is loadable.
- What every eval and training run measures changes for all consumers: skills are no longer
  inlined, and scores of skill-heavy agents will shift.
- There is no DB, REST API or event change, and the `eval-input-v1` / `eval-result-v1` wire shapes
  are unchanged. This sits between C1 and C2; per the hard rule, ambiguous → higher.

## Scope

### In

1. **Shared skill join (Go, `internal/types/bundle.go`).**
   - New `JoinSkills(arch Architecture, p Prompts) []BundleSkill`, the inverse of `SplitBundle`.
     It returns one entry per `skills_meta` item, in `skills_meta` order, as
     `{Name, Description, Prompt: p.SkillPrompts[name]}`.
   - Names present only in `skill_prompts` are ignored, which matches today's export.
   - A skill is **loadable** iff its `Prompt` is non-empty (after trimming whitespace). A helper
     `LoadableSkills([]BundleSkill)` filters to those.
2. **Runtime `Skill` tool resolves from the running version (tool-runtime).**
   - The orchestrator builds the skill list with `JoinSkills(agent.Architecture, version.Prompts)`
     once per turn, from data it already loads, and passes it to the tool handler for both `Tools`
     and `Call`.
   - The tool handler no longer parses skills from the architecture. `Builder.Build` keeps
     receiving the architecture for its endpoint tool list, unchanged.
   - Tool definition: name `Skill` and description unchanged (`composite.go:128`). The input schema
     is `{"name": {"type":"string","enum":[<loadable names>]}}`, required `name`.
   - When there are **no loadable skills**, the `Skill` tool is not offered at all. Today it is
     always offered with an empty `enum`.
   - Call results, with shapes unchanged:
     - A loadable name → `{"prompt": "<skill prompt>"}`.
     - Any other name, including one that is in `skills_meta` but has an empty prompt →
       `{"error": "unknown skill: <name>"}`, with `IsError: true`.
   - The fix applies to BO chats, deployed turns and agent-tool child sessions alike, since all of
     them run `Orchestrator.Turn` (`chat/subagent.go:173`). A child session resolves its **target**
     version's skills.
   - `MockHandler` (`internal/llm/tools/mock.go`) gets the same change, so mock-mode BO chats behave
     the same.
3. **Eval export uses the shared join (evaluation).**
   - `GetBundle` (`eval_store_adapter.go:39-87`) replaces its inline loop with `JoinSkills`.
   - Output is **unchanged**: every `skills_meta` entry, including empty prompts. This keeps
     training able to write a first prompt for a skill that has none (the patcher targets
     `skill.<name>.prompt`).
4. **aikdm uses the `Skill` tool instead of inlining (eval-scoring).**
   - `build_system_prompt` returns `main_prompt` only, the same system text the runtime sends.
   - `build_tool_specs` adds the `Skill` tool, with the same name, description and schema, and the
     `enum` set to the loadable skills. Loadable means a non-empty trimmed prompt, in bundle order.
     The tool is omitted when there are none.
   - `Skill` calls are answered locally from `bundle.skills`, with the same result shapes as the
     runtime. They never go through `ToolMock` or a real tool caller, whatever `mock_mcp_tools` is
     set to.
   - Training (`train-agent`) inherits this through `run_eval`. The teacher still patches
     `main_prompt` and `skill.<name>.prompt`, unchanged.
5. **Parity golden test (Go → Python),** same pattern as the client_context golden test.
   - A Go test builds the runtime `Skill` tool from a fixture version and writes
     `aikdm/tests/fixtures/skill_tool_parity.json`. The fixture version has `skills_meta` with three
     entries (two with prompts, one empty) and no `skills` key. The file records:
     - the bundle as exported;
     - the tool definition offered to the model;
     - the results for a hit, for a miss on an unknown name, and for a miss on the empty-prompt
       name.
   - A Python test loads the same bundle and asserts that aikdm's tool definition and results are
     equal (as parsed JSON).
6. **Go tests.**
   - Replace the unrealistic `{"skills":[…]}` fixture in `composite_test.go` with the production
     shape.
   - Cover:
     - a hit;
     - an unknown name, which returns the error;
     - an empty prompt, which means the skill is not loadable;
     - no loadable skills, which means no `Skill` tool;
     - an orchestrator-level test where a turn on a version with `skills_meta` + `skill_prompts`
       resolves a `Skill` call;
     - a child session that resolves its target version's skills.
   - Update the tool-count assertions that assume `Skill` is always present (`mock_test.go`,
     `builder_test.go`, `unmapped_backend_test.go`).
7. **aikdm tests.**
   - The system prompt no longer contains `<skill` blocks.
   - The `Skill` tool is offered, or omitted when there are no loadable skills.
   - A simulated turn that calls `Skill` gets the prompt and continues.
   - `Skill` is answered the same with `mock_mcp_tools` true and false.

### Out

- **`skills_index` sync.** `skills_index` is static text that aikdm renders into `main_prompt` at
  generation time (`aikdm/templates/main_prompt.xml.j2:30-34`). Nothing here regenerates it or
  checks it against the loadable list (see Risks).
- **Validating `skill_prompts` keys against `skills_meta`** in the configurator
  (`parsePromptsForm`) or in training completion (`TrainingSessionHandler.Complete`).
- **Configurator UI changes.** It already joins the two for display.
- **Changing `Skill`'s name, description or result shapes.**
- **Any change to the `eval-input-v1` or `eval-result-v1` schemas.**

## Contracts

1. **LLM-facing `Skill` tool (runtime and aikdm).**
   - Name `Skill`. The description is unchanged:
     "Load the prompt for a named skill into your working context. Use when the user's intent
     matches a skill in skills_index."
   - Input schema:
     `{"type":"object","properties":{"name":{"type":"string","enum":[<loadable names in skills_meta order>]}},"required":["name"]}`.
   - **Changed:** `enum` is populated from the version, where it is always `[]` today, and the tool
     is omitted when no skill is loadable.
   - Results (unchanged shapes): `{"prompt": "<text>"}` on a hit; `{"error":"unknown skill: <name>"}`
     with the error flag on a miss.
   - Pinned by `aikdm/tests/fixtures/skill_tool_parity.json`.
2. **Internal Go interface (tool-runtime → orchestrator).** The tool handler's `Tools` and `Call`
   take the joined skill list (`[]types.BundleSkill`) instead of parsing skills from the
   architecture `json.RawMessage`. This is internal only: tool-runtime publishes no external
   surface.
3. **`eval-input-v1`:** unchanged. `agent_version.bundle.skills[]` keeps `{name, description,
   prompt}` for every `skills_meta` entry. Its **meaning** for aikdm changes: the entries are now
   served through the `Skill` tool instead of being inlined.
4. **`eval-result-v1`:** unchanged. `Skill` calls appear in transcripts like any other tool call
   (`name: "Skill"`, `args: {name}`, `result`).
5. **DB, REST API, events:** none.

## Acceptance criteria

1. On a version whose architecture has `skills_meta` and no `skills`, and whose
   `prompts.skill_prompts` holds the prompts, a BO chat calling `Skill` with a `skills_meta` name
   that has a prompt receives `{"prompt": <that version's skill prompt>}`. The same holds for a
   deployed turn.
2. An unknown name, or a `skills_meta` name with an empty prompt, returns
   `{"error":"unknown skill: <name>"}` marked as an error.
3. The offered `Skill` tool's `enum` equals the loadable names in `skills_meta` order. A version
   with no loadable skills is offered no `Skill` tool.
4. An agent-tool child session resolves skills from its target version, not the parent's.
5. The `GET /evals/{id}/export.yaml` output for a fixture version is byte-identical before and
   after the change.
6. `aikdm evaluate`'s system prompt equals `main_prompt` (no `<skill` blocks), the model is offered
   the `Skill` tool, and a `Skill` call in a simulated session returns the prompt, in both mock and
   real tool modes.
7. The Python parity test passes against `skill_tool_parity.json` (tool definition, hit, unknown
   miss, empty-prompt miss).
8. `aikdm train-agent` runs end-to-end with the new mode; a patch to `skill.<name>.prompt` is
   reflected in the next candidate's `Skill` result.
9. Existing Go and aikdm test suites pass after the tool-count fixture updates in Scope → In → 6.

## Risks & assumptions

- **Eval and training baselines shift once.** Scores for skill-heavy agents will likely drop when
  skills stop being inlined, because the agent now has to load them. This is the intended, honest
  measurement, but it makes scores before and after the change incomparable. Consumers should
  re-baseline, and the release note will say so.
- **`skills_index` drift.** The agent picks skill names from the static `skills_index` text in
  `main_prompt`, while the `enum` comes from `skills_meta` + `skill_prompts`. If someone edits one
  and not the other, the model may ask for a name that isn't loadable and get "unknown skill". The
  `enum` constrains well-behaved models, and the error is visible in transcripts. Keeping the two in
  sync is a possible follow-up.
- **Empty-prompt skills disappear from the tool.** A skill listed in `skills_meta` with no prompt
  is no longer offered. We assume that is correct, since loading it would return nothing.
- **Assumption: no consumer relies on the current `{"skills":[…]}` architecture shape.** Nothing in
  OpenBBC writes it, as verified via `SplitBundle` and `seed_bundle.py`. An externally hand-written
  architecture with `skills` would stop being read; that shape is not part of any documented
  contract.
- **Token cost.** At runtime, skill prompts now enter the context when loaded, as designed. In
  evals, prompts get shorter by default, but each `Skill` call adds a round-trip, so eval cost and
  latency change per agent.
- **The orchestrator decodes `skill_prompts` every turn.** That is a negligible cost: the prompts
  are already loaded with the version row.
