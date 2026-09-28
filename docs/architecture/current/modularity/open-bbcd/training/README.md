---
id: training
level: 2
parent: open-bbcd
title: training
---

# training

## Purpose

Training-session state + Backoffice UI on the open-bbcd side. Does **not** run the
hill-climb loop itself — that lives in [`aikdm`](../../aikdm/README.md). This L2 owns the
state machine, the per-session gating rules, and result ingestion (materialising a new
agent version on `POST /training-sessions/{id}/complete`).

Owns table: `training_sessions` (state `PENDING → IN_PROGRESS → DONE|FAILED`, migration
024; partial unique index `idx_ts_one_active_per_eval` — at most one non-terminal session
per eval; `source_eval_id`, `parent_version_id`, `new_version_id` on DONE,
`training_report` JSONB on completion, `stopped_reason` on FAILED).

Publishes: `/training-sessions` (list + detail), `/training-sessions/{id}/start`,
`/training-sessions/{id}/complete` (inserts the new agent version in-transaction),
`/training-sessions/{id}/fail`, `GET /training-sessions.json?status=PENDING` (drainer
discovery surface), plus the Train button gate rendered on the eval detail page (requires
eval status `DONE`, `score < 1.0`, and no active training session).

Implements the open-bbcd side of the [`training`](../../../ddd/contexts/training.md) DDD
context. On DONE it inserts into [`agent-lifecycle`](../agent-lifecycle/README.md)'s
`agent_versions` table (the ACL between contexts — enforces `agent_versions.prompts`-only
mutation and never touches `agents.architecture`).

## Scope (in / out)
