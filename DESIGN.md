# TLCMap AI Skills — design and plan

**Status:** draft for discussion · **Date:** 2026-09-24 · **Phase:** design, nothing implemented

## Decisions taken

| Decision | Choice | Consequence |
| --- | --- | --- |
| Proof-of-concept scope | **Broad** — retrieval through to visualisation (Phase 1) | Proves the whole shape against today's read-only API; the resolver follows in Phase 2 |
| Skill split | **Seven skills**, as designed in §5 | Each keeps a distinct trigger; they compose |
| Audience | **A capability TLCMap itself hosts** | Reshapes §4.1, §6, §8.1 and §9 — see below |

The third decision is the one with teeth. It means:

- **The toolkit is shared code inside the plugin, structured so it could be extracted later.**
  A `lib/` directory that every skill's scripts import — the standard skills-with-scripts
  pattern, supported by Claude Code, Codex and anything else that runs a skill. Publishing it
  as an installable package is a *separate, later, optional* decision (§4.3), driven by
  whether "researcher in a notebook, no agent" turns out to be a real audience. It is not a
  precondition for the skills or for MCP, and nothing in Phase 1 depends on it.
- **MCP moves forward, and does not wait for the write API.** TLCMap's read API needs no
  authentication, so a hosted read-only MCP server is deployable *now*. That was the
  assumption worth revisiting: MCP was previously tied to auth, but auth is a reason MCP
  becomes *necessary*, not a precondition for it being *useful*. A server at
  `mcp.tlcmap.org` reaches Claude Desktop, a chatbot on the TLCMap site, and any other agent
  framework — audiences a skill plugin cannot.
- **Platform gaps get fixed in the platform** (§3.8). Both sides are in-house and there is a
  single deployment, so §8 is a list of **prerequisites** on the critical path — not gaps the
  skills route around. This design contains no workarounds, no fallbacks and no feature
  detection; a skill ships when its prerequisite is live (§8.6). The layer extent facet is the
  worked example: one `GROUP BY dataset_id` on the server, which fixes the browser interface
  at the same time.
- **Distribution, versioning and support become real constraints.** Third parties will install
  the plugin, and the API is explicitly unversioned (§2.4). CI, tests and a compatibility
  check against production are worth having whether or not anything is ever published —
  they protect the skills, not a package.

---

## 1. What we are building and why

TLCMap holds curated Australian historical place data that nothing else holds: the ANPS
Gazetteer's colonial-era name variants, two thousand contributed layers, and the texts that
have been geoparsed into them. It publishes all of it over plain HTTP with no key and no
client library.

What it does not have is anything that turns a research question into a query, or a query
result into an answer. That gap is the whole opportunity. A researcher who knows exactly
which of `name`, `containsname` and `fuzzyname` to use, that `sort` silently deletes every
undated record, and that `limit` returns a *random sample* rather than the first N, can get
a lot out of TLCMap today. Nobody knows those things without reading the developer docs end
to end.

**The proposition:** a set of agent skills that carry that knowledge, retrieve TLCMap data
correctly, and apply reusable analysis, visualisation and round-trip capabilities on top of
it — so that a natural-language research request produces a citable, reproducible result.

This document proposes the skill set, the shared toolkit beneath it, the boundary between
what the model decides and what code computes, the gaps in the platform that need filling,
and a phased plan with a concrete first demonstration.

---

## 2. What the platform actually offers today

Grounded in the developer documentation and in live checks against production on
2026-09-23.

### 2.1 Reads that work well

| Capability | Endpoint | Notes |
| --- | --- | --- |
| Name search across all four sources | `GET /places?format=json&…` | Ceiling of 5,000 **total** matches |
| Paged harvesting | `GET /api?format=json&per_page=&page=` | No ceiling, follows `next`; **gazetteers only** |
| Single place | `GET /places/{id}/{format}` | The citable URL |
| Layer contents + metadata | `GET /layers/{id}/json` | The right way to read contributed data |
| Layer catalogue | `GET /layers/json` | 2,118 layers, 2.7 MB, no paging, no filter |
| Multilayer manifest | `GET /multilayers/{id}/json` | Points at component layer feeds |
| Server-side analysis | `/layers/{id}/{basicstatistics,clusteranalysis,temporalclustering,closenessanalysis}/json` | Public layers only |
| Snapshot packaging | `/layers/{id}/ro-crate`, `/places?format=rocrate` | Frozen, citable |
| Visualisation | `views.tlcmap.org/latest/{3d,cluster,journey,timeline,werekata,fulltext}.html?load=` | Configured by the feed, not the URL |
| Text ↔ place linkage | `GET /layers/{id}/json?textmap` | Character offsets per mention |

### 2.2 Measured on the live catalogue

```
public layers ........ 2118
with description ..... 2113
with a licence ......... 704   (free text, not identifiers)
with a citation ........ 323
with a warning ......... 165   (cultural sensitivity / data quality)
with temporal extent ... 641
with a bounding box ...... 4
```

**The last number is the single most consequential fact in this document.**

`latitude_from`, `latitude_to`, `longitude_from` and `longitude_to` are *contributor-declared*
metadata — typed into a form, never computed from the records. Four layers out of 2,118 have
them (78, 284, 1341, 1333); all four fields are null on the rest. The per-layer feed's
`?metadata` has the same gap, so the catalogue is not withholding anything it holds.

The information exists, it is just not exposed in aggregate. Layer 152, for instance:

```
46 records, every one with coordinates
DECLARED bbox : null, null, null, null
ACTUAL bbox   : lon 4.7978..116.2544   lat -34.3568..53.0548
```

So **layer discovery by region is impossible from the catalogue**, and there is no `bbox`
parameter on `/layers/json` to do it server-side either. That blocks use case 3 and half of
use case 4.

The fix is one `GROUP BY dataset_id` with `ST_Extent` on the server, exposed as a catalogue
facet — **§8.1 ①, a prerequisite for `tlcmap-search`** and the highest-value change on the
list. It fixes the same discovery gap in the browser interface at the same time.

::: warning
**Compute the extents; do not promote the declared fields.** The four that exist cannot be
trusted either: layer 284 declares `latitude_from: -10, latitude_to: -30` — "from" is *north*
of "to" — and layer 1341 declares `longitude_to: 182.167965`, outside the valid ±180 range.
Keep the contributor-supplied values as a separate hint, and derive the facet from geometry.
:::

### 2.3 Behaviours that will break a naive client

Every one of these is a silent failure — wrong data, not an error. This table is the evidence
base for §8: each row is something the API should stop doing, not something the toolkit
should learn to survive (§3.8). The right-hand column is why each one matters.

