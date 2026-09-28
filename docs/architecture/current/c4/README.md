# C4 lens — system/infra layer

## What C4 is

C4 is the system-and-infrastructure lens: **what's built, in which containers, running where,
under which trust boundaries, integrating with what external systems**. It answers the "how
does it actually run" question — orthogonal to what the business does (BIZBOK) and how the
domain is modelled (DDD).

## What lives here

- [`context.md`](context.md) — system-boundary diagram + external actors + external systems.
- [`containers.md`](containers.md) — one `### <Container>` subsection per runtime binary /
  process, with tech stack, data ownership, published API, links to DDD context.
- [`data-flows.md`](data-flows.md) — canonical inter-container flows as sequence diagrams.
- [`deployment.md`](deployment.md) — environments, network zones, trust boundaries, STRIDE-lite
  threat model.
- [`integrations.md`](integrations.md) — external-system contracts with protocol, auth,
  SLA/regulatory.

## How it connects to BIZBOK and DDD

Every container here runs code that owns one or more [DDD contexts](../ddd/README.md); every
container is the runtime home for one or more [BIZBOK capabilities](../bizbok/capabilities.md).
The capability → context/container map in `bizbok/capabilities.md` is the source of truth for
those joins; `deployment.md` places each container into ≥1 network zone.
