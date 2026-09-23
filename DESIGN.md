# TLCMap AI Skills — design and plan

**Status:** draft for discussion · **Date:** 2026-09-23 · **Phase:** design, nothing implemented

## Decisions taken

| Decision | Choice | Consequence |
| --- | --- | --- |
| Proof-of-concept scope | **Broad** — retrieval through to visualisation (Phase 1) | Proves the whole shape against today's read-only API; the resolver follows in Phase 2 |
| Skill split | **Seven skills**, as designed in §5 | Each keeps a distinct trigger; they compose |
| Audience | **A capability TLCMap itself hosts** | Reshapes §4.1, §6, §8.1 and §9 — see below |

The third decision is the one with teeth. It means:

- **The toolkit is a published package, from day one.** Not an implementation detail bundled
  inside a skill folder — a versioned, tested, installable Python library that TLCMap owns,
  documents and supports. Skills, MCP server and researchers' own notebooks are all just
  clients of it.
- **MCP moves forward, and does not wait for the write API.** TLCMap's read API needs no
  authentication, so a hosted read-only MCP server is deployable *now*. That was the
  assumption worth revisiting: MCP was previously tied to auth, but auth is a reason MCP
  becomes *necessary*, not a precondition for it being *useful*. A server at
  `mcp.tlcmap.org` reaches Claude Desktop, a chatbot on the TLCMap site, and any other agent
  framework — audiences a skill plugin cannot.
- **The extent index (§6.2) should be built in-house, not client-side.** If TLCMap is running
  this, it has the database. Computing layer extents server-side is a small query, not a
  2,118-request harvest, and it closes the gap for every client at once rather than for each
  user separately. This is now the single highest-value platform change.
- **Distribution, versioning and support become real constraints.** Third parties will install
  the plugin; the API is unversioned (§2.4) while the package will be versioned; CI, tests
  and a compatibility check against production are no longer optional.

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

**The last number is the single most consequential fact in this document.** Layer discovery
by region is impossible from the catalogue. Verified workaround: search *records* by bbox
restricted to contributed layers, then group by the `TLCMapDataset` property —

```
GET /places?format=json&bbox=151.0,-33.0,151.2,-32.8&searchpublicdatasets=on
  → features[].properties.TLCMapDataset = "https://tlcmap.org/publicdatasets/461"
```

— which works, but inherits the 5,000-record ceiling and so fails exactly where the region
is richest. A layer-extent index is therefore a first-class component of the toolkit
(§6.2), and a server-side facet is our top platform ask (§8).

### 2.3 Behaviours that will break a naive client

Every one of these is a silent failure — wrong data, not an error. The toolkit exists in
large part to absorb them.

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

### 3.4 Assert, never assume

Because the API fails silently, the client verifies rather than trusts:

- An `extended_data` filter is run twice — with and without — and the counts compared. Equal
  counts mean the expression was discarded; that is raised as an error.
- A response missing `features` is a private-or-missing layer, not an empty result.
- A response whose content type is HTML is the `/maxpaging` redirect.
- A sorted result is compared against the unsorted count, and the drop is reported.

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

Three deliverables, one repository, deliberately layered so the bottom one can outlive the
other two.

```
tlcmap-skills/
├─ packages/
│  └─ tlcmap/                      # ① the published Python package (§6)
│     ├─ src/tlcmap/               #    client, query, model, catalogue, resolve,
│     ├─ tests/                    #    stats, export, views, attribution, provenance
│     ├─ docs/
│     └─ pyproject.toml            #    versioned, installable, TLCMap-owned
│
├─ .claude-plugin/plugin.json      # ② the skill plugin
├─ skills/
│  ├─ tlcmap-search/SKILL.md
│  ├─ tlcmap-resolve/SKILL.md
│  ├─ tlcmap-geoparse/SKILL.md
│  ├─ tlcmap-analyse/SKILL.md
│  ├─ tlcmap-visualise/SKILL.md
│  ├─ tlcmap-prepare/SKILL.md
│  └─ tlcmap-cite/SKILL.md
├─ reference/                      #    cheatsheets the skills load on demand
│  ├─ api-quickref.md
│  ├─ gotchas.md
│  ├─ data-model.md
│  └─ views.md
│
├─ servers/mcp/                    # ③ the MCP server (Phase 3) — a thin wrapper
│
├─ evals/                          # gold sets and skill evals (§10)
└─ examples/                       # the worked demonstrations (§9)
```

**① The package is the asset.** Because TLCMap hosts this, the library is not an internal
detail of a skill bundle — it is a supported artefact with its own version number, test
suite and documentation. A researcher can `pip install tlcmap` and use it in a notebook with
no agent involved at all, which is both a real audience and the thing that keeps the skills
honest: anything the skills can do, the library can do, reproducibly and without a model.