| Trap | Consequence |
| --- | --- |
| `limit=N` takes a **random sample** (`shuffle()` then `take()`) | Two identical requests return different records. Never use it to truncate a result set. |
| `sort=anything` drops records with no start *or* end date | A 216-record search returns 55 with `sort=title`. Same for `line=time`. |
| Dated searches exclude undated records entirely | Most gazetteer records are undated; adding a date bound can cut results by an order of magnitude. |
| >5,000 matches → `302` to `/maxpaging` (HTML) | A client following redirects gets HTML with a `200` where it expected GeoJSON. |
| Private or missing layer → `200` with a FeatureCollection, no `features` | Detect by the **absence of `features`**; the warning key is misspelled `warnnig` here. |
| Unparseable `extended_data` expression is **discarded silently** | `Capacity>200` (no spaces) returns the entire unfiltered result set. |
| Search output sets `udateend = udatestart` | Every record looks like a single instant on a timeline. Layer feeds are correct. |
| `name` matches `title` only, never `placename` | The parameter people reach for first is the one that misses. |
| Extended data is merged into `properties` | A contributor's column named `description` silently replaces the built-in one. |
| Import sanitises field headings | `Catalogue no. 3` → `Catalogue no ` (trailing space significant); `Area m2` → `Area m`. |
| One unparseable date aborts an entire upload | No partial import. |
| `bbox`/`polygon` take **longitude first**; polygon rings must be closed | Silently wrong area, or a PostGIS error. |
| DBScan `distance` is divided by 100 and passed as **degrees** | Labelled km in the UI. `distance=100` ≈ 111 km N–S, less E–W. |
| Missing analysis parameter → `500` | Send every parameter, including empty ones (`withinRadius=`). |
| `basicstatistics/json` returns **geometry only** | The actual statistics exist only in the HTML page. |
| No rate limiting, no versioning | Politeness and caching are the client's responsibility. |

### 2.4 Not available

- **Any write.** No create, update or delete. Everything a skill produces must currently be
  handed to a human to upload through the browser.
- **Private layers.** No token exposes them to a script, by design.
- **Catalogue filtering or search.** Fetch 2.7 MB and filter locally.
- **Statistics as numbers.** Only as a rendered page.
- **Change feeds.** No `updated_since`, no ETag — a scheduled re-run cannot ask what changed.

---

## 3. Design principles

These are the load-bearing decisions. Everything in §5 follows from them.

### 3.1 The determinism boundary

The most important rule in the project, and the one that makes the output citable.

> **The model chooses; code computes; TLCMap supplies the facts.**

| The model may | The model may never |
| --- | --- |
| Pick which API parameter expresses a request | Produce a coordinate |
| Rank candidate records using textual evidence | Produce a date, a count, a distance or a density |
| Classify a record against a taxonomy | Assert a place exists that TLCMap did not return |
| Draft prose *from computed numbers* | Fill a gap with a plausible-looking record |
| Say "unknown" | Infer a licence permission from free text |

Enforced mechanically, not by instruction: every exported coordinate is validated against
the cached API response it came from, and an export whose coordinates do not trace back to
a fetched record fails. Every model judgement is written to an audit log with the evidence
it was shown.

### 3.2 Provenance by construction

Every artefact carries how it was made: the query URL (TLCMap returns the canonical form in
`metadata.url`), the fetch timestamp, a checksum of the response, the TLCMap UID of every
record, and — for model judgements — the candidates considered and the confidence. A result
a reviewer cannot retrace is not a research output.

### 3.3 Attribution travels, permission does not get guessed

`license` and `rights` are free text. A layer may say "CC BY 4.0", or a sentence, or
nothing. **No skill ever parses those fields into a yes/no.** It surfaces them verbatim,
alongside `creator`, `citation` and — critically — `warning`, the field contributors use
for cultural sensitivity notices. If data is republished, the warning is republished with
it. Layers relating to Indigenous knowledge are routed to a human decision, never an
automated one. (CARE principles; see §11.)

### 3.4 Report what was excluded

The API will report its own failures honestly once §8.1 ② lands — a rejected filter is a
`400`, a private layer is a `403`, an oversized result is a `413` — so the client checks
status codes rather than sniffing content types. That is the whole of error handling, and it
is why none of it appears in the toolkit.

What remains is not error handling but *honesty about scope*, and it stays:

- **Exclusions are reported, always.** A dated search leaves out undated records; a sorted
  one leaves out records with no date to sort by. "216 records, 55 of them dated" is the
  truthful version of a temporal claim, and every summary carries it.
- **Contributor-supplied metadata is evidence, not fact.** Declared extents can be malformed
  (§2.2), licences are free text (§3.3), and extended-data values are untyped strings. These
  are properties of community-contributed data, not API defects, and no endpoint will fix
  them. Validate, surface, and never silently repair.
- **Model output is verified against the cache** (§3.1). That guards against the model, not
  the API, so it is unaffected by anything in §8.

### 3.5 Files, not context

TLCMap responses are large — a 2.7 MB catalogue, feeds of thousands of features. Nothing of
that size passes through the model. Scripts write to disk, the model reads a summary. This
is why the initial implementation is **skills with scripts rather than MCP tools** (§4.1).

### 3.6 Courtesy is a feature

The docs are explicit: one machine serves both the application and the API, and there is no
rate limiting. A shared on-disk cache keyed by URL is not an optimisation, it is a
requirement — and it doubles as the provenance store and the reproducibility snapshot.

### 3.7 Reproducibility as an output

Where an analysis produces numbers, the skill emits the notebook or script that produced
them alongside the figure. The researcher can re-run it, a reviewer can check it, and the
model is visibly not the source of the statistics.

### 3.8 Fix it upstream

TLCMap owns the API and this capability both, and there is exactly **one deployment** —
tlcmap.org. So where the skills need something the API does not do, the API changes.

This design therefore contains **no workarounds, no fallbacks and no feature detection**. A
client-side workaround would be code written, tested, documented and maintained indefinitely,
to defend against behaviour we control, on behalf of an installed base of one — while every
other TLCMap client keeps hitting the same wall. Layer discovery by region is the clearest
case: either a 2,118-request harvest plus a recursive bbox-subdivision algorithm in every
client, or one `GROUP BY dataset_id` on the server that fixes the browser interface at the
same time.

§8 is accordingly a list of **prerequisites**, not of gaps to route around. Each item names
the skills that depend on it and the phase it must land by. The skills are written against
the API as it will be, and a skill whose prerequisite has not landed simply does not ship
yet — it does not ship with a workaround.

One boundary this does not cross: §8.5 sets out what genuinely belongs in the client.
Judgement, disambiguation and ethics do not become endpoints.

---

## 4. Architecture

### 4.1 Skills now, MCP later — and eventually both

The brief asks whether MCP is the right approach. The recommendation is **skills first, MCP
at Phase 3, both permanently** — not skills instead of MCP. Four reasons the order runs that
way:

1. **The hard part is knowledge, not plumbing.** Calling `GET /places` is trivial. Knowing
   that `fuzzyname` beats `name`, that a dated search silently discards undated gazetteer
   records, and that `limit` samples randomly is what separates a correct answer from a
   confident wrong one. That knowledge lives naturally in a skill's instructions; an MCP
   tool description is the wrong shape and the wrong length for it.
2. **These are pipelines, not calls.** "Map the places in this diary" is nine steps with
   branching, a review bucket and a human checkpoint. MCP exposes tools; it does not express
   a workflow. A skill does.
3. **Context economy.** An MCP tool result returns through the model. A 2.7 MB catalogue or
   a 5,000-feature collection cannot. Scripts write to disk and report a summary — the
   difference between a feasible and an infeasible session.
