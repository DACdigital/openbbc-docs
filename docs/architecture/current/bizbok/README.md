# BIZBOK lens — business layer

## What BIZBOK is

BIZBOK is the business-architecture lens: **what the system does, for whom, and via what
information**. It answers "what business capabilities do we expose, to which stakeholders,
along which value streams, over which information concepts?" — deliberately independent of
implementation technology.

## What lives here

- [`capabilities.md`](capabilities.md) — L1 + L2 capability tree with capability → DDD context
  + C4 container mapping.
- [`stakeholders.md`](stakeholders.md) — actor catalog: roles, goals, scope.
- [`value-streams.md`](value-streams.md) — end-to-end user journeys as capability sequences.
- [`information-map.md`](information-map.md) — canonical information concepts + owning
  capability + regulatory tag.

## How it connects to DDD and C4

BIZBOK names *what* the system does; [DDD](../ddd/README.md) carves *how the domain is
modelled*; [C4](../c4/README.md) shows *how it's built and runs*. Every L2 capability here
resolves to a DDD context (owning the domain rules) and a C4 container (running the code) via
the capability map in `capabilities.md`.
