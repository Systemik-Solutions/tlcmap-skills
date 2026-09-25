# TLCMap AI Skills

AI skills that let a research assistant work with [TLCMap](https://tlcmap.org) — Australia's
Time Layered Cultural Map — as a structured source of historical place data, without inventing
any of it.

> **The model chooses; code computes; TLCMap supplies the facts.**

## Status

**Design complete; implementation not started.** The two documents below are the current
deliverable. Nothing in `skills/` exists yet.

## Documents

| Document | What it covers |
| --- | --- |
| [documents/DESIGN.md](documents/DESIGN.md) | The whole design — platform assessment, principles, the three layers, the build plan, risks, open questions |
| [documents/USE-CASES.md](documents/USE-CASES.md) | The eight research use cases in full, with example prompts and worked examples from real TLCMap data |

Read DESIGN.md first. USE-CASES.md is the detail behind its §8.

## The shape of it

Three layers, built bottom-up, because each one is the foundation for the next:

```
Layer 3  SKILLS        workflow, judgement, ethics     Markdown + Python   this repository
Layer 2  MCP SERVER    discrete tools, no judgement    PHP / Laravel       TLCMap application
Layer 1  API           the contract                    PHP / Laravel       TLCMap application
```

Only Layer 3 lives here. The API enhancements ([DESIGN.md §5](documents/DESIGN.md#5-layer-1--the-api))
and the MCP server ([§6](documents/DESIGN.md#6-layer-2--the-mcp-server)) are changes to the TLCMap
application itself, and most of them are prerequisites for the skills — so the plan sequences them
rather than working around them.

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
documents/     the design documents (this is currently all there is)
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
```

## Related repositories

- [GHAP](https://github.com/HughCraig/GHAP) — the main TLCMap application (Laravel); Layers 1 and 2 are built here
- [TLCMap Views](https://github.com/HughCraig/TLCMapViews) — the visualisation layer
- [TLCMap documentation](https://guide.tlcmap.org/) — including the [developer API reference](https://guide.tlcmap.org/developers/)

## Contributing

See [CLAUDE.md](CLAUDE.md) for the working conventions — they apply to people as well as agents.