4. **Reproducibility.** A script committed to a repository can be re-run by a researcher
   without an agent, cited in a methods section, and reviewed. A tool call cannot.

**Where MCP earns its place — and, given that TLCMap will host this, sooner than first
assumed:**

- **Reach.** Skills run in Claude Code and the Claude apps. An MCP server reaches Claude
  Desktop, a chatbot on tlcmap.org, and any other agent framework. If TLCMap owns the
  integration, that reach is most of the point.
- **Ownership and versioning.** TLCMap versions and deploys a server it controls, rather than
  depending on what a user happens to have installed.
- **Auth, when the write API lands.** A server is the clean place to hold a credential and
  enforce scopes. A script asking a researcher to paste a token into a shell is worse in
  every way.

The important correction: **a read-only MCP server does not need to wait for the write API.**
TLCMap's read endpoints need no authentication at all, so a hosted server is deployable as
soon as the toolkit is stable. Auth is a reason MCP becomes *necessary*; it is not a
precondition for MCP being *useful*.

The two remain complementary, not alternatives: **MCP supplies tools, skills supply
judgement.** Neither replaces the other. A tool cannot carry the nine-step pipeline of
"map the places in this diary", and a skill cannot serve a chatbot on the TLCMap website.
Both are clients of the same package (§6) — the same code, two front ends, no rewrite.

**MCP design rules, for when we build it:**

- Tools return *handles and summaries*, never bulk data. `tlcmap_search` returns
  `{count, extent, date_range, sources, resource_uri}`; the features live behind an MCP
  resource the client fetches only if it needs them. This is §3.5 restated for a protocol
  where everything returns through the model.
- Tool descriptions carry the traps that fit — "never use `limit` to truncate" belongs in a
  description; the full gotcha table does not. What does not fit is exactly what justifies
  the skills continuing to exist.
- The server surfaces the same provenance the scripts do: every tool result names the query
  URL and fetch time, so an MCP-driven answer is as retraceable as a skill-driven one.
- Read and write tools ship as separate scopes from the start, so a public read-only
  deployment and an authenticated one are the same server configured differently.

### 4.2 Packaging

A **plugin with a shared library** — the ordinary skills-with-scripts pattern, which is what
Claude Code, Codex and other skill-running clients already support. No packaging step, no
install step, nothing published.

```
tlcmap-skills/
├─ .claude-plugin/plugin.json
├─ skills/
│  ├─ tlcmap-search/SKILL.md
│  ├─ tlcmap-resolve/SKILL.md
│  ├─ tlcmap-geoparse/SKILL.md
│  ├─ tlcmap-analyse/SKILL.md
│  ├─ tlcmap-visualise/SKILL.md
│  ├─ tlcmap-prepare/SKILL.md
│  └─ tlcmap-cite/SKILL.md
│
├─ lib/tlcmap/          # the shared implementation (§6), imported by the scripts below
├─ scripts/             # thin CLI entry points the skills invoke
│  ├─ search.py         #   each carries PEP 723 inline dependency metadata
│  ├─ harvest.py
│  ├─ analyse.py
│  └─ …
├─ tests/               # unit tests + the production compatibility check
│
├─ reference/           # cheatsheets the skills load on demand
│  ├─ api-quickref.md
│  ├─ gotchas.md
│  ├─ data-model.md
│  └─ views.md
│
├─ evals/               # gold sets and skill evals (§10)
└─ examples/            # the worked demonstrations (§9)
```

**Why `lib/` rather than scripts per skill.** Seven skills would otherwise each carry their
own copy of the client, and therefore their own copy of the `/maxpaging` detection, the
missing-`features` check, the `warnnig` misspelling and the date parser. One of them would
drift, and the failure would be silent — which is exactly the failure mode §3.4 exists to
prevent. Shared code is the cheap fix, and it costs nothing structurally.

**Dependencies, without an install step.** Each script declares its own dependencies with
PEP 723 inline metadata, so `uv run scripts/analyse.py` resolves them per-script into an
ephemeral environment. This matters because the heavier skills want `pandas`, `shapely`,
`rapidfuzz` and possibly `geopandas`, and nobody should have to install that stack to run
`tlcmap-search`. It also means the plugin has no setup instructions beyond installing it.

**The MCP server, at Phase 3,** is added as `servers/mcp/` in this same repository and
imports `lib/tlcmap` directly. It needs no packaging either — only a path. Publishing is not
on its critical path.

**Versioning against an unversioned API.** TLCMap's API is explicitly not versioned (§2.4),
and the skills depend on a dozen documented quirks. `tests/` therefore includes a
compatibility check run against production that fails loudly when one of them changes. When
the `udateend` bug is fixed, we should find out from a red test, not from a wrong timeline.
This is worth having regardless of how the code is distributed.

### 4.3 On publishing the library — a later, optional decision

An earlier draft of this document called for a published, `pip install`-able package from day
one. That was wrong, and the correction is worth recording because the reasoning is easy to
repeat.

Publishing is only required by one audience: **a researcher using the library in a notebook,
with no agent and without cloning this repository.** Every other consumer — the seven skills,
the MCP server, our own tests, a researcher who has cloned the repo — reaches `lib/` by path.

So the question is not "should the toolkit be a package" but "is agentless notebook use a
real audience we intend to serve?" That is a product question, it can be answered any time,
and answering it late costs nothing **provided one discipline holds now**:

> `lib/tlcmap` contains no agent-specific code — no prompts, no model calls, no assumptions
> about who is calling it.

That constraint is worth keeping on its own merits (it is what lets the MCP server reuse the
code, and what makes the analyses reproducible without a model), and it happens to leave
extraction to a package as a half-day's work rather than a refactor. Preserve the option;
don't pay for it up front.

### 4.4 How a skill is shaped

Each skill follows the same internal structure, which keeps them predictable and keeps
token cost low:

1. **Trigger and scope** — in the frontmatter description, tuned for reliable activation.
2. **Decide** — a short decision procedure the model follows to turn the request into a
   plan (which endpoint, which parameters, what the traps are here).
3. **Execute** — call a bundled script. The script does the network, the validation and the
   computation, and writes to a working directory.
4. **Check** — read the script's summary, confirm it answers the question, report the
   exclusions (how many records were dropped by a date filter, how many matches were
   ambiguous).
5. **Hand off** — name the artefacts, the attribution block, and the next skill if there is
   one.

Progressive disclosure matters: `SKILL.md` stays short and loads `reference/gotchas.md` only
when the task touches one.

---

## 5. The skill set

Seven skills. Each has a distinct trigger, and they compose.

```
                    ┌─────────────────┐
   NL request ─────▶│  tlcmap-search  │────┐
                    └─────────────────┘    │
   text corpus ────▶│ tlcmap-geoparse │────┤
                    └────────┬────────┘    │
   spreadsheet ────▶│ tlcmap-resolve │◀────┘   (resolve is geoparse's engine)
                    └────────┬────────┘
                             ▼
                    ┌─────────────────┐   ┌──────────────────┐
                    │ tlcmap-analyse  │──▶│ tlcmap-visualise │
                    └─────────────────┘   └──────────────────┘
                             │
                             ▼
                    ┌─────────────────┐   ┌──────────────┐
                    │ tlcmap-prepare  │──▶│  tlcmap-cite │  (gate on every export)
                    └─────────────────┘   └──────────────┘
```

