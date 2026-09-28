# arch-schema-migration — reshape docs/architecture/current/ into schema

**Date**: 2026-09-28
**Codename**: arch-schema-migration

**Driver**: template pulled schema-enforcing /check-setup — migrate to conform

**Decision**: reshape docs/architecture/current/ from freestyle into the 20-file schema
defined in .claude/skills/check-setup/arch-schema.md

**Rationale**: /check-setup now enforces the schema; existing freestyle content (three
files adopted from DACdigital/OpenBBC docs/) must be bucketed into schema shape (or
quarantined for human review) before /check-setup can pass.

**Alternatives rejected**:
- Keep freestyle — rejected: /check-setup fails on missing required files.
- Manual reshape — rejected: LLM bucketing preserves provenance and structure at scale.

**Impact**:
- reshaped: docs/architecture/current/** (20 files written per schema — 4 top-level, 5 bizbok, 3 ddd + 6 context files, 6 c4)
- quarantined: docs/architecture/current/_migration-quarantine/{ARCHITECTURE.md,DESIGN.md,PRODUCTION.md} (source retained as audit trail)
- + docs/architecture/logs/2026-09-28-arch-schema-migration/README.md

**Links**:
- .claude/skills/check-setup/arch-schema.md
- docs/superpowers/specs/2026-09-18-arch-current-schema-design.md
