---
id: flow-map-compiler
level: 1
parent: root
title: flow-map-compiler
---

# flow-map-compiler

Container: [flow-map-compiler](../../c4/containers.md#flow-map-compiler) · Context: [discovery](../../ddd/contexts/discovery.md)

## Purpose

Claude Code plugin skill (shipped from `bbc-discovery/flow-map-compiler/`) that scans a
client frontend repo and compiles a complete business-and-technical understanding of the app
into a structured `.flow-map/` wiki: `AGENTS.md`, `APP.md`, `glossary.md`, `skills/<id>.md`,
`flows/<id>.md`, `endpoints/<id>.md`. Feeds **both** prompt generation (skills + flows +
glossary become the runtime agent's context) **and** tool wiring (endpoints become
`tool_backends` bindings). Fully agent-driven — no scripted pipeline; contract triple
(`references/output-schemas.md` ↔ `references/lint-contract.md` ↔ `assets/templates/*.tmpl`)
governs output shape.

**LOCKED anti-goals.** Never generate MCP server code. Never assume an MCP server exists.
Never run target-repo code. Never call any registry API. Never generate runtime agent
prompts (that's the aikdm bundle's job).

**Runs client-side** on the discovery author's machine inside Claude Code — no direct
contact with `open-bbcd` REST beyond uploading the resulting zip via the wizard.

## Scope (in / out)
