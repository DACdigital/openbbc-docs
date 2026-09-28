# Event & Message Conventions

Events are contracts too: consumers depend on their shape and evolution being predictable, not on the
producer's internals.

## Naming
- `<context>.<entity>.<pastTenseAction>` (see `naming.md`), e.g. `onboarding.user.registered`. One
  event states one fact that already happened, not a request to do something.
- Command-style messages (an explicit request for another service to act) are named as imperatives on
  a separate channel, kept distinct from fact-events.

## Versioning & schema evolution
- Every event payload carries an explicit `schemaVersion` (or is wrapped in a versioned envelope);
  never let consumers infer the version from the shape.
- Additive, backward-compatible changes (a new optional field) do not bump the version. Removing a
  field, changing a type, or changing semantics requires a new major version published alongside the
  old one until all consumers migrate.
- Store the schema (JSON Schema/Avro/protobuf) alongside the code that produces it; a schema check in
  CI (or a schema registry) blocks incompatible changes.

## Delivery semantics
- Assume **at-least-once** delivery from the broker; consumers must be idempotent — dedupe on event
  id — rather than relying on exactly-once guarantees.
- Producers stamp a stable, unique `eventId` per logical occurrence so consumers can dedupe safely
  across retries and redeliveries.

## Ordering
- Do not assume global ordering across topics or partitions. Where order matters within an entity,
  partition/key by that entity's id so its events stay ordered relative to each other.
- Consumers that need cross-entity ordering should reconstruct it from timestamps or sequence numbers
  in the payload, not from arrival order.
