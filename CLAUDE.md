# TLCMap AI Skills — working conventions

AI skills for [TLCMap](https://tlcmap.org). Start with [README.md](README.md); the design is in
[documents/DESIGN.md](documents/DESIGN.md) and the use cases in
[documents/USE-CASES.md](documents/USE-CASES.md).

## Files

- **All files use LF (`\n`) line endings.** No CRLF, on any platform.
- UTF-8, no BOM. Files end with a single newline.
- Temporary scripts and scratch output go outside the repository, never into it.
- `.docx` files are pandoc build products — generate them, never commit them.

## Writing

- Australian English spelling and grammar.
- Never use the old name "GHAP"; it is "TLCMap".
- When documenting TLCMap behaviour, the **source code is the primary reference**. The published
  documentation and repository READMEs may be outdated — secondary reference only.
- Claims about the API are verified against production before they are written down, and dated.

## Design rules that constrain implementation

These are the ones most easily broken by accident. Full statements in
[DESIGN.md §3](documents/DESIGN.md#3-design-principles).

- **The model chooses; code computes; TLCMap supplies the facts.** No coordinate, count, distance
  or date is ever produced by a model. If a number appears in output, code computed it from data
  TLCMap returned.
- **Attribution travels; permission is never guessed.** Licence and rights text is surfaced
  verbatim. Never parse it into a yes or no.
- **Report what was excluded**, not just what was found. Silent omission is the failure mode this
  design exists to prevent.
- **Fix it upstream.** Where the TLCMap API is inadequate, the fix belongs in the API. There is one
  deployment (tlcmap.org), so do not write client-side workarounds or feature detection — record
  the prerequisite and move on.

## Skills

- One self-contained folder per skill under `skills/`, to the
  [Agent Skills open standard](https://agentskills.io): `SKILL.md` with `name` and `description`
  frontmatter, plus optional `scripts/`, `references/`, `assets/`.
- Keep `SKILL.md` to the portable core. No client-specific frontmatter.
- A skill must not reach outside its own folder — it has to work when copied into any client.
- Scripts declare their own dependencies with PEP 723 inline metadata and run under `uv run`.
- There is no build step. Verified against Claude Code, Codex and Gemini CLI.

## Editing the documents

- The documents cross-reference each other by section anchor. After editing a heading, check both
  files for links to it.
- DESIGN.md carries no decision history — state the decision, not how it was reached.
