# modularize-aikdm — decompose aikdm into 3 L2 subcommand modules

**Date**: 2026-09-28
**Codename**: modularize-aikdm

**Driver**: /modularize run on aikdm

**Decision**: decomposed L1 `aikdm` into 3 L2 children: `bundle-generation`,
`eval-scoring`, and `training-loop` — one L2 per CLI subcommand.

**Rationale**: `aikdm` is a Python CLI shipping three subcommands (`generate-agent`,
`evaluate`, `train-agent`), each with its own input/output contract, its own LLM
orchestration shape, and its own release-cycle sensitivity. One L2 per subcommand is the
natural axis — each module maps 1:1 to a capability in `bizbok/capabilities.md`
(`agent-bundle-generation`, `eval-scoring`, `hill-climb-loop`), and the three modules
compose cleanly (training-loop uses eval-scoring as its reward function, but they remain
separately testable and separately versionable).

The versioned bundle-schema at `aikdm/schemas/prompt-v1.yaml` was **not** modelled as its
own L2 despite being a shared wire contract. Rationale: modules represent functional units
(code + responsibility), not passive artifacts. The schema is a header-file-like contract
declaration owned by `bundle-generation` as the primary emitter and referenced by every
other subcommand + by open-bbcd + by `seed_bundle.py`. Making it an L2 would fragment the
tree without payoff — same principle that kept `backoffice-ui`, `service-lifecycle`,
`llm-adapters`, and `cli-framework` out of the L1s and L2s elsewhere.

Naming choice: L2 slugs use descriptive nouns (`bundle-generation`, `eval-scoring`,
`training-loop`) rather than the CLI subcommand names (`generate-agent`, `evaluate`,
`train-agent`). The descriptive form is greppable by domain concern ("where's training?")
rather than requiring familiarity with aikdm's CLI first. `training-loop` in particular
was chosen over `hill-climb` (the algorithm-name variant) so the tree stays scannable by
someone thinking "training", not "hill-climb algorithm".

**Alternatives rejected**:
- **`bundle-schema` as its own L2** — rejected: the schema is a versioned contract file,
  not a functional module. Folded into `bundle-generation` as the primary emitter's owned
  contract. Consumers (`eval-scoring`, `training-loop`, open-bbcd, `seed_bundle.py`) all
  reference the same schema without needing it modelled as a peer L2.
- **`llm-adapters` as its own L2** — rejected: LiteLLM + Google ADK wrappers + provider
  dispatch (Anthropic / OpenAI / Gemini) + API-key handling from env are cross-cutting
  infra with no external contract; folded into each subcommand's implementation.
- **`cli-framework` as its own L2** — rejected: click-based CLI wiring, entrypoint, and
  error-envelope formatting on non-zero exit (`{"error","details"}` JSON on stderr; exit
  codes `1`/`2`/`3`) is too small to be a peer L2. Absorbed into each subcommand.
- **`rest-client-to-obbcd` as its own L2** — rejected: HTTP client wrappers used by
  `evaluate` and `train-agent` are ~50 lines of glue; folded into each subcommand.
- **`hill-climb` (as the row-3 slug)** — rejected in favour of `training-loop`: the
  algorithm-name form obscures the module's domain purpose when someone scans the tree.

**Impact**:
- + docs/architecture/current/modularity/aikdm/bundle-generation/README.md
- + docs/architecture/current/modularity/aikdm/eval-scoring/README.md
- + docs/architecture/current/modularity/aikdm/training-loop/README.md

**Links**:
- (user may add related spec PRs, feature docs, or tracker items before commit)
