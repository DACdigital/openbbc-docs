# training

## Purpose

Automate "eval → improve prompts → re-eval" as a bounded hill-climb loop. Each training
session originates from a DONE eval with `score < 1.0`; on success it materialises a new
agent version whose bundle strictly outperforms the parent's on the same input.

## Aggregates & entities

- **Training session** (aggregate root) — `training_sessions` row (migration 024). Fields:
  `source_eval_id`, `parent_version_id`, `new_version_id` (NULL until DONE), `status ∈
  {PENDING, IN_PROGRESS, DONE, FAILED}`, `epochs`, `patience`, `initial_score`, `final_score`,
  `total_epochs_run`, `stopped_reason`, `training_report` (JSONB), timestamps `requested_at`,
  `started_at`, `completed_at`, `error_message`.
- **Training report** (embedded value on Training session) — per-epoch teacher patches,
  candidate scores, promote/reject decisions, stopped reason.

<!-- migrated from _migration-quarantine/ARCHITECTURE.md § Training sessions, DESIGN.md § Phase IV on 2026-09-28 -->

## Domain events

N/A because OpenBBC does not emit domain events on any transport. State transitions in
this context are Postgres-only — `INSERT` / `UPDATE` against `training_sessions` — and
downstream consumers poll the REST surface for status changes (the training drainer
enumerates `GET /training-sessions.json?status=PENDING`; on DONE the agent-lifecycle
context reads the new `agent_versions` row via `new_version_id` directly, again over
REST).

## Invariants

- **At most one PENDING or IN_PROGRESS training session per source eval** — partial unique
  index `idx_ts_one_active_per_eval` (migration 024).
- **Creation gate:** the source eval must be `status = DONE`, `score < 1.0`, and no active
  training session for that eval (`internal/handler/eval.go:20-25`).
- **Hill-climb is strict-greater-than.** Candidate patches promote only if the resulting eval
  score is `>` best-so-far; equal scores do not promote. `patience` consecutive
  non-improvements → early stop.
- **Perfect-baseline shortcut.** If `initial_score >= 1.0`, skip the hill-climb loop entirely.
- **DONE inserts a new agent version.** `new_version_id` points at a newly-created
  `agent_versions` row that inherits the parent's `agents.architecture` (agent-level
  structural fields) and gets a new `prompts` JSONB.
- **State machine:** `PENDING → IN_PROGRESS → DONE|FAILED`; `FAILED` reachable from any state
  via `POST /training-sessions/{id}/fail`.

<!-- migrated from _migration-quarantine/ARCHITECTURE.md § Training sessions, DESIGN.md § Phase IV on 2026-09-28 -->

## Published surface

- **REST:**
  - `POST /training-sessions` (typically from BO Train button; public)
  - `GET /training-sessions`, `GET /training-sessions/{id}`, `GET /training-sessions/{id}/json`,
    `GET /training-sessions/{id}/report.json`, `GET /training-sessions.json?status=PENDING`
  - `POST /training-sessions/{id}/start`, `/complete`, `/fail`
- **UI:** `/training-sessions` (list) + `/training-sessions/{id}` (detail).
- **Consumed contract:** DONE eval row with `score`, `agent_version_id`, `dataset_version_id`.
- **Emitted contract for `agent-lifecycle`:** on DONE, a new `agent_versions` row via
  `new_version_id`.

<!-- migrated from _migration-quarantine/ARCHITECTURE.md § REST API, § Training sessions, DESIGN.md § Phase IV on 2026-09-28 -->
