---
id: aikdm
level: 1
parent: root
title: aikdm
---

# aikdm

Container: [aikdm](../../c4/containers.md#aikdm) · Context: [agent-lifecycle](../../ddd/contexts/agent-lifecycle.md)

## Purpose

Python 3.12+ CLI for LLM-heavy work: `generate-agent` (two-agent generator+critic loop
producing the aikdm bundle), `evaluate` (per-session simulator/target/tool_mock/judge
pipeline), and `train-agent` (hill-climb loop with `patience` early-stop). In `evaluate`
and `train-agent`, multi-agent versions run their real topology in-process from the
transitive `subagents` bundles in `eval-input.yaml` (the agent tool is never mocked;
`mock_mcp_tools` covers leaf MCP tools only); `generate-agent` does not emit agent-tool
config. Multi-provider
LLM via Google ADK + LiteLLM (Anthropic default, OpenAI, Gemini). Deps managed with `uv`.

Out-of-process and **DB-unaware** — only talks REST to `open-bbcd` through
`scripts/run_eval.sh` and `scripts/train_from_session.sh`, plus (for the alpha drainer
path) `generate_alpha.sh` which wraps `aikdm generate-agent` alongside `seed_bundle.py`
(the DB write happens in `seed_bundle.py`, not in `aikdm` itself). Non-zero exit produces
structured JSON on stderr (`{"error":"<kind>","details":"<msg>"}`; codes `1` unexpected · `2`
input/config · `3` LLM).

**Primary DDD context:** [`agent-lifecycle`](../../ddd/contexts/agent-lifecycle.md) (via
`generate-agent`). Also hosts
[`evaluation`](../../ddd/contexts/evaluation.md) (scoring pipeline in `evaluate`) and
[`training`](../../ddd/contexts/training.md) (hill-climb loop in `train-agent`).

**Runtime shipping forms** (not separate L1s):
- Standalone image `ghcr.io/dacdigital/openbbc/aikdm` (compose profile).
- Multi-layer image `ghcr.io/dacdigital/openbbc/aikdm-runner` (adds bash + curl + tini +
  `scripts/` + a `runner` uid) — consumed by the Helm chart's three CronJobs to drain
  `PENDING` alphas / evals / trainings. The runner image is aikdm's k8s packaging, not a
  separate service.
- Shell scripts under `scripts/` — helper glue absorbed into aikdm's release cycle.

## Scope (in / out)
