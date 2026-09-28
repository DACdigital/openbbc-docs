---
id: evaluation
level: 2
parent: open-bbcd
title: evaluation
---

# evaluation

## Purpose

Eval state + Backoffice UI on the open-bbcd side. Kicks off runs (BO Evaluate button →
PENDING row), exports run inputs for aikdm consumption, ingests aikdm's results, computes
the global pass-rate score. Does **not** run the scoring pipeline itself — that lives in
[`aikdm`](../../aikdm/README.md).

Owns tables: `evals` (state `PENDING → IN_PROGRESS → DONE|FAILED`; `mock_mcp_tools` toggle;
`header_overrides` per-backend map per migration 023; per-run `score`, `passed_criteria`,
`total_criteria` after completion) + `eval_sessions` (per-simulator `transcript` +
per-criterion `judgments` JSONB, migration 022).

Publishes: `/evals` (list + create), `/evals/{id}` (detail with per-session breakdown +
Train button gate), `/evals/{id}/start`, `/evals/{id}/result`, `/evals/{id}/fail`,
`/evals/{id}/upload-result` (internal aikdm callback), `GET /evals/{id}/export.yaml` (input
material aikdm reads), `GET /evals.json?status=PENDING` (drainer discovery surface).

Score formula (locked): `sum(passed_criteria) / sum(total_criteria)` across all sessions in
the eval — every criterion counts equally; longer sessions weigh proportionally more.

Implements the open-bbcd side of the [`evaluation`](../../../ddd/contexts/evaluation.md) DDD
context. Downstream `training` (Train-button gate); upstream `feedback-datasets` (CLOSED
dataset version) + `agent-lifecycle` (target agent version).

## Scope (in / out)
