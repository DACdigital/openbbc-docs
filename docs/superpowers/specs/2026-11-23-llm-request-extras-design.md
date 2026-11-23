# llm-request-extras — config-driven per-model extra headers and body fields for outbound LLM calls

**Date:** 2026-11-23
**Status:** Draft — pending `/spec-review`
**Modularity attachment:** `docs/architecture/current/modularity/open-bbcd/` (L1). The LLM adapter
(Bifrost and direct Anthropic) is described there. It also affects [`aikdm`](../../architecture/current/modularity/aikdm/README.md)
(L1): `bundle-generation`, `eval-scoring` and `training-loop` all call models through LiteLLM and
must apply the same rules.

**Origin:** handoff from the ai-sales-copilot project (an OpenBBC consumer),
`.local-salescopilot/HANDOFF-openbbc-glm.md` (2026-11-23). The handoff proposes a hard-coded
"GLM mode". This spec replaces that with a generic, configuration-only mechanism. The GLM switch
becomes its first config entry.

**Grounded against:** OpenBBC `main` @ `bcab68c`, with `github.com/maximhq/bifrost/core v1.9.1`,
`anthropic-sdk-go v1.49.0`, `google-adk 2.1.0` and `litellm 1.86.1`.

## Business value / Why

Consumers run OpenBBC agents on whatever model their LLM gateway offers. New models regularly need
provider-specific request extras. The trigger here is GLM 5.3 behind the client's Bifrost gateway:
it only keeps its "thinking" short when a request carries the header
`x-bf-passthrough-extra-params: true` **and** the body field
`chat_template_kwargs: {"reasoning_effort": "low"}`.

The handoff measured what happens without them:
- **Output:** 3.5–4× more output tokens.
- **Speed:** 3–4× slower.
- **Cutoff:** a real RFP-analysis turn needed 4,192 output tokens, so with the production
  `OPENBBC_MAX_TOKENS=4096` it was cut off at `max_tokens`. With the extras it needed about 1,100.
- **No answer:** in the sister project, GLM without the switch sometimes thought for over 5 minutes
  without answering.

Today neither open-bbcd nor aikdm can send either extra by configuration. A per-model code switch
would need a release for every model change, version quirk, or gateway behaviour. In the handoff's
own data, only `low` works reliably, `medium` and `high` behave oddly, and the header is a gateway
feature, not a GLM one.

**Outcome:** an operator adapts OpenBBC to a model by editing one environment variable, in both
open-bbcd and aikdm, with the same rule meaning the same thing in both.

Success is judged by:
- The GLM handoff's checks pass with **no GLM-specific code**, only config:
  - the RFP turn ends `end_turn` under `OPENBBC_MAX_TOKENS=4096` (the gate);
  - the live tool-round test's time against its 93 s baseline is recorded for information.
- A model without a matching rule sends byte-for-byte the same request as today.

## Change level

**C2.** It adds a new configuration contract (`OPENBBC_LLM_REQUEST_EXTRAS`) read by two services
(open-bbcd and aikdm) and surfaced in compose and Helm. It changes outbound requests to external
providers. The config may carry credentials in header values. There is no DB, REST API or event
change. Per the hard rule (ambiguous → higher), this is C2 rather than C1.

## Scope

### In

1. **Config: `OPENBBC_LLM_REQUEST_EXTRAS`** (Contracts → 1). It is a JSON array of rules, each
   `{match, headers?, body?}`. Both open-bbcd and aikdm read it **under the same name and with
   the same semantics**. Unset or empty means no rules.
