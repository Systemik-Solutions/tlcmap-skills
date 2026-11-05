# TLCMap AI Skills

AI skills that let a research assistant work with [TLCMap](https://tlcmap.org) — Australia's
Time Layered Cultural Map — as a structured source of historical place data, without inventing
any of it.

> **The model chooses; code computes; TLCMap supplies the facts.**

## Status

**Design complete; milestone A0 done.** The MCP server skeleton,
[tlcmap-mcp](https://github.com/Systemik-Solutions/tlcmap-mcp), runs on staging at
`https://test-mcp.tlcmap.org` with one read-only tool, `tlcmap_get_layer`
([acceptance record](https://github.com/Systemik-Solutions/tlcmap-mcp/blob/main/docs/acceptance/A0.md),
2026-11-03). Nothing in `skills/` exists yet.

## Documents

| Document | What it covers |
| --- | --- |
| [documents/DESIGN.md](documents/DESIGN.md) | The whole design — platform assessment, principles, the three layers, the build plan, risks, open questions |
| [documents/USE-CASES.md](documents/USE-CASES.md) | The eight research use cases in full, with example prompts and worked examples from real TLCMap data |
| [documents/specs/A0-SERVER-SKELETON.md](documents/specs/A0-SERVER-SKELETON.md) | Milestone A0 expanded into a buildable spec — the MCP server skeleton and its staging deployment |
| [documents/plans/A0-SERVER-SKELETON-PLAN.md](documents/plans/A0-SERVER-SKELETON-PLAN.md) | The task-by-task implementation plan for A0 |
| [documents/specs/A1-API-V2.md](documents/specs/A1-API-V2.md) | The API half of milestone A1 — the first API v2 endpoints, items ① to ⑧ |
| [documents/plans/A1-API-V2-PLAN.md](documents/plans/A1-API-V2-PLAN.md) | The task-by-task implementation plan for the A1 API half |
| [documents/LARAVEL-UPGRADE.md](documents/LARAVEL-UPGRADE.md) | Upgrading the TLCMap application from Laravel 5.8 to 13, with a time estimate — deferred to a later phase; nothing here depends on it |

Read DESIGN.md first. USE-CASES.md is the detail behind its §8.

## The shape of it

Three layers, built bottom-up, because each one is the foundation for the next:

```
Layer 3  SKILLS        workflow, judgement, ethics     Markdown + Python   this repository
Layer 2  MCP SERVER    discrete tools, no judgement    TypeScript          its own repository and service
Layer 1  API v2        the contract                    PHP / Laravel       TLCMap application
```

Only Layer 3 lives here. The API work ([DESIGN.md §5](documents/DESIGN.md#5-layer-1--the-api))
is a new version, v2, built in the TLCMap application beside the current API, which stays
untouched. The MCP server ([§6](documents/DESIGN.md#6-layer-2--the-mcp-server)) is a separate
service that reaches TLCMap only through v2. Most of both are prerequisites for the skills — so
the plan sequences them rather than working around them.

Seven skills are planned ([§7.3](documents/DESIGN.md#73-the-skills)):

| | |
| --- | --- |
| `tlcmap-search` | Find and retrieve places and layers |
| `tlcmap-resolve` | Placename strings → TLCMap records |
| `tlcmap-geoparse` | Places mentioned in a document → mentions with offsets |
| `tlcmap-analyse` | Spatial and temporal distribution |
| `tlcmap-visualise` | Maps, timelines, charts, embeds |
| `tlcmap-prepare` | Build and validate a TLCMap-ready layer |
| `tlcmap-cite` | Attribution, licensing, citable packaging |

The proof of concept is a vertical slice through use case 3 — a regional fieldwork brief for the
Hunter Valley ([§9](documents/DESIGN.md#9-the-vertical-slice)).

## Repository layout

```
documents/     the design documents, with specs/ and plans/ per milestone
skills/        the deliverable — one self-contained folder per skill  [not yet created]
tests/         unit tests + the production compatibility check        [not yet created]
evals/         gold sets, fixtures, ground-truth artefacts            [not yet created]
examples/      worked demonstrations                                  [not yet created]
```

Skills are built to the [Agent Skills open standard](https://agentskills.io) and distributed with
`gh skill`. There is no build step and no per-vendor packaging; see
[§4.3](documents/DESIGN.md#43-packaging-and-distribution).

## Generating the documents as `.docx`

[Pandoc](https://pandoc.org) is required. The `.docx` files are build products and are not
committed.

```sh
pandoc documents/DESIGN.md    -o documents/DESIGN.docx    --toc --toc-depth=3
pandoc documents/USE-CASES.md -o documents/USE-CASES.docx --toc --toc-depth=3
pandoc documents/LARAVEL-UPGRADE.md -o documents/LARAVEL-UPGRADE.docx
```

## Related repositories

- [GHAP](https://github.com/HughCraig/GHAP) — the main TLCMap application (Laravel); Layer 1 is built here
- [tlcmap-mcp](https://github.com/Systemik-Solutions/tlcmap-mcp) — the MCP server (TypeScript); Layer 2 is built here
- [TLCMap Views](https://github.com/HughCraig/TLCMapViews) — the visualisation layer
- [TLCMap documentation](https://guide.tlcmap.org/) — including the [developer API reference](https://guide.tlcmap.org/developers/)

## Contributing

See [CLAUDE.md](CLAUDE.md) for the working conventions — they apply to people as well as agents.