---

### 5.1 `tlcmap-search` — find and retrieve places and layers

**Triggers.** "Find TLCMap records about…", "what layers cover the Hunter Valley", "get me
layer 152", "search the gazetteer for…".

**What it does.** Turns a natural-language request into a correct query, runs it, caches the
result, and reports what it found *and what it excluded*.

The decision procedure it encodes:

- **Which name parameter.** `containsname` by default; `name` only for a known exact title;
  `fuzzyname` when spelling is uncertain — which, for colonial-era sources, is usually.
- **Which sources.** Gazetteers for authoritative placenames, contributed layers for
  research data, `searchgeocoder` for text-derived places (available via the API but not the
  browser interface).
- **Which endpoint.** `/places` for a bounded result set; `/api` for harvesting — with the
  trade-off stated plainly, because `/api` cannot see contributed layers.
- **Region.** bbox or polygon, longitude first, ring closed, antimeridian handled.
- **Result size.** `/api` pages through any number of matches across all sources (§8.1 ③), and
  `count=true` says how many there are before committing to the fetch (④). Large results are a
  question to put to the researcher, not an obstacle to route around. **Never** `limit`.
- **Layer discovery.** One faceted catalogue request — keyword, region, date, record type
  (§8.1 ①).

**Outputs.** `records.geojson` (cached, checksummed), `query.json` (the canonical
`metadata.url`, re-runnable), `summary.md` (counts, extent, date range, source breakdown,
layers touched, records excluded and why), and an attribution block from `tlcmap-cite`.

**Covers use cases** 3 (retrieval half), 4 (finding the right contributed layers), 5
(harvest).

---

### 5.2 `tlcmap-resolve` — placenames to TLCMap records

**Triggers.** A spreadsheet of events with place strings and no coordinates; a list of
toponyms; "geocode these historical Australian places"; "which TLCMap record is this?"

This is the skill that most clearly needs a language model, and the one where the
determinism boundary matters most.

**Flow.**

1. **Normalise** the input column: strip qualifiers, expand abbreviations, note the row's
   other evidence (year, colony, nearby features, event type).
2. **Generate candidates** from TLCMap, cascading `name` → `containsname` → `fuzzyname`,
   with priors applied as filters: state, LGA, feature term, bbox, date overlap. Fuzzy
   string pre-ranking (rapidfuzz) narrows the field; it does not decide.
3. **Adjudicate.** For each ambiguous row, the model is shown the row's full context and the
   candidate records — *and nothing else* — and returns a structured verdict:
   `{uid | "unknown", confidence, reasoning, evidence_used}`. It may always answer
   "unknown"; that is a correct answer, not a failure.
4. **Band.** High confidence → accepted. Medium → accepted with a flag. Low or unknown →
   the review bucket. The thresholds are configurable and reported.
5. **Attach and export.** Coordinates are **copied from the TLCMap record**, never generated.
   A post-export validator re-reads the cache and fails the run if any coordinate does not
   match a fetched record exactly.

**Outputs.** Enriched CSV, GeoJSON, `needs-review.html` (a reviewer page with each uncertain
row, its candidates, map thumbnails and one-click accept/reject), and `decisions.jsonl` —
the full audit log of every model judgement and the evidence behind it.

**Covers use cases** 2 (entirely), 1 (the resolution half), 6 (place resolution stage).

---

### 5.3 `tlcmap-geoparse` — map the places mentioned in a document

**Triggers.** "Map every place mentioned in this diary/journal/newspaper corpus", a folder of
OCR text, a Trove export.

**Honest framing, built into the skill:** TLCMap already geoparses uploaded texts, and for a
single moderate document *that route is better* — it produces a proper text layer with
character offsets and the Full Text view for free. This skill exists for what that cannot
do: large or private corpora, non-standard formats, custom entity types, OCR that needs
repair, and control over the disambiguation. The skill says so, and offers the upload route
first when it fits.

**Flow.** Ingest and normalise (PDF/OCR/Trove XML) → sentence segmentation → NER (spaCy or
stanza; a model pass for degraded OCR where a statistical tagger fails) → a mention table
with context windows → hand off to `tlcmap-resolve` → re-attach character offsets and
mention frequency.

**Outputs.** GeoJSON with per-place mention counts and source passages; per-decade layers; a
`textcontexts`-shaped sidecar so the result can be fed to the TLCMap **Full Text view**; a
mention-level CSV; and a layer file ready for upload via `tlcmap-prepare`.

**Covers use case** 1.

---

### 5.4 `tlcmap-analyse` — spatial and temporal distribution

**Triggers.** "Analyse the distribution of…", "cluster these places", "how does this change
over time", "compare these two layers".

**Server-side or local?** The skill knows both and chooses:

| Use the API | Compute locally |
| --- | --- |
| DBScan / KMeans on a public layer | Anything on a search result or a derived set |
| Temporal clustering | The actual basic statistics (the endpoint returns geometry only) |
| Closeness between two public layers | Nearest-neighbour, KDE, Ripley's K, per-decade counts |
| Convex hull / centroid geometry | Breakdowns by state, LGA, feature term, layer |

It carries the analysis traps: DBScan's `distance` is degrees ÷ 100 and is presented to the
user as an approximate kilometre figure with the latitude caveat; KMeans needs
`withinRadius=` present but empty; temporal clustering drops records without a start date;
closeness analysis is a cross join and will time out on large layers, so it warns before
running and offers a local alternative.

**Outputs.** `stats.json` (every number, named), figures (following the `dataviz` skill's
design guidance), a **reproducible notebook** that recomputes everything, and a prose
summary drafted strictly from `stats.json` — with the exclusion counts stated, because "216
records, 55 of them dated" is the honest version of a temporal claim.

**Covers use cases** 5, 6 (analysis stage), 3 (the consolidation).

---

### 5.5 `tlcmap-visualise` — maps, timelines, charts and embeds

**Triggers.** "Make a map of…", "put this on a timeline", "give me something I can embed",
"build a storymap".

Two routes, and the skill picks by what the user will do with it:

**A. TLCMap Views** — when the data is, or can be, a TLCMap feed. The skill matches view to
data: 3D for general points, Cluster for density, Journey for `LineString` features,
Timeline for `udatestart`/`udateend`, Werekata for an ordered flight, Full Text for text
layers. It handles the traps: the `load` parameter must be percent-encoded or the feed's own
query string is swallowed; `/download` variants have no CORS headers and will not load; a
timeline built from a *search* feed shows every place as an instant because of the
`udateend` bug, so a layer feed is used instead.

**B. Local artefacts** — when the data is derived, private or needs styling TLCMap Views
cannot do: a self-contained Leaflet or kepler.gl HTML page, matplotlib/plotly figures, a
QGIS project file with layers and styling, KML/GPX for field devices, and a static storymap
scaffold (Astro/Hugo) with waypoints, primary-source excerpts, comprehension questions and
alt text.

Every output carries the attribution block. Nothing is published anywhere without the user
asking.

