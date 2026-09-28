---
id: training-loop
level: 2
parent: aikdm
title: training-loop
---

# training-loop

## Purpose

The `aikdm train-agent` subcommand: bounded hill-climb over prompt sections, using
[`eval-scoring`](../eval-scoring/README.md) as the reward function. Baseline eval on the
parent version establishes the score to beat; each epoch does teacher-LLM patch proposal
→ apply → run eval as reward → promote if candidate score is **strictly greater than** best
(non-strict equality does not promote — this is the locked convention). Early stop on
either `patience` consecutive non-improvements or a perfect score (1.0).

Emits `bundle.yaml` (the new agent version's prompts, conforming to the same
`aikdm/schemas/prompt-v1.yaml` contract as [`bundle-generation`](../bundle-generation/README.md))
plus `training-report.json` (per-epoch teacher patches, candidate scores, promote/reject
decisions, stopped-reason). The operator (or the training-drainer CronJob) posts these
back to open-bbcd's [`training`](../../open-bbcd/training/README.md) L2 via
`POST /training-sessions/{id}/complete`, which inserts the new agent version and flips the
session to DONE in-transaction.

Structural mutation is limited to `agent_versions.prompts` only — never
`agents.architecture` (enforced by the ACL between the `training` and `agent-lifecycle`
DDD contexts). Implements the `hill-climb-loop` L2 capability.

## Scope (in / out)
