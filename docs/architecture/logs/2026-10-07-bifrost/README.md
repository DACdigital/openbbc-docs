# bifrost — embedded Bifrost Go SDK as open-bbcd's multi-provider LLM adapter

**Date**: 2026-10-07
**Codename**: bifrost

**Driver**: new feature: bifrost. `open-bbcd` reaches models through a provider-agnostic
`llm.LLM` interface (`internal/llm`) that has exactly one implementation: a direct
`anthropic-sdk-go` adapter. Each new provider (OpenAI, Gemini, Bedrock, Vertex, …) would
mean another hand-written adapter: request mapping, stream normalisation, tool-use deltas
and multimodal rendering. We want one adapter that is the entrypoint to most models.

**Decision**: add a second `llm.LLM` adapter that wraps the **Bifrost Go SDK**
(`github.com/maximhq/bifrost/core`, Apache-2.0), embedded in-process in `open-bbcd`.
`OPENBBC_LLM_ADAPTER` (`anthropic` default | `bifrost`) selects the adapter once at boot.
That one adapter serves BO chat, the deployed runtime and every sub-agent. On `bifrost`:
- `OPENBBC_DEFAULT_MODEL` becomes `<provider>/<model>`. It stays deployment-global.
- Bifrost's `Account` is built in memory from per-provider `<PROVIDER>_API_KEY` env vars.
  Nothing goes into Postgres and no Bifrost config file is used.
- Fallbacks and load-balancing are not used yet.
- MIMEs the adapter cannot render natively fall back to the existing `TextSurrogate`.

The direct Anthropic adapter stays, and stays the default. `aikdm` is out of scope and
keeps Google ADK + LiteLLM. There is no new container, zone, bounded context or
modularity node.

**Rationale**: one Bifrost adapter gives `open-bbcd` access to 20+ providers through a
single normalised request/stream surface. New providers become configuration, not new
`llm.LLM` code. Embedding the SDK, rather than running the Bifrost gateway, keeps the
single-binary, no-sidecar, distroless deployment. It adds no network hop and no second
service to secure, and provider keys follow the same env-only pattern as today's
`ANTHROPIC_API_KEY` and the artifact-store credentials. Keeping the direct Anthropic
adapter as the env-selectable default avoids a big-bang switch. Existing deployments are
unchanged until they opt in, and there is a fallback path while the Bifrost adapter
reaches parity on streaming tool use and native image/PDF rendering. The model stays
deployment-global because a per-version model is a data-model change with eval/training
parity rules of its own. It is recorded as an open question instead.

**Alternatives rejected**:
- One hand-written `llm.LLM` adapter per provider: N adapters to write and maintain, each re-implementing stream and tool-use normalisation. This is what we want to stop doing.
- Bifrost (or LiteLLM proxy) deployed as an HTTP gateway container: adds a network hop, a new workload in the Helm chart, and a second auth/secret surface. Gateway features (UI, budgets, governance) are not needed today.
- Replacing the Anthropic adapter outright: rejected for now. It removes the fallback path before the Bifrost adapter is proven on streaming tool use and native media; retirement is tracked as an open question.
- Bifrost config file (JSON/YAML mounted via Secret/ConfigMap): a second config surface next to env vars. Env-only matches the existing LLM-key and artifact-store patterns.
- Per-version provider/model selection now: needs an `agent_versions` column and eval/training parity rules. Deferred as an open question.
- Bringing `aikdm` onto Bifrost too: `aikdm` already has multi-provider access through LiteLLM. Out of scope; runtime-vs-eval parity is tracked as an open question.

**Impact**:
- `docs/architecture/current/c4/integrations.md` — LLM providers row: adapter selection, Bifrost SDK, env-only `Account`.
- `docs/architecture/current/c4/context.md` — LLM providers external system mentions Bifrost (library, not external system).
- `docs/architecture/current/c4/containers.md` — open-bbcd label and LLM relation; Purpose paragraph on `llm.LLM` + `OPENBBC_LLM_ADAPTER`; tech stack lists both adapter SDKs.
- `docs/architecture/current/c4/deployment.md` — external zone note (no Bifrost workload); provider-credential-leak threat row covers multi-provider keys.
- `docs/architecture/current/assumptions.md` — new locked decision (Bifrost LLM adapter); four open questions (per-version model, fallbacks, retiring direct Anthropic, runtime-vs-eval parity).
- `docs/architecture/current/constraints.md` — `OPENBBC_LLM_ADAPTER`, `<provider>/<model>` format and boot-failure rules, CGO-off embedding / env-only keys.
- `docs/architecture/current/glossary.md` — new term: LLM adapter.
- `docs/architecture/current/modularity/open-bbcd/README.md` — Purpose mentions the boot-selected LLM adapter.

**Links**:
- Bifrost: https://github.com/maximhq/bifrost — Go SDK docs: https://docs.getbifrost.ai/quickstart/go-sdk/setting-up
