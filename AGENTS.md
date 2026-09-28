# AGENTS.md

Cross-tool entry point for any agent working in this repo.

This is a **spec-driven docs repo**: features are specified and reviewed here before code is written
in the service repos under `workspace/`. Before doing feature work, read:

- [docs/process/README.md](docs/process/README.md) — the process: C0/C1/C2 change classification,
  the unified `/spec-review` gate and the on-PR `/arch-log-review` gate, per-feature flow, tracker
  adapters.
- [docs/process/AGENTS.md](docs/process/AGENTS.md) — **hard rules**: required spec sections and
  constraints that override any tool's looser defaults.
- [docs/conventions/](docs/conventions/) — cross-repo standards (architecture, naming, api,
  persistence, events, testing).

This file is a hub, not a duplicate — do not restate process content here; follow the links above.