**Covers use cases** 3 (field outputs), 4 (publication figure), 8 (entirely).

---

### 5.6 `tlcmap-prepare` — build and validate a TLCMap-ready layer

**Triggers.** "Prepare this for TLCMap", "check my layer file", "why did my upload fail",
"update the records in my layer".

This one is as much for TLCMap's own contributor community as for the pipelines, and it is
the cheapest large win in the set — the import is strict and its failures are opaque.

**Validates.**

- **Dates** against all eight accepted forms, including `-YYYY` for BCE and the `YYYY-00-00`
  zeroed forms. **One bad date aborts the entire file**, so every row is checked and every
  failure is listed at once rather than one per attempt.
- **Headings**, with a before/after preview of import sanitisation: `Catalogue no. 3` →
  `Catalogue no ` with a significant trailing space, `Area m2` → `Area m`. Collisions after
  sanitisation are flagged as errors.
- **Collisions with built-in fields** — a column named `description` or `source` will
  silently replace the built-in one in GeoJSON output.
- **Title present** on every row (the only required field).
- **Coordinates** present, in range, plausible for the stated state, not obviously
  transposed, not in the ocean when the feature term says otherwise.
- **Record type** against the nine valid values.
- **`ghap_id` round-trip** for updates, with a dry-run diff against the current layer: rows
  added, changed (field by field), unchanged, and orphaned.
- **Metadata completeness** — licence, citation, creator, and a prompt for `warning` where
  the content suggests one is needed.

**Outputs.** An upload-ready layer CSV in exactly the export format TLCMap accepts back, a
validation report, the dry-run diff, and — until the write API exists — a short upload
checklist naming the browser steps. Once it exists, this skill gains a `push`.

**Covers use case** 7 (the preparation half), and supports 1, 2 and 3.

---

### 5.7 `tlcmap-cite` — attribution, licensing and citable packaging

**Triggers.** "Who do I need to credit", "can I republish this layer", "package this for
deposit", "generate a data availability statement". Also invoked automatically by every
export path in the other six skills.

**What it does.** Collects `?metadata` for every layer touched — the cheap call that returns
metadata without building features — and assembles an attribution block: creator, publisher,
citation, licence, rights, DOI, source URL, and **`warning` reproduced verbatim and
prominently**.

It states what the licence field says. It does not decide what the licence permits. Where a
layer has no licence — 1,414 of the 2,118 public layers — it says so, and says that absence
of a licence is not permission.

**Ethics routing.** Where a layer's metadata, keywords or warning indicate Indigenous
knowledge, cultural material or sensitive sites, the skill stops and surfaces the question
to the user rather than proceeding. See §11.

**Packaging.** Fetches RO-Crates for a frozen, citable snapshot; assembles a crate for a
derived dataset; drafts a data availability statement and a methods paragraph describing
exactly which queries were run and when.

**Covers use case** 4 (the rights half), and gates all seven.

---

## 6. The shared toolkit

Plain Python in `lib/tlcmap/`, with **no agent-specific code in it at all** — no prompts, no
model calls, no assumptions about who is calling. The skills' scripts import it, the MCP
server will import it, and it runs perfectly well in a notebook by anyone who has the repo.

That one constraint is what matters; it is not a claim about how the code is distributed
(§4.3). It is what lets the same implementation serve several front ends, what makes the
analyses reproducible without a model in the loop, and what would make extracting a package
later a half-day's work if that turns out to be wanted.

### 6.1 Modules

| Module | Responsibility |
| --- | --- |
| `client.py` | HTTP with caching, retry and courtesy delay. Maps the API's status codes to typed errors (requires §8.1 ②). No content-type sniffing — a `413`, `403` or `400` says what happened. |
| `query.py` | Safe parameter construction. Validates bbox longitude order and closes polygon rings before sending. A malformed `extended_data` expression comes back as a `400` naming it (§8.1 ②), so the client does not second-guess the server. |
| `model.py` | One canonical `Record` regardless of which naming convention the response used (GeoJSON `id` / layer CSV `ghap_id` / analysis output). Reads extended data from its own namespace (§8.2 ⑤) rather than inferring it. Parses all eight date forms including BCE. |
| `catalogue.py` | Layer discovery in one faceted request — keyword, creator, licence, region, date (requires §8.1 ①). Treats any contributor-declared extent as a hint, never as fact. |
| `resolve.py` | Candidate generation, prior application, fuzzy pre-ranking, evidence packaging for adjudication, confidence banding, review-bucket routing. |
| `stats.py` | Spatial and temporal statistics computed locally: hull, centroid, nearest-neighbour, KDE, Ripley's K, per-period counts, categorical breakdowns. |
| `export.py` | GeoJSON with TLCMap `display` configuration, upload-format CSV, KML, GPX, QGIS project, RO-Crate bundle. Runs the coordinate-provenance validator before writing. |
| `views.py` | Builds TLCMap Views URLs with correct percent-encoding, and picks the view the data can actually support. |
| `attribution.py` | Layer metadata collection, attribution block rendering, warning propagation, sensitivity flagging. |
| `provenance.py` | The manifest: every URL fetched, when, with what checksum; every model judgement with its evidence. |

### 6.2 What the toolkit deliberately does not contain

Earlier drafts had the toolkit carrying a layer extent index — a harvest of all 2,118 layer
feeds — plus recursive bbox subdivision to stay under the result ceiling, content-type
sniffing to catch HTML redirects, a run-the-query-twice check to detect discarded filters,
and re-derivation of `udateend` from `dateend`.

**None of it is being built.** Each was a workaround for something §8 fixes at the source
(§3.8), and with one deployment there is nobody to carry them for. They are recorded here
only so the decision reads as deliberate rather than forgotten.

The corresponding prerequisites are §8.1 ①–④ and §8.2 ⑤–⑦.

---

## 7. Use cases mapped

| # | Use case from the brief | Skills | Gaps |
| --- | --- | --- | --- |
| 1 | Historical text → mapped corpus | geoparse → resolve → visualise → prepare | Upload is manual |
| 2 | Spreadsheet enrichment | resolve → cite | Fully supported today |
| 3 | Regional fieldwork brief | search → analyse → visualise → cite | Needs the extent index |
| 4 | Indigenous / colonial co-mapping | search → cite → visualise | Licence is free text; **human gate required** |
| 5 | Toponym pattern analysis | search (`/api` harvest) → analyse → visualise | Fully supported today |
| 6 | Environmental / event history overlay | geoparse → resolve → analyse → visualise | External data sources are the researcher's |
| 7 | Longitudinal managed layer | prepare → *(write API)* → visualise | **Blocked**: no write, no change feed |
| 8 | Teaching storymaps | search → visualise | Fully supported today |

Five of eight are fully deliverable against today's read-only API. One is partly blocked, one
needs a human gate by design, and one needs the extent index we are building anyway.

### Additional capabilities worth including

These are not in the brief but are cheap, high-value, and directly serve TLCMap's community.

- **Layer health report** *(in `tlcmap-prepare`)* — point it at any public layer and get a
  data-quality assessment: unparseable dates, implausible coordinates, duplicate records,
  headings that were mangled on import, missing licence or citation. Useful to contributors,
  and an excellent low-risk first demonstration.
