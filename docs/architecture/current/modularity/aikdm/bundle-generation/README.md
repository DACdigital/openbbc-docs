---
id: bundle-generation
level: 2
parent: aikdm
title: bundle-generation
---

# bundle-generation

## Purpose

The `aikdm generate-agent` subcommand: two-agent generator+critic loop over `.flow-map/`
inputs + domain-expert prompt guidance, producing the aikdm bundle (`bundle.yaml`).

Owns emission of bundles conforming to the versioned wire contract at
`aikdm/schemas/prompt-v1.yaml` — sections `metadata`, `main_prompt`, `capabilities[]`,
`skills[]`, `external_actions[]`. Same schema open-bbcd consumes for bundle persistence and
prompt rendering, and that `seed_bundle.py` reads on the alpha-drainer path. Schema bumps
(e.g. `prompt-v2.yaml`) are coordinated cross-service changes.

Runs in two shipping modes:
- **Compose profile** — one-shot from an operator against a local `open-bbcd`.
- **Alpha-drainer path** (production k8s) — the `aikdm-runner` CronJob invokes this
  subcommand via `scripts/generate_alpha.sh`, which then hands the resulting bundle to
  `seed_bundle.py`. `seed_bundle.py` writes the bundle directly to Postgres and transitions
  the agent version `PENDING → READY`. **The DB write lives in the script, not in this
  L2** — aikdm remains DB-unaware.

Implements the `agent-bundle-generation` L2 capability. Consumes multi-provider LLM
completions via Google ADK + LiteLLM (Anthropic default, OpenAI, Gemini).

## Scope (in / out)