2. **Matching.**
   - `match` is a glob with exactly this syntax, implemented by a small matcher of its own in each
     service (not Go `path.Match` / Python `fnmatch`, which disagree):
     - `*` matches any run of characters, including none and including `/`;
     - `?` matches exactly one character;
     - every other character, including `[`, `]` and `\`, is literal;
     - matching is case-sensitive and covers the whole string.
   - The pattern is tested against the **effective** model string: the configured value with
     the service's own default applied, before any provider-prefix normalisation.
     - open-bbcd: `OPENBBC_DEFAULT_MODEL`, e.g. `openai/vllm/glm-5.3-flash`. When it is unset,
       the env default `claude-sonnet-4-6` is used.
     - aikdm: `AIKDM_MODEL_<ROLE>`. When it is unset, the role default `claude-haiku-4-5` is
       used.
   - Rules are checked in array order, and **the first match wins**. Matching rules are never
     merged.
   - No match means the request is unchanged.
3. **Validation at startup.** Any error fails boot (open-bbcd: `load config: …`; aikdm:
   `ConfigError`, exit code 2) with a message naming the rule index. Errors are:
   - invalid JSON, or a top level that isn't an array;
   - a rule that isn't an object, or has a missing or empty `match`;
   - a rule with neither `headers` nor `body`;
   - unknown keys in a rule;
   - a header value that isn't a string;
   - a `body` that isn't a JSON object;
   - a reserved **body** key at the top level: `model`, `messages`, `tools`, `tool_choice`,
     `stream`, `stream_options`, `max_tokens`, `max_completion_tokens`, `system`.
     Bifrost's merge would otherwise silently overwrite them.
   - a reserved **header**, case-insensitive: `Authorization`, `x-api-key`, `Content-Type`,
     `Content-Length`, `Host`, `Transfer-Encoding`.
4. **Startup log.** Log once per rule: index, `match`, header **names** and top-level body
   **keys**, never values. In open-bbcd, also log whether the configured model matches a rule and
   which one (`llm request extras active rule=0 match="*/glm-*"`), next to the existing
   `llm adapter ready` line. In aikdm, log the same per role.
5. **open-bbcd, Bifrost adapter.** When the configured model matches a rule:
   - The rule's headers go on every chat request through the per-request context
     `schemas.BifrostContextKeyExtraHeaders`, set in `Generate`.
   - The rule's body goes into `ChatParameters.ExtraParams`, built in `buildChatRequest`.
     **Whenever `ExtraParams` is non-empty**, `Generate` also sets
     `schemas.BifrostContextKeyPassthroughExtraParams = true`. Without that flag the SDK drops the
     body silently. This is the one place the flag is set.
   - Rules are resolved **once at boot** against the effective `OPENBBC_DEFAULT_MODEL` (Scope → In → 2). There is no
     per-request resolution. No code sets a different `llm.Request.Model` today, and per-version
     models are out of scope. *(Builder call: resolving per request would also have to match a
     bare `req.Model` without the provider prefix, which would make the same glob behave
     differently.)*
   - **Provider coverage.** Behaviour is guaranteed and tested for the Bifrost `openai` provider
     (the gateway path) and for the direct Anthropic adapter (Scope → In → 6). For the other
     Bifrost providers in the allow-list, the same mechanism is applied: context headers, plus
     `ExtraParams` with the passthrough flag. It is not separately tested, and the docs say so.
6. **open-bbcd, direct Anthropic adapter.** The same rules apply through
   `option.WithHeader` (per header) and `option.WithJSONSet` (per top-level body key) on
   `Messages.NewStreaming`.
7. **open-bbcd timeouts.**
   - Both variables are optional. When set, each must be a positive integer; anything else fails
     boot.
   - `OPENBBC_LLM_REQUEST_TIMEOUT_SECONDS`:
     - On Bifrost, it sets `NetworkConfig.DefaultRequestTimeoutInSeconds`. Unset keeps Bifrost's
       own default of 300.
     - On the direct Anthropic adapter, it is applied through `option.WithRequestTimeout`
       **only when set explicitly**. Unset means no timeout option, exactly as today.
   - `OPENBBC_LLM_STREAM_IDLE_TIMEOUT_SECONDS`, Bifrost only: it sets
     `NetworkConfig.StreamIdleTimeoutInSeconds`. Unset keeps Bifrost's default of 120.
   - With both unset, behaviour is identical to today on both adapters.
8. **aikdm.**
   - One shared helper resolves `(model) → (extra_headers, extra_body)` from the same variable.
   - `models.build_model` passes `extra_headers=` / `extra_body=` to ADK `LiteLlm(...)`, which
     forwards them to `litellm.acompletion`. This covers generator, critic, simulator, judge and
     teacher.
   - `eval/target.py` `target_turn` passes them to its direct `litellm.acompletion` call. This is
     the target role.
   - Each role matches against its own effective `AIKDM_MODEL_<ROLE>` (Scope → In → 2).
   - Without a match, no kwargs are added.
9. **Observability.** The Bifrost stream translator counts reasoning-delta characters
   (`reasoning_content` / `reasoning`, which it ignores today) and adds `reasoning_chars` to the
   existing debug call-log line. It is not shown to users and doesn't change the stream. aikdm
   gets no equivalent; LiteLLM usage logging is unchanged.
10. **Small fixes on the same path.**
    - `scripts/generate_alpha.sh` checks the API key of the configured generator and critic
      providers, rather than always `ANTHROPIC_API_KEY`. aikdm's own `load_settings` already fails
      fast.
    - `docker-compose.yml` passes `OPENAI_BASE_URL`, `OPENBBC_LLM_REQUEST_EXTRAS` and the two
      timeout variables through to the services that read them.
11. **Docs.**
    - `docs/PRODUCTION.md` §7 (bifrost bullets):
      - the gateway recipe (`OPENBBC_DEFAULT_MODEL=openai/<gateway model id>`, base URL **without**
        `/v1` for open-bbcd and **with** `/v1` for aikdm/LiteLLM);
      - the new variable, with the GLM rule as the worked example;
      - the reserved keys, and that **non-reserved** fields the adapter also sets (e.g.
        `temperature`, `top_p`, `thinking`, `metadata`) can be overwritten by a rule;
      - the timeouts;
      - the exact glob syntax, and a recommendation to use leading-`*` patterns so one rule fits
        both services' model strings;
      - a note that header values may be secret, so the variable belongs in a Secret
        (`envFromSecret` / `aikdmRunner.secrets`), not a ConfigMap.
    - `open-bbcd/.env.example` gets the same lines.
    - The Helm `values.yaml` comments mention the variable for `openbbcd` and `aikdmRunner`.

### Out

- **Per-agent-version model selection.** The model stays global in open-bbcd.
- **Showing reasoning text to users,** or streaming "thinking" indicators.
- **Hot reload:** rules change on restart only.
- **Per-role rule overrides in aikdm** beyond matching on each role's model string.
- **Merging multiple matching rules,** or templating values from other env vars.
- **A rules file / `_FILE` variable.** It was considered and rejected (Approach B); it can be
  added later without breaking this contract.
- **Allowing `vllm` as a Bifrost provider.** `openai` plus a base URL already works.
- **Validating that a provider accepts the extras.** That is the operator's responsibility.
- **Model quality decisions** (GLM vs Claude). The consumer decides those.

### Approaches considered

- **A. One environment variable holding JSON, shared by both services (chosen).** It fits
  open-bbcd's env-only configuration precedent (the artifact-store registry) and Helm's
  `extraEnv` / Secret plumbing, and needs no file mounts. It gets awkward if the rules grow long.
- **B. A rules file (`…_FILE`).** It is easier to edit, but there's no file-mount plumbing in
  compose or Helm today, so it means a new deployment surface for one setting.
- **C. Both, with the file winning.** It is the most flexible, but two code paths to test and
  document.
- **(Rejected) A hard-coded per-model mode, as in the handoff's "GLM mode".** It needs a release
  for each model or quirk change, and the header is a gateway concern.

## Contracts

1. **`OPENBBC_LLM_REQUEST_EXTRAS`.** An environment variable read by open-bbcd and aikdm.
   ```json
   [
     {
       "match": "*/glm-*",
       "headers": { "x-bf-passthrough-extra-params": "true" },
       "body": { "chat_template_kwargs": { "reasoning_effort": "low" } }
     }
   ]
   ```
   - Each rule is an object with keys `match` (required, non-empty string glob), `headers`
     (optional, object of string → string) and `body` (optional, JSON object). At least one of
     `headers` / `body` is required. No other keys are allowed.
   - `body` is merged into the provider request body, and adapters differ:
     - Bifrost deep-merges JSON objects.
     - The direct Anthropic adapter (`WithJSONSet`) replaces each whole top-level key.
     - LiteLLM `extra_body` adds the top-level keys.
     - Reserved keys fail boot (Scope → In → 3).
   - Semantics are identical in both services: the first rule whose glob matches the effective
     model string applies.
   - **Compatibility:** unset or `[]` means today's behaviour. The variable is new, so nothing
     existing breaks.
2. **`OPENBBC_LLM_REQUEST_TIMEOUT_SECONDS`, `OPENBBC_LLM_STREAM_IDLE_TIMEOUT_SECONDS`.** These
   are optional open-bbcd environment variables, positive integers when set. Unset means today's
   behaviour: Bifrost's 300 / 120, and no timeout on the direct Anthropic adapter.
3. **Outbound LLM requests.** For a model with a matching rule, requests gain the rule's headers
   and body fields. For all other models they are unchanged byte for byte, on all three
   outbound paths (Bifrost, direct Anthropic, aikdm/LiteLLM). This is pinned by tests on each
   path (Acceptance 1).
4. **Debug call log.** The existing per-call line gains `reasoning_chars` (int, Bifrost adapter
   only; 0 when the model streams no reasoning).
5. **Deployment surfaces.** compose passes the new variables through. Helm needs no template
   change, because `openbbcd.extraEnv` / `envFromSecret` and `aikdmRunner.extraEnv` / `secrets`
   already exist; only documentation is added. Note: `aikdmRunner.extraEnv` reaches the evals and
   trainings CronJobs, but not the alphas CronJob, so generation must get it through
   `aikdmRunner.secrets`. That is documented, not changed.
6. **Unchanged:** DB, REST API, events, `eval-input-v1`, `eval-result-v1`, and the AG-UI stream.

## Acceptance criteria

1. With `OPENBBC_LLM_REQUEST_EXTRAS` unset, and separately for a model that matches no rule,
   requests are unchanged on all three paths:
   - **Bifrost:** a request recorded by `bifrosttest` has the same headers and the same JSON body
     as before the change.
   - **Direct Anthropic adapter:** a request recorded by an `httptest` server is unchanged.
   - **aikdm:** `LiteLlm` / `litellm.acompletion` receive no `extra_headers` / `extra_body`
     kwargs (checked by monkeypatching).
2. With the GLM rule above and `OPENBBC_DEFAULT_MODEL=openai/vllm/glm-5.3-flash`, the recorded
   request carries the header `x-bf-passthrough-extra-params: true`, and its body contains
   `chat_template_kwargs.reasoning_effort == "low"`. This proves the passthrough flag is set.
3. The same rule set with `openai/gpt-4o-mini` → no extras (first match wins, no match →
   unchanged). With two matching rules, only the first applies.
4. Each invalid config in Scope → In → 3 fails open-bbcd boot, and makes aikdm `load_settings`
   raise `ConfigError`. In both, the message names the rule index and the offending field. The
   exact text may differ between Go and Python.
   - Shared fixtures live at the monorepo root under `testdata/llm-request-extras/`, read by
     both the Go tests (`open-bbcd/`) and the Python tests (`aikdm/`).
   - `invalid.json` lists invalid configs, each with its input plus the expected rule index and
     field. Both test suites read it, so the two services provably reject the same set.
   - `matching.json` lists `(rules, model) → expected rule index or none`. It pins identical
     matching in both services, including:
     - `*` crossing `/`;
     - literal `[`, `]` and `\`;
     - case-sensitivity;
     - the effective model with an unset variable (the service default).
5. The startup log shows rule matches, header names and body keys, and never a header or body
   value. This is asserted in a test with a secret-looking header value.
6. On the direct Anthropic adapter, a matching rule's header and body key reach the request,
   asserted against an `httptest` server.
7. aikdm: `build_model` and `target_turn` pass `extra_headers` / `extra_body` only for roles whose
   model matches, and pass nothing extra otherwise. This is asserted by monkeypatching `LiteLlm`
   and `litellm.acompletion`.
8. Set timeout variables reach Bifrost's `NetworkConfig`, and invalid values fail boot.
   - Unset leaves Bifrost's 300 / 120 in place.
   - On the direct Anthropic adapter, a set request timeout is applied. Unset adds no timeout
     option, which a test asserts on the client options.
9. `reasoning_chars` appears on the debug call-log line and counts reasoning deltas in a
   `bifrosttest` stream that contains them.
10. **Live (manual, opt-in):** GLM only, never a Claude model through the gateway, and no
    Anthropic key in the environment.
    - **Gate:** the handoff's RFP turn on a throwaway instance with `OPENBBC_MAX_TOKENS=4096`
      ends `end_turn`, not `max_tokens`. About 1,000–1,500 output tokens are expected, against
      about 4,200 without the rule.
    - **Informational, not a gate:** with the GLM rule set, `TestLive/openai/tool_round`
      against `vllm/glm-5.3-flash` passes, and its duration is recorded against the 93 s
      baseline. Gateway latency varies 9–149 s per call independent of our settings, so a time
      threshold would be flaky.
11. `scripts/generate_alpha.sh` with `AIKDM_MODEL_GENERATOR/CRITIC=openai/...` and only
    `OPENAI_API_KEY` set gets past its key check.
12. Existing Go and aikdm test suites, and the e2e suite (`make -C e2e e2e`), pass unchanged.

## Risks & assumptions

- **Secrets in the variable.** Header values may be credentials, and the startup log never prints
  them. Operators must put the variable in a Secret, which is documented. A ConfigMap would expose
  it to anyone who can read ConfigMaps.
- **Reserved-key guard is top-level only.** Nested fields that a provider treats specially can
  still be sent. That is accepted: the operator owns what they configure.
- **Name-based matching.** An alias like `openai/vllm/router` that routes to GLM gets no extras
  unless the operator adds a rule for it. This is the same accepted limitation the handoff notes,
  and the rule is now in the operator's hands rather than in code.
- **Same rule, different strings.** The two services match against their own effective model strings.
  If an operator writes `openai/vllm/glm-5.3-flash` in one and `vllm/glm-5.3-flash` in the other,
  a glob like `vllm/glm-*` matches only one of them. The docs recommend leading-`*` globs.
- **Merge semantics differ per adapter.** Bifrost deep-merges JSON objects, while the Anthropic
  adapter and LiteLLM set whole top-level keys. A body that collides with an adapter-set nested
  object behaves differently per adapter. That is unlikely given the reserved-key guard.
- **Non-reserved adapter fields can be overwritten.** A rule may set `temperature`, `top_p`,
  `thinking`, `metadata` and similar. That is accepted (the operator owns their config) and
  documented.
- **No HTTPS check on aikdm's base URL.** In open-bbcd, header values sent through Bifrost are
  covered by the `<PROVIDER>_BASE_URL` check: HTTPS, or plain HTTP to loopback only. aikdm uses
  LiteLLM's `OPENAI_BASE_URL` with no such check, so a credential-bearing header could travel
  unencrypted if an operator configures a plain-HTTP remote URL. This is accepted for this
  change and documented in `docs/PRODUCTION.md`. Adding the same check to aikdm is a possible
  follow-up.
- **Assumption (verified in code):** Bifrost v1.9.1 sends per-request context headers on the
  OpenAI streaming path without filtering `x-bf-*`, and merges `ExtraParams` only with the
  passthrough flag. ADK 2.1.0 `LiteLlm` forwards unknown kwargs to `litellm.acompletion`.
- **Gateway latency is not ours.** Longer timeouts let slow gateways finish but don't make them
  faster. A long multi-tool turn still multiplies the wait.
- **Architecture sync after merge.** `/arch-review` should update:
  - `c4/integrations.md` (LLM providers row: header values are credential-bearing);
  - `constraints.md` (new boot-validated variables);
  - the `c4/deployment.md` threat model (header values are a second route by which credentials
    could leak);
  - `assumptions.md` (aikdm now shares the rules variable).

  This edits existing rows. No new capability is added, so no `/new-feature`.
- **Quality on GLM is out of scope.** The handoff recommends re-running the consumer's datasets on
  GLM before switching, and notes that aikdm's structured-output roles have not been tried on GLM.
