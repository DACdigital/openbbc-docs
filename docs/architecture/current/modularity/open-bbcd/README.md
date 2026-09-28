---
id: open-bbcd
level: 1
parent: root
title: open-bbcd
---

# open-bbcd

Container: [open-bbcd](../../c4/containers.md#open-bbcd) · Context: [agent-lifecycle](../../ddd/contexts/agent-lifecycle.md)

## Purpose

Go 1.22+ daemon shipping the backoffice UI + REST API + deployed agent runtime + MCP-over-
REST bridge + artifact-store adapter in a single binary
(`gcr.io/distroless/static-debian12:nonroot`, CGO off). Two orchestrator instances (BO chat
+ Deployed) share a stateless `tools.Builder`. Owns every stateful thing in Postgres
transactionally on behalf of the DDD contexts — agents (+ discovery_zip BYTEA per
migration 026), agent_versions (`INITIALIZING/PENDING/DRAFT/TRAINING/READY/DEPLOYED` per
migration 025), tool_backends + `agent_endpoint_backend` + `agent_version_mcp_backend`,
chat_sessions + chat_messages + chat_message_feedback, datasets + dataset_versions,
evals + eval_sessions, training_sessions, deployed_sessions + deployed_messages, and
`artifact_ref` content blocks embedded in message JSONB.

**No local disk state.** Discovery zip inline in Postgres (mig 026); artifact bytes flow
through the pluggable `artifact-store-adapter` to the deployer's Object store (never touch
Postgres). Artifact-store registry is env-driven, hydrated at boot.

**Primary DDD context:** [`agent-lifecycle`](../../ddd/contexts/agent-lifecycle.md). Also
hosts [`feedback-datasets`](../../ddd/contexts/feedback-datasets.md),
[`evaluation`](../../ddd/contexts/evaluation.md) (state + UI),
[`training`](../../ddd/contexts/training.md) (state + UI),
[`deployed-runtime`](../../ddd/contexts/deployed-runtime.md), and
[`artifacts`](../../ddd/contexts/artifacts.md) (store-registry hydration + adapter
dispatch).

## Scope (in / out)