**② The plugin** groups the seven skills so they install, version and update together, and
share one copy of the toolkit and reference set. Loose skills would each need their own.

**③ The MCP server** is a thin front end over ① — tool definitions, resource handling and
(later) token scopes, with no logic of its own.

**Versioning against an unversioned API.** TLCMap's API is explicitly not versioned (§2.4),
while the package will be. The package therefore pins the *behaviours* it depends on and
ships a compatibility check — a small suite run against production that fails loudly when a
documented quirk changes. When the `udateend` bug is fixed, we should find out from a test,
not from a wrong map.

### 4.3 How a skill is shaped

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
- **Ceiling strategy.** When a query exceeds 5,000 matches: narrow by region, date or
  feature term, or switch to `/api`, or read the layers directly. **Never** `limit`.
- **Layer discovery.** Catalogue keyword match over the cached 2.7 MB catalogue, plus the
  bbox-then-group-by-layer technique from §2.2, plus the extent index (§6.2).

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

A plain Python package, `packages/tlcmap/`, with **no agent-specific code in it at all** —
no prompts, no model calls, no assumptions about a caller. Skills invoke it as a script, the
MCP server imports it, and a researcher installs it and uses it in a notebook.

That constraint is deliberate and load-bearing. It is what lets the same code serve three
front ends, it is what makes the analyses reproducible without an agent, and — given TLCMap
will own and support this — it is what keeps the maintenance burden on a normal Python
library rather than on something that only works inside an assistant.

### 6.1 Modules

| Module | Responsibility |
| --- | --- |
| `client.py` | HTTP with caching, retry, courtesy delay. Detects the `/maxpaging` HTML redirect, the missing-`features` private-layer response and the `warnnig` misspelling. Raises typed errors instead of returning wrong data. |
| `query.py` | Safe parameter construction. Validates bbox longitude order, closes polygon rings, formats `extended_data` with its required spaces **and verifies the filter was actually applied** by comparing filtered and unfiltered counts. |
| `model.py` | One canonical `Record` regardless of which naming convention the response used (GeoJSON `id` / layer CSV `ghap_id` / analysis output). Separates defined fields from extended data by cross-checking the layer CSV header. Parses all eight date forms including BCE. Recomputes `udateend` from `dateend` to work around the search-output bug. |
| `catalogue.py` | Caches the 2,118-layer catalogue; keyword, creator and licence search over it; builds and maintains the extent index (§6.2). |
| `resolve.py` | Candidate generation, prior application, fuzzy pre-ranking, evidence packaging for adjudication, confidence banding, review-bucket routing. |
| `stats.py` | Spatial and temporal statistics computed locally: hull, centroid, nearest-neighbour, KDE, Ripley's K, per-period counts, categorical breakdowns. |
| `export.py` | GeoJSON with TLCMap `display` configuration, upload-format CSV, KML, GPX, QGIS project, RO-Crate bundle. Runs the coordinate-provenance validator before writing. |
| `views.py` | Builds TLCMap Views URLs with correct percent-encoding, and picks the view the data can actually support. |
| `attribution.py` | Layer metadata collection, attribution block rendering, warning propagation, sensitivity flagging. |
| `provenance.py` | The manifest: every URL fetched, when, with what checksum; every model judgement with its evidence. |

### 6.2 The layer extent index

Because only 4 of 2,118 layers declare an extent, something has to compute one. It is what
makes "which layers cover the Hunter Valley between 1820 and 1860" answerable in a single
lookup rather than through thousands of requests.

**Given that TLCMap hosts this, build it server-side.** TLCMap has the database. Computing
each layer's true bounding box, date range, record count and feature-term profile is one
query against PostGIS, refreshed when layers change — against a client-side harvest of 2,118
layer feeds that every installation would have to repeat and keep fresh independently. The
server-side version closes the gap for every client at once, including the browser interface,
which has the same discovery problem.

This is now the **single highest-value platform change** (§8.1), and it is in-house work
rather than an ask of someone else.

The toolkit still ships the client-side harvest, for two reasons: it unblocks Phase 1 before
the platform change lands, and it works against any TLCMap deployment that has not applied
it. It is polite by construction — rate limited, resumable, cached, run once rather than per
session — and it is written to be deleted when the facet exists.

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

## 8. Platform gaps and what to ask for

Ordered by how much each would improve the skills. Since TLCMap is hosting this capability,
these are not requests of another team — they are the platform half of the same project, and
items 1–4 should be scheduled alongside Phase 1 rather than after it.

### 8.1 Reads

1. **Layer extent and facets in the catalogue.** Computed bbox, date range and record count
   per layer, plus `GET /layers/json?bbox=&from=&to=&q=`. One PostGIS query behind it. This
   single change removes the need for §6.2 entirely, makes regional discovery a one-request
   operation, and fixes the same discovery problem in the browser interface. **Highest value
   by a wide margin, and now in-house work.**