- **Cross-layer duplicate detection** *(in `tlcmap-analyse`)* — when composing a multilayer,
  find records that are the same place in different layers, so a merged map does not
  triple-count. Closeness analysis plus local matching.
- **Resolution evaluation harness** *(in `evals/`)* — a gold set of hand-checked
  placename→UID pairs, so we can state a precision/recall figure for `tlcmap-resolve`
  instead of asserting it works. Research credibility depends on this, and it is what makes
  the proof of concept a *proof*.
- **Reproducible notebook emission** *(in `tlcmap-analyse`)* — already described, but worth
  calling out as a deliverable in its own right: the skill's output includes the means to
  verify the skill.
- **Saved-search awareness** — a TLCMap saved search is a stored query, not a stored result.
  Skills should prefer emitting a re-runnable query URL over a frozen extract wherever the
  research question is ongoing.

---

## 8. API enhancements — the platform half of this project

TLCMap owns both the API and this capability, and runs exactly one deployment. So these are
**prerequisites for the skills**, not gaps the skills will route around (§3.8) — a work
programme on the critical path rather than a wishlist.

Each item names the skills that depend on it and the phase it is needed by. §8.6 summarises
that as a schedule.

### 8.1 Tier 1 — required by Phase 1

**① Layer extent and facets on the catalogue.**

```
GET /layers/json?bbox=&datefrom=&dateto=&q=&recordtype=&page=&per_page=
```

with computed `bbox`, `date_range` and `record_count` on every entry. One PostGIS query
(`ST_Extent` and `min`/`max` over dates, grouped by `dataset_id`), refreshed on layer change.

*Computed, not declared* — the contributor-filled fields are populated on 4 of 2,118 layers
and two of those four are malformed (§2.2). Keep them as a separate contributor hint; do not
promote them to the facet.

> **Required by:** `tlcmap-search` (layer discovery), `tlcmap-cite` (finding layers by rights
> status), use cases 3 and 4. Also fixes the same discovery gap in the browser interface.
> **The single highest-value change.** Needed by Phase 1.

**② Honest errors instead of HTML redirects.**

Today every failure mode returns something a client must sniff for:

| Situation | Now | Should be |
| --- | --- | --- |
| Over the 5,000 ceiling | `302` → `/maxpaging` HTML | `413` + JSON `{error, total, suggestions}` |
| Layer private | `200`, no `features`, key `warnnig` | `403` + JSON |
| Layer missing | `200`, no `features` | `404` + JSON |
| Unparseable `extended_data` | `200`, filter silently dropped | `400` + JSON naming the expression |
| Missing analysis parameter | `500` | `400` + JSON naming the parameter |
| Bad `format` on `/api` | `302` → home page | `400` + JSON |
| `id=` on `/places` | `302` → the path form | Serve it directly |

The `extended_data` case is the most dangerous thing in the API, and it is worth stating
precisely because it is easy to underestimate. Layer 461 holds 17,917 records:

```
extended_data=Years > 50    →    612 records      (filter applied)
extended_data=Years>50      →  17,917 records     (filter silently discarded)
```

A single missing space silently returns 29× the data as though it were a filtered result.
Here it happens to exceed the ceiling and fail loudly-but-wrongly as HTML; on any layer under
5,000 records it would return a plausible, complete, entirely unfiltered answer that no
client could distinguish from a correct one.

There is a second-order trap in the same table. `id=` redirects to the path form, so a client
*must* follow redirects — but following redirects on an over-ceiling query hands back
`/maxpaging` HTML with a `200`. The API currently requires clients to both follow and not
follow redirects.

> **Required by:** every skill, because it is the difference between `client.py` reading a
> status code and `client.py` guessing from content types. Needed by Phase 1.

**③ Extend `/api` to contributed layers.**

`/api` pages properly through any number of matches and reports `total`. It serves the two
gazetteers only. Contributed layers — the research data — are reachable only through
`/places`, which is the endpoint with the ceiling.

> **Required by:** `tlcmap-search` and `tlcmap-analyse` for any question spanning contributed
> layers — which is most research questions. Probably the best value-to-effort ratio on the
> list, since the paging machinery already exists. Needed by Phase 1.

**④ A cheap count.**

`GET /places?…&count=true` returning `{total}` without building features — or simply putting
`total` in `/places` output the way `/api` already does.

Today a client cannot ask how big a result set is without running the query, and running it
is exactly what fails when it is large: `paging=1` on a 94,615-match query still `302`s,
because the ceiling is checked against the total, not the page.

> **Required by:** `tlcmap-search`, to answer "this will return 94,615 records, shall I narrow
> it?" before spending the request. Needed by Phase 1.

### 8.2 Tier 2 — data correctness

**⑤ Namespace extended data in GeoJSON.** Return it under `properties.extended_data{}` rather
than merged into `properties`, where a contributor's column named `description` or `source`
silently overwrites the built-in field and nothing in the output says which is which.

> **Required by:** `model.py`, and therefore everything downstream of it — without it there is
> no reliable way to tell a contributor's field from a defined one. Note this one changes an
> existing response shape, so it needs a transition for current clients. Needed by Phase 1.

**⑥ Fix `udateend` in search output.** It is computed from the start date, so every record
reports `udateend == udatestart` and every timeline built from a search shows instants.

> **Required by:** `tlcmap-visualise` (timelines built from searches) and `tlcmap-analyse`
> (any temporal claim over search results). Needed by Phase 1.

**⑦ Stop `sort` and date filters silently discarding records.** `sort` drops every record
with neither start nor end date — 216 results become 55. Dated searches exclude undated
records entirely. Both are defensible defaults and neither is escapable.

Add `nulls=last` to `sort`, and `include_undated=true` to date filtering.

> **Required by:** `tlcmap-search` and `tlcmap-analyse`. Without it, honest temporal analysis
> means fetching everything and sorting locally, which is impossible above the ceiling.
> Needed by Phase 1.

**⑧ Make `limit` deterministic; add `sample`.** `limit` currently does `shuffle()` then
`take()`, so identical requests return different records. Make `limit=N` take the first N,
and move the existing behaviour to `sample=N`. Both uses are legitimate; one parameter
should not silently mean the surprising one.

**⑨ Return the match score on `fuzzyname`.** The trigram similarity is computed to rank
results and then discarded. Exposing it as a property, and allowing a threshold, would give
`tlcmap-resolve` a server-computed signal it currently cannot obtain at any price.

> **Required by:** `tlcmap-resolve`. **This is the one item that cannot be obtained any other
> way** — the score is computed and discarded, and no client can recover it. Needed by
> Phase 2.

### 8.3 Tier 3 — friction

**⑩ Bulk fetch by ID.** `GET /places?ids=a1353c,n77b93,tcfe93&format=json`. Today `id=` takes
one, and comma-separated values redirect to a nonsense path. The resolver needs N records for
N candidates and must make N requests.

**⑪ Controlled vocabulary endpoints.** `state`, `lga`, `feature_term` and `parish` are
**exact** matches against the gazetteer's own vocabulary — `lga=CESSNOCK` works,
`lga=Cessnock Council` does not — and nothing exposes the valid values.

