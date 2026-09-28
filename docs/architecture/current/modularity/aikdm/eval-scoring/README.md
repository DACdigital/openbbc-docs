---
id: eval-scoring
level: 2
parent: aikdm
title: eval-scoring
---

# eval-scoring

## Purpose

The `aikdm evaluate` subcommand: per-session simulator → target → tool-mock (or real MCP)
→ judge pipeline. Consumes `eval-input.yaml` from `GET /evals/{id}/export.yaml` (published
by open-bbcd's [`evaluation`](../../open-bbcd/evaluation/README.md) L2), emits
`eval-result.json` with a full `transcript` and per-criterion `judgments` JSONB per
simulated session.

Per-session flow: simulator LLM plays the user role; target LLM runs the agent version
under test; the `mock_mcp_tools` toggle on the eval row selects between synthetic tool
responses and real MCP calls (with `header_overrides` merged when real). The judge LLM
scores each acceptance criterion pass/fail.

Global pass-rate math (`sum(passed_criteria) / sum(total_criteria)` across all sessions)
happens on the open-bbcd side when it ingests the result at `POST /evals/{id}/result`; this
L2 emits per-session numbers only, not the final score.

Implements the `eval-scoring` L2 capability. Also serves as the reward function for
[`training-loop`](../training-loop/README.md).

## Scope (in / out)