2. **Statistics as JSON.** `basicstatistics/json` returns geometry only; the numbers are
   trapped in HTML. Return them.
3. **Report discarded `extended_data` expressions** instead of silently ignoring them — the
   most dangerous behaviour in the API, because it returns a plausible wrong answer.
4. **Fix `udateend` in search output** (currently copied from `udatestart`), the `warnnig`
   misspelling, the RO-Crate GeoJSON filename mismatch, and the `chunks` 500.
5. **A structured licence identifier** alongside the free-text field — SPDX or a CC code —
   so republication decisions can at least be *assisted*.
6. **Conditional requests.** `ETag` / `Last-Modified` on feeds, and an `updated_since`
   parameter. Needed for use case 7's scheduled re-runs, and it is the polite way to poll.
7. **Sort without data loss.** `sort` dropping undated records is defensible in the UI and
   surprising in an API; a `nulls=last` option would fix it.
8. **Rate limiting.** Asking clients to be polite works until it does not. A published limit
   is easier to respect than an unstated one.

### 8.2 Write API — proposed shape

Needed for use cases 1, 3 and 7, and the precondition for the MCP server.

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

Requirements that matter more than the routes:

- **Scoped personal access tokens** (`layer:read`, `layer:write`, per-layer where possible),
  revocable, not the session cookie.
- **Idempotency.** An `Idempotency-Key` header, and `upsert` on a client-supplied external
  key. Without these, use case 7's cron re-run duplicates the whole layer on every failure.
- **A validation dry run** returning the same errors as a real import, so a client can check
  before it commits — and so the import's strictness stops being opaque.
- **Partial import, or at minimum a full error list.** Today one bad date aborts the file and
  reports one error. Report all of them.
- **Provenance preservation.** Extended-data fields written by an agent pipeline should
  survive round-trips unmodified, and the sanitisation rules should be documented at the API
  rather than discovered.
- **Clear ownership and licence capture at creation**, so agent-created layers cannot be
  published without attribution.

---

## 9. Plan

Scope, split and audience are settled (see **Decisions taken**). The sequence below reflects
them: broad Phase 1, seven skills, and MCP pulled forward out of the tail.

Two tracks run in parallel — the **capability track** (package, skills, server) and the
**platform track** (§8). They meet at the extent facet, which is why §8.1 items 1–4 are
scheduled against Phase 1 rather than left to the end.

### Phase 0 — Agree the design *(this document)*

Remaining: the demonstration dataset, the NER stack, and whether §8.2's write API is on the
TLCMap roadmap. See §12.

### Phase 1 — Core toolkit and the retrieval-to-visualisation path

Package core (`client`, `query`, `model`, `catalogue`, `provenance`, `export`, `views`) with
its test suite and the production compatibility check, plus `tlcmap-search`,
`tlcmap-analyse`, `tlcmap-visualise`, `tlcmap-cite`.

Because the package is a supported deliverable, Phase 1 ends with it installable and
documented, not merely working inside the skills.

*Platform track, in parallel:* §8.1 items 1–4 — the extent facet above all.

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

Contingent on §8.2. Adds scoped token handling to the MCP server, `push` to
`tlcmap-prepare`, and unblocks use case 7's managed layer. The server built in Phase 3 is
where the credential lives; nothing needs restructuring to get here.

### Sequencing note

Phases 1–4 need nothing from the TLCMap application to *work*. The platform items in §8.1
make Phase 1 substantially better and are in-house, so they should be scheduled with it
rather than deferred; §8.2 blocks only Phase 5. **The capability work can start immediately
and run in parallel with the platform work.**

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
2. **Write API timeline.** Is §8.2 on TLCMap's roadmap? It sets Phase 5, and it is the only
   thing standing between use case 7 and a working managed layer.
3. **Platform track ownership.** §8.1 is now in-house work. Who does it, and does the extent
   facet land in time for Phase 1 to depend on it rather than ship the harvest workaround?
4. **NER stack** — spaCy, stanza, or model-based extraction for the geoparsing skill? Affects
   the install footprint considerably, which matters more now the package is something people
   install rather than something bundled.
5. **MCP hosting.** Where does the Phase 3 server run, and does it sit behind the same
   infrastructure as the application? Relevant because §2.3 notes the API has no rate
   limiting, and a public MCP endpoint makes that more pressing.
6. **Licensing and governance of the capability itself** — the package, the skills and the
   server are now TLCMap-owned artefacts that third parties will install. What licence, and
   what support expectation?
7. **Does the browser interface benefit too?** The extent facet (§6.2) fixes a discovery
   problem the website has as well. Worth confirming before scoping it as agent work.