```
GET /vocabularies/{feature_term|state|lga|parish}/json
```

> **Required by:** `tlcmap-search` and `tlcmap-resolve`, which otherwise cannot construct a
> valid `state`, `lga` or `feature_term` filter at all. Needed by Phase 2.

**⑫ Statistics as JSON.** `basicstatistics/json` returns geometry only — hull, centroid, box
— while the actual numbers (count, area, density, date range, median, mean) are computed for
the HTML page and thrown away. Advanced statistics has no JSON endpoint at all.

> **Required by:** `tlcmap-analyse`, which otherwise recomputes locally what the server
> already calculates for its own HTML page. Needed by Phase 1.

**⑬ DBScan distance in real units.** `distance` is labelled kilometres, divided by 100, and
passed to PostGIS as degrees on a geometry — so its meaning changes with latitude and no
caller can reason about it. Accept metres against geography.

**⑭ Conditional requests and a change feed.** `ETag` / `Last-Modified` on feeds, and
`?updated_since=` on layers and the catalogue.

> **Required by:** use case 7's scheduled pipeline, which otherwise refetches and diffs
> everything on every run. Needed by Phase 5.

**⑮ A structured licence identifier** alongside the free-text field — SPDX or a CC code. The
free text stays and is still what gets displayed; the identifier lets a client *assist* a
republication decision without ever making it (§3.3).

**⑯ Import fixes.** Stop stripping digits from field headings (`Area m2` → `Area m`). Report
every date error rather than aborting on the first. Expose the validator as
`POST /layers/{id}/validate` so a client can dry-run before committing.

**⑰ Small defects.** `warnnig` → `warning`; the RO-Crate metadata declaring
`TLCMLayer_{id}.json` while shipping `tlcmap_output.json`; `chunks` returning `500`.

**⑱ Published rate limits**, so clients have something to respect other than good manners —
more pressing once a public MCP endpoint (§9, Phase 3) exists.

### 8.4 Tier 4 — the write API

Unchanged from the earlier draft, and still the precondition for use case 7.

```
POST   /api/v1/layers                          create, with metadata
PATCH  /api/v1/layers/{id}                     update metadata
POST   /api/v1/layers/{id}/records             bulk append (CSV or GeoJSON body)
POST   /api/v1/layers/{id}/records:upsert      upsert on a client-supplied external key
PUT    /api/v1/layers/{id}/records/{ghap_id}   update one record
DELETE /api/v1/layers/{id}/records/{ghap_id}
POST   /api/v1/layers/{id}/validate            dry run — the same report, server-side
POST   /api/v1/multilayers                     compose
```

What matters more than the routes: **scoped personal access tokens** (`layer:read`,
`layer:write`), revocable and not the session cookie; **idempotency** via an
`Idempotency-Key` header and upsert on a client-supplied key, without which use case 7's cron
duplicates the layer on every failed run; **a validation dry run** returning the same errors
as a real import; **all errors at once**, not the first; and **provenance preservation**, so
extended-data fields written by a pipeline survive round-trips unmodified.

### 8.5 What legitimately stays in the skills

The point of §8 is not that everything belongs in the API. These are genuinely the agent's
work, and no API change would or should absorb them:

| Stays client-side | Why |
| --- | --- |
| Turning a research question into a query | Judgement, not a parameter |
| Choosing among `name` / `containsname` / `fuzzyname` | Depends on what the researcher knows about their own data |
| LLM disambiguation of candidates | The candidates come from the API; the choice does not |
| Confidence banding and review-bucket routing | A research-workflow decision |
| Coordinate provenance validation (§3.1) | Guards against the *model*, not the API |
| Attribution presentation and ethics gating | Requires human judgement by design (§3.3, §11) |
| Reproducible notebook emission | An output format, not a data source |
| Caching | Helped by ⑭, but still ours to do |
| Longitude-first `bbox` | Correct GeoJSON convention, not a defect |
| The date formats themselves | A genuine feature — mixed `1856` / `1856-03` / `-400` is right for historical data |

### 8.6 Prerequisite summary

What must land before each phase can ship. There is one deployment, so "landed" means
deployed to tlcmap.org — there is no capability negotiation and no fallback path (§3.8).

| Phase | Prerequisites | Without them |
| --- | --- | --- |
| **1** — search, analyse, visualise, cite | ① catalogue facet · ② honest errors · ③ `/api` over contributed layers · ④ cheap count · ⑤ namespaced extended data · ⑥ `udateend` fix · ⑦ `nulls=last` / `include_undated` · ⑫ statistics as JSON | Regional discovery, honest temporal analysis and reliable field identification are all unavailable. Phase 1 does not ship. |
| **2** — resolve, prepare | ⑨ match score · ⑩ bulk fetch by ID · ⑪ vocabulary endpoints · ⑯ import fixes and validation dry run | The resolver loses its one server-side signal, and `tlcmap-prepare` cannot validate before upload |
| **3** — MCP server | ⑱ published rate limits | A public endpoint with no stated limit |
| **4** — geoparse, narrative | *(none beyond Phase 2)* | — |
| **5** — write | ⑭ conditional requests · §8.4 write API | Use case 7 stays blocked |

⑧ (deterministic `limit`), ⑬ (DBScan units), ⑮ (structured licence) and ⑰ (small defects)
are not blocking, and can land whenever convenient.

The compatibility suite in `tests/` asserts each of these behaviours against production, so a
regression in the API shows up as a failing test rather than as a wrong map. Since the API is
unversioned (§2.4), that suite is the only thing standing between a silent platform change
and a silently wrong research output.

---

## 9. Plan

Scope, split and audience are settled (see **Decisions taken**). The sequence below reflects
them: broad Phase 1, seven skills, and MCP pulled forward out of the tail.

Two tracks run in parallel and are **equally first-class**: the **platform track** (§8) and
the **capability track** (toolkit, skills, server). Per §3.8 the platform track is not a
follow-up and not optional — it is on the critical path, because the skills are written
against the API as it will be and ship when their prerequisites do (§8.6).

**The platform track leads.** Each phase below names what must be deployed to tlcmap.org
first. In practice the two can be built concurrently — the API changes are small and mostly
independent — but a skill is not finished until its prerequisite is live, and none of them
will be given a temporary implementation in the meantime.

### Phase 0 — Agree the design *(this document)*

Remaining: the demonstration dataset, the NER stack, and whether §8.4's write API is on the
TLCMap roadmap. See §12.

### Phase 1 — Core toolkit and the retrieval-to-visualisation path

`lib/tlcmap` core (`client`, `query`, `model`, `catalogue`, `provenance`, `export`, `views`)
with its test suite and the production compatibility check, plus `tlcmap-search`,
`tlcmap-analyse`, `tlcmap-visualise`, `tlcmap-cite`.

Phase 1 ends with a plugin someone can install and use. Nothing needs publishing, and no
setup step beyond installing the plugin — the scripts carry their own dependencies (§4.2).

