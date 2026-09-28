# DDD lens — domain layer

## What DDD is

Domain-Driven Design is the domain-model lens: **how the problem space is carved into
bounded contexts, each owning its aggregates, invariants, events, and published surface**.
It answers "which team of concepts owns which rules, and how do contexts talk to each other
without leaking those rules?".

## What lives here

- [`context-map.md`](context-map.md) — the six OpenBBC bounded contexts and how they relate
  (upstream/downstream, ACL, conformist, partnership, open-host).
- [`contexts/agent-lifecycle.md`](contexts/agent-lifecycle.md) — agents, versions, MCP
  wiring, deployment.
- [`contexts/feedback-datasets.md`](contexts/feedback-datasets.md) — BO chat, per-message
  feedback, dataset lifecycle.
- [`contexts/evaluation.md`](contexts/evaluation.md) — evals and per-session judgments.
- [`contexts/training.md`](contexts/training.md) — training sessions and the hill-climb loop.
- [`contexts/deployed-runtime.md`](contexts/deployed-runtime.md) — production runtime
  sessions over AG-UI.
- [`contexts/discovery.md`](contexts/discovery.md) — `.flow-map/` compilation from a client
  frontend repo.
- [`access-model.md`](access-model.md) — auth pattern and policy table.

## How it connects to BIZBOK and C4

Every DDD context here owns some subset of the [BIZBOK capabilities](../bizbok/capabilities.md)
and runs inside one or more [C4 containers](../c4/containers.md). The capability →
context/container map in `bizbok/capabilities.md` is the source of truth for those joins.