*Platform prerequisites (§8.6):* ① catalogue facet, ② honest errors, ③ `/api` over contributed
layers, ④ cheap count, ⑤ namespaced extended data, ⑥ `udateend` fix, ⑦ `nulls=last` /
`include_undated`, ⑫ statistics as JSON. These are deployed to tlcmap.org before the phase is
complete; the skills are written against them.

**Demonstration 1 — the brief's own sentence, end to end.** *"Find records in this TLCMap
dataset relating to a particular subject, period or region, analyse their spatial and
temporal distribution, and produce an appropriate visualisation."*

One natural-language request produces: the query, a cached and checksummed result, a
statistics file, a timeline and a distribution map, a reproducible notebook, an attribution
block, and prose that cites its own numbers — including how many records the date filter
excluded. That last detail is the demonstration's real point: it shows the system is honest
about what it does not know.

**Demonstration 1b — layer health report.** Point it at a public layer, get a data-quality
assessment. Small, immediately useful to TLCMap's contributors, and a good hedge.

### Phase 2 — Resolution and round-trip

`resolve.py`, `attribution.py`, plus `tlcmap-resolve` and `tlcmap-prepare`. Build the
evaluation gold set alongside, not after.

**Demonstration 2 — enrich and return.** A spreadsheet of undated, uncoordinated historical
events becomes an enriched, citable geodataset with a reviewer page for the uncertain
matches and an upload-ready layer file — with a stated accuracy figure from the gold set.
This is the demonstration that shows what the model adds beyond a script.

### Phase 3 — Read-only MCP server

Brought forward from the tail, because TLCMap is hosting and the read API needs no auth.
A thin wrapper over the Phase 1–2 package: summary-and-handle tools, resources for bulk
data, provenance in every result, and read/write scopes separated from the start even though
only read ships.

**Demonstration 3 — the same question, three front ends.** The Phase 1 research question
answered through a skill in Claude Code, through the MCP server in Claude Desktop, and
through the library in a notebook — same code, same numbers, same provenance. For a hosted
capability this is the demonstration that matters most: it shows TLCMap is building one
thing, not three.

### Phase 4 — Text and narrative

`tlcmap-geoparse`, the fieldwork brief, the storymap scaffold. Depends on Phase 2's resolver,
independent of Phase 3.

**Demonstration 4 — a colonial diary becomes a map**, with each place linked to the sentence
it came from, viewable in TLCMap's own Full Text view.

### Phase 5 — Write

Contingent on §8.4. Adds scoped token handling to the MCP server, `push` to
`tlcmap-prepare`, and unblocks use case 7's managed layer. The server built in Phase 3 is
where the credential lives; nothing needs restructuring to get here.

### Sequencing note

**Every phase has platform prerequisites, and they gate it** (§8.6). The skills are written
against the API as it will be, so a phase completes when its API changes are deployed to
tlcmap.org — there is no version of this plan where the capability ships ahead of the
platform and catches up later.

Design and construction of both can start immediately and proceed in parallel: the API
changes are small, mostly independent of each other, and none requires the skills to exist
first. What cannot happen in parallel is *finishing* — a skill is not done until its
prerequisite is live.

The practical consequence for planning: **§8.1 Tier 1 is the long pole for the whole
project.** Four changes, all modest, and Phase 1 waits on them.

---

## 10. How we will know it works

A proof of concept that cannot be measured is a demonstration, not a proof.

- **Resolution accuracy.** A gold set of ~200 hand-checked placename→UID pairs drawn from
  real historical sources, with precision, recall and — most importantly — the *abstention
  rate*, because a resolver that says "unknown" correctly is worth more than one that
  guesses well.
- **Query correctness.** A fixture suite of natural-language requests with known-correct API
  queries, checking the model picks the right parameter and avoids the traps.
- **Skill triggering.** Eval suites (via `skill-creator`) confirming each skill activates on
  its intended requests and not on the others'.
- **Provenance integrity.** An automated check that every coordinate in every output traces
  to a cached API response. This should be impossible to fail, and tested as if it were not.
- **Courtesy.** Request counts per demonstration run, kept visible and kept low.

---

## 11. Risks and ethics

**Cultural sensitivity is the first-order risk, not a compliance footnote.** TLCMap holds
Indigenous placenames, massacre sites and mission records. 165 public layers carry an
explicit contributor warning.

- Warnings travel with data, always, including into derived outputs and figures.
- Licence and rights text is surfaced, never interpreted into a permission.
- Any pipeline touching Indigenous knowledge stops at a human decision. The skills do not
  decide; they present what the contributor said and ask.
- CARE principles (Collective benefit, Authority to control, Responsibility, Ethics) are
  stated in the skills' own instructions, not only in this document.
- Aggregation is itself a risk: combining layers can reveal sensitive site locations that
  each layer alone did not. `tlcmap-analyse` flags composition across layers carrying
  warnings.

**Fabrication.** Addressed architecturally in §3.1 and tested in §10, because instructing a
model not to invent coordinates is necessary and not sufficient.

**Silent wrong answers.** The API's failure modes return plausible data. §3.4's assertions
are the mitigation, and the gotcha table in §2.3 is the test list.

**Coverage bias.** The gazetteers are uneven, contributed layers reflect who contributed.
Analyses must report what was excluded — undated records dropped, regions with no layers —
rather than presenting a partial distribution as a complete one.

**Load.** No rate limiting means we set our own. Cache aggressively, harvest once, never
poll.

**Over-automation.** The temptation is to make the pipeline run end to end without stopping.
The review bucket and the human gates are the product, not friction in it.

---

## 12. Open questions

Three are settled — see **Decisions taken** at the top. What remains:

1. **Demonstration dataset.** Which layer or region should Demonstration 1 use? It wants good
   date coverage, an interesting spatial story, and no sensitivity complications. Worth
   choosing before Phase 1 starts, because it shapes what the skills are tuned against.
2. **Write API timeline.** Is §8.4 on TLCMap's roadmap? It sets Phase 5, and it is the only
   thing standing between use case 7 and a working managed layer.
3. **Platform track scheduling.** §8 is in-house work on the critical path (§8.6). Who picks
   it up, and when does Tier 1 land? Phase 1 ships behind it, so this is the question that
   sets the whole timeline.
4. **NER stack** — spaCy, stanza, or model-based extraction for the geoparsing skill? The
   heaviest dependency question in the project. PEP 723 per-script environments (§4.2) keep
   it off everyone who is not geoparsing, but a multi-hundred-megabyte model download is
   still a real cost to the one skill that needs it.
5. **MCP hosting.** Where does the Phase 3 server run, and does it sit behind the same
   infrastructure as the application? Relevant because §2.3 notes the API has no rate
   limiting, and a public MCP endpoint makes that more pressing.
6. **Licensing and governance of the capability itself** — the skills, the toolkit and the
   server are TLCMap-owned artefacts that third parties will install. What licence, and what
   support expectation?
7. **Does the browser interface benefit too?** The extent facet (§6.2) fixes a discovery
   problem the website has as well. Worth confirming before scoping it as agent work.
8. **Is agentless notebook use a real audience?** The only thing that would justify publishing
   `lib/tlcmap` as an installable package (§4.3). Answerable at any time, at no cost, as long
   as the no-agent-code-in-`lib` discipline holds. Nothing in Phases 1–5 depends on it.
