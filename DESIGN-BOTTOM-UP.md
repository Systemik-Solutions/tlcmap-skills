# TLCMap AI — bottom-up delivery plan

**API → MCP → Skills, proved by a vertical slice**

**Status:** draft for discussion · **Date:** 2026-09-25 · **Supersedes the sequencing in [DESIGN.md §9](./DESIGN.md#9-plan)**

---

## 0. What this document is, and what it replaces

[DESIGN.md](./DESIGN.md) designed the capability top-down: seven skills, a Python toolkit
beneath them, an MCP server later, and a list of API changes derived from what the skills
turned out to need. That analysis stands. What it got wrong was the **build order**.

This document proposes building in dependency order instead — **API first, then MCP, then
skills** — with a single vertical slice cutting through all three layers up front to prove the
shape before any layer is built out.

| Carried over from DESIGN.md unchanged | Superseded by this document |
| --- | --- |
| §2 What the platform offers, and its traps | §4.1 Skills-first rationale |
| §3 Design principles (all eight) | §4.2 Packaging |
| §8 The API enhancement programme | §6 The shared toolkit — shrinks by roughly half |
| §11 Risks and ethics | §9 The phase plan |
| [USE-CASES.md](./USE-CASES.md) in full | §5 The skills — four hold, two reshape, one dissolves (§6.3) |

Read §3 of DESIGN.md before this document. The determinism boundary, provenance, attribution
and "fix it upstream" are the rules this plan executes; none of them change.

---

## 1. Why bottom-up

### 1.1 It is what the design already implies

DESIGN.md §3.8 says platform gaps get fixed in the platform, and §8.6 makes API changes
prerequisites that gate every phase. Then §9 schedules skills at Phase 1 and the MCP server at
Phase 3. That ordering was a holdover from treating skills as the product and MCP as reach.
Once TLCMap is the host, the dependency order and the delivery order should agree.

### 1.2 It stops us building the same thing twice

This is the strongest argument, and it is about wasted work rather than principle.

Under the top-down plan, Phase 1 builds a Python HTTP client, query builder, response
normaliser and catalogue cache. Phase 3 then builds an MCP server that makes most of that
redundant for anything able to reach the server. Building API → MCP → skills, that layer is
never written at all: the skills call tools, and the tools are the server's job.

What survives in Python is only what must be local:

| Stays local | Why |
| --- | --- |
| Text chunking and offset anchoring | Operates on the researcher's document, not on TLCMap |
| Export to QGIS, GPX, Leaflet, notebooks | File production on the researcher's machine |
| Provenance manifests | Records what *this* run did |
| Local statistics not exposed by the API | Shrinks further as §8.3 ⑫ lands |

Gone: `client.py`, `query.py`, `model.py`, `catalogue.py` — roughly half of
[DESIGN.md §6.1](./DESIGN.md#61-modules).

### 1.3 The middle layer has standalone value

A read-only MCP server is useful with no skills at all: Claude Desktop, a chatbot on
tlcmap.org, and any other agent framework. The API work is useful with no MCP server at all —
it improves every existing TLCMap client and the browser interface.

That de-risks the programme. If this is deprioritised after two layers, TLCMap still has a
better API and an AI gateway. Under the top-down plan, stopping after Phase 1 leaves a Claude
Code plugin and nothing else.

### 1.4 It plays to the team in the right order

The first two layers are PHP and Laravel in a codebase the team owns. Skills are the
unfamiliar part, and doing them last means building on ground that is already solid.

### 1.5 The risk this creates, and the answer to it

Designing an API surface and a tool surface for workflows nobody has built yet. DESIGN.md §8
is careful, but it was derived from *imagining* the skills, and some of it will be wrong.

The classic failure mode is specific and worth naming: **tool surfaces designed without
workflow experience mirror the data model rather than the task.** You expose `getLayer`,
`searchPlaces`, `listLayers` — a thin CRUD veneer — then discover agents actually need
`resolve_candidates` with evidence attached, which cuts across those primitives.

The answer is §3: build one thin vertical slice through all three layers first.

---

## 2. The three layers

```
┌─────────────────────────────────────────────────────────────┐
│  Layer 3  SKILLS            workflow, judgement, ethics     │
│           Markdown + thin Python.  Orchestrates tools.      │
├─────────────────────────────────────────────────────────────┤
│  Layer 2  MCP SERVER        discrete tools, no judgement    │
│           PHP/Laravel, in-app.  Opens TLCMap to any agent.  │
├─────────────────────────────────────────────────────────────┤
│  Layer 1  API               the contract. One deployment.   │
│           PHP/Laravel.  Fixes help every client.            │
└─────────────────────────────────────────────────────────────┘
```

**The division of labour**, stated once so each layer's scope is unambiguous:

| | Layer 1 — API | Layer 2 — MCP | Layer 3 — Skills |
| --- | --- | --- | --- |
| **Owns** | Data, query semantics, correctness | Tool surface, agent ergonomics | Workflow, judgement, output |
| **Makes model calls** | No | **No** | Yes |
| **Knows about agents** | No | Tool shapes only | Entirely |
| **Reversible?** | Hard — public contract | **Hardest — clients depend on it** | Easy — rewrite freely |
| **Built in** | PHP / Laravel | PHP / Laravel | Markdown + Python |

The reversibility row is the one to act on. The MCP tool surface is the least reversible thing
in the programme — once Claude Desktop users and a site chatbot depend on `tlcmap_search`'s
shape, changing it breaks them, while skills can be rewritten at will because nothing depends
on their internals. **Effort should be allocated inversely to volume here:** the server is the
smallest layer and deserves the most design scrutiny per line.

---

## 3. The vertical slice

### 3.1 Target: use case 3, scoped to one region

[Use case 3](./USE-CASES.md#3-regional-knowledge-synthesis-for-fieldwork) — regional knowledge
synthesis for fieldwork. A bounding box in; a consolidated brief, field files and a complete
attribution block out.

Three things make it the right slice:

1. **It does something researchers cannot do at all today.** Not "does it faster" — cannot do.
   Regional discovery is impossible from the catalogue because only 4 of 2,118 layers declare
   an extent ([DESIGN.md §2.2](./DESIGN.md#22-measured-on-the-live-catalogue)).
2. **It exercises the largest API change end to end.** §8.1 ①, the computed catalogue facet, is
   the item everything else depends on for discovery, and the biggest single piece of platform
   work. Proving it in a real workflow first is a better use of a slice than proving four small
   fixes.
3. **It puts the rights machinery under real load** (§3.3), which is the part of this design
   that is hardest to retrofit and most costly to get wrong.

The discipline that keeps it a slice rather than a phase is **narrow scope within the use
case**:

| In the slice | Explicitly out |
| --- | --- |
| **One region**, end to end | Arbitrary regions at arbitrary scale |
| Discovery → retrieval → rights → dedupe → brief → field files | Resolution, geoparsing, upload, write |
| 6 MCP tools | The remaining six |
| API ① ② ④ ⑦ ⑫ — five items | The other thirteen |
| Markdown brief, GPX, KML | QGIS project styling, PDF typesetting, storymaps |
| Rights surfaced and gated | Automated rights *decisions* — never in scope at all |

### 3.2 The region: the Hunter Valley

```
bbox 150.8,-33.1,151.4,-32.6
  → 273 records across 36 distinct contributed layers
```

Chosen because it is real, dense enough to be interesting without exceeding the 5,000-record
ceiling, and carries the colonial and convict history that makes a brief worth reading.
Wollombi, Cessnock and the Singleton parishes sit inside it, so the gazetteer half is
substantial too.

The 36 layers are a genuine long tail: weather stations (461) contributes 69 records, polling
places (717) nineteen, and a dozen layers contribute one record each. That tail is what makes
deduplication and grouping non-trivial rather than decorative.

### 3.3 What is actually there — measured

Run on 2026-09-25 against production. The 36 layers break down like this:

| | |
| --- | --- |
| Carry a **warning** | 8 |
| Carry a **licence** | 12 — in six different spellings |
| Carry a **citation** | 8 |
| **No licence at all** | **24** |

That distribution is the slice's real test, and four layers make it concrete.

**Layers 2749 and 2849 — "Public schools attended by Aboriginal and/or Torres Strait Islander
students"**, licensed `CCBY-NC-ND`, each carrying:

> *"Aboriginal and/or Torres Strait Islander Peoples are advised that this map may contain
> links to images and words…"*

An ordinary regional query surfaces these without anyone asking for sensitive material. The
advisory has to travel into the brief, the GPX file and any figure — not be summarised, and
not be dropped because an output format is inconvenient.

**Layer 1125 — "50 words project"**, licensed:

> `"Closed (subject to the access condition details)"`

The exact string [use case 4](./USE-CASES.md#4-indigenous--colonial-name-co-mapping) uses to
argue that licences must never be parsed. A client matching known identifiers finds none and
treats it as unrestricted; one that pattern-matches "Closed" and guesses is worse for being
confident. Correct behaviour is to surface it verbatim and stop.

**Layer 206 — "Music communities"**, whose `warning` field contains the literal string:

> `"None"`

A present warning that says nothing, which is not the same as an absent one. Small, and
exactly the kind of contributor-data reality that separates a design that works from one that
demos.

**And 24 layers carry no licence**, which is the majority. The brief must say that absence of
a licence is not permission, for each of them, without editorialising further.

### 3.4 Success criteria

Three of the five are numbers rather than judgements, which was the main worry about this use
case as a slice.

1. **Layer discovery recall and precision.** Ground truth computed once, offline, by harvesting
   all 2,118 public layers and calculating true extents — an *evaluation artefact, not a
   product feature* (§3.5). `tlcmap_list_layers` with a bbox must find every layer with a record
   in the box and no others. **Target: recall 1.0.** Anything less means §8.1 ① is wrong, which
   is precisely what the slice exists to find out.
2. **Attribution completeness — must be 1.0.** All 36 layers appear in the attribution block
   with `creator`, `licence`, `citation` and `warning` reproduced verbatim. Automatable, binary,
   and a single omission fails the slice.
3. **Warning propagation — must be 1.0.** The two advisory warnings appear in every derived
   output, including the GPX and KML files, not only the markdown brief.
4. **The restricted layer stops the pipeline.** Layer 1125 routes to a human decision rather
   than into the brief, and the reason shown is the licence text itself.
5. **Exclusions are stated.** The brief says what it could not see: undated records dropped by
   any date bound, layers whose licence is absent, records without coordinates.

Plus one qualitative criterion, reviewed rather than scored: **a domain reader finds the brief
usable in the field.** Worth doing, and worth not pretending is a metric.

### 3.5 What the slice does not prove

Stated plainly, because this choice leaves a larger gap than a resolver-driven slice would.

- **It does not test resolution.** No adjudication, no candidate ranking, no abstention. The
  model's role here is synthesis and grouping, not judgement over evidence — so the central
  claim behind use cases 1, 2 and 6 stays unproven until Phase D. This is the main cost of the
  choice, and it should be planned for rather than discovered.
- **It does not test extraction or anchoring**, and so gives no evidence on the decision to drop
  NER libraries ([DESIGN.md §5.3](./DESIGN.md#53-tlcmap-geoparse--map-the-places-mentioned-in-a-document)).
- **It does not validate the tool surface for the resolver-driven use cases.** Six tools
  exercised, six inferred.
- **It does not prove scale.** One region, comfortably under the ceiling. A dense region —
  Sydney, Melbourne — is a different problem, and is Phase B's.
- **It does not make the harvest a product.** The exhaustive extent harvest exists once, in
  `evals/`, to generate ground truth. It is not shipped, not run by users and not a fallback
  path — that would reintroduce exactly what [DESIGN.md §3.8](./DESIGN.md#38-fix-it-upstream)
  removed.

### 3.6 How Phase A is staged

Two milestones, so the smaller result finishes and is demonstrable before the second begins.

| | Scope | Deliverable |
| --- | --- | --- |
| **A1** | Discovery and rights. Tools 1–4; API ① ② ④ ⑦. Skills `tlcmap-search`, `tlcmap-cite`. | **A rights-complete regional inventory** — every layer and record TLCMap holds for the Hunter Valley, with attribution, warnings and exclusions. Useful on its own, and not obtainable today by any means. |
| **A2** | Synthesis and field outputs. Tools 5–6; API ⑫. Skills `tlcmap-analyse`, `tlcmap-visualise`. | **The brief** — deduplicated, grouped by feature type, characterised with computed statistics, plus GPX and KML for the field. |

A1 carries all three numeric success criteria. If A2 runs long or the synthesis disappoints,
A1 has already answered what the slice was built to ask: whether §8.1 ① works, and whether the
rights machinery holds under a real query.

---

## 4. Layer 1 — the API foundation

The full programme is [DESIGN.md §8](./DESIGN.md#8-api-enhancements--the-platform-half-of-this-project),
unchanged. What changes is sequencing: instead of "prerequisites for Phase 1", the items are
now ordered by **what the slice needs, then what the tool surface needs, then the rest.**

### 4.1 Needed by the slice

| Item | Milestone | Why the slice needs it |
| --- | --- | --- |
| **① Computed catalogue facet** | A1 | Regional layer discovery in one request. **The slice exists to prove this one**, and it is the largest single item in the programme. |
| **② Honest errors** | A1 | The server maps status codes to tool errors. Without it every tool sniffs content types — and the sniffing would live in PHP, permanently, in the layer hardest to change. |
| **④ Cheap count** | A1 | A regional query must know how big it is before committing to the fetch. Today `paging=1` on an oversized query still fails. |
| **⑦ `nulls=last` / `include_undated`** | A1 | A brief for "the Hunter Valley, 1820–1860" silently loses every undated record otherwise — and most gazetteer records are undated. The trap this slice is most likely to hit. |
| **⑫ Statistics as JSON** | A2 | Characterising what is in the region. The numbers exist today only inside an HTML page. |

### 4.2 Needed by the full tool surface

**③ `/api` over contributed layers** (removes the 5,000 ceiling as a design concern),
**⑤ namespaced extended data**, **⑥ `udateend` fix**, **⑨ match score**, **⑩ bulk fetch by ID**,
**⑪ vocabulary endpoints**.

⑨ ⑩ ⑪ are the resolver's, and land with Phase D rather than Phase A — the consequence of a
slice that does not exercise resolution (§3.5).

### 4.3 The rest

**⑧ ⑬ ⑭ ⑮ ⑯ ⑰ ⑱** — non-blocking, land when convenient. **§8.4, the write API** — Layer 1
work, but scheduled with the write phase (§7).

### 4.4 A note on sequencing the API work

Resist doing all eighteen items before anything else. §4.1's four unblock the slice; that is
where to start. The point of bottom-up is dependency order, not a big-bang API project with no
feedback for months.

---

## 5. Layer 2 — the MCP server

The part DESIGN.md left as a sketch. This is the design.

### 5.1 Architecture

**PHP and Laravel, inside the TLCMap application.** Same stack, same deployment, same
database, and at the write phase the same authentication and permission model. It is not in
this repository and shares no code with the skills' Python.

```
  Agent host (Claude Code / Desktop / site chatbot / other)
        │  MCP over HTTP
        ▼
  ┌──────────────────────────────┐
  │  mcp.tlcmap.org              │   Laravel, in-app
  │   ├─ tool definitions        │   thin: validation + shape
  │   ├─ resource handles        │   bulk data by reference
  │   └─ provenance envelope     │
  └────────────┬─────────────────┘
               │  internal query layer (not HTTP)
               ▼
        TLCMap database / PostGIS
```

Served over HTTP rather than stdio, because the point is reach: a remote endpoint needs no
local install, which keeps the skills' setup to one line of configuration.

### 5.2 Design rules

Six rules, each with a reason. They matter more than the tool list because they are what makes
the surface survivable.

1. **No model calls, ever.** The server exposes data and leaves judgement to the client that
   holds the model. This is what keeps prompts, adjudication and confidence banding in one
   place rather than reimplemented in PHP — and it is why `tlcmap_resolve_candidates` returns
   candidates rather than a choice.
2. **Handles and summaries, never bulk.** A tool result returns
   `{count, extent, date_range, resource_uri}`; a layer's features live behind a resource the
   client fetches only if it needs them. DESIGN.md §3.5 restated for a protocol where
   everything returns through the model.
3. **The tools expose the API contract, nothing more.** A server with direct database access
   *could* answer questions the HTTP API cannot, and must not: one contract, two transports, so
   a question has the same answer whichever way it is asked. Anything worth adding is added to
   §8 and reaches both.
4. **Every result carries provenance** — the equivalent query URL, the fetch time, and the
   record UIDs. An MCP-driven answer must be as retraceable as a skill-driven one.
5. **Errors are typed and actionable.** `RESULT_TOO_LARGE` carries the total and a suggestion;
   `FILTER_REJECTED` names the expression it could not parse. This is §8.1 ② surfacing at the
   tool layer, and it is why ② is slice-critical.
6. **Read and write are separate scopes from day one**, so the public read-only deployment and
   an authenticated one are the same server configured differently.

### 5.3 The tool surface

Twelve read tools. **Six are in the slice** (marked ●), specified in full below; the other six
are sketched and follow once the slice has validated the shape.

`tlcmap_list_layers` is the one the slice really tests, because it is the tool §8.1 ① exists to
make possible.

#### ● `tlcmap_list_layers`

Faceted catalogue search. The tool regional discovery is impossible without.

| | |
| --- | --- |
| **In** | `query`, `bbox`, `polygon`, `date_from`, `date_to`, `include_undated`, `record_type`, `creator`, `has_license`, `has_warning`, `min_records`, `limit`, `cursor` |
| **Out** | `{total, returned, layers[{layer_id, name, description, creator, record_count, bbox, date_range, has_warning, license_present}], cursor, provenance}` |
| **Errors** | `INVALID_GEOMETRY`, `RESULT_TOO_LARGE` |

**Design notes.** `bbox` matches against the **computed** extent (§8.1 ①), never the
contributor-declared fields — which exist on 4 of 2,118 layers and are malformed on two of
those. Returns layer *summaries* only; records come from `tlcmap_get_layer`.

`has_license` and `has_warning` are booleans reporting presence, **not** an interpretation of
content. They let an agent find the 24 no-licence layers in a region without the server ever
deciding what a licence permits.

#### ● `tlcmap_search_places`

Search places across all four sources.

| | |
| --- | --- |
| **In** | `name_query`, `match` (`exact`\|`contains`\|`fuzzy`), `sources[]`, `bbox`, `polygon`, `date_from`, `date_to`, `include_undated`, `state`, `lga`, `feature_term`, `record_type`, `layer_ids[]`, `extended_data[]`, `count_only`, `limit`, `cursor` |
| **Out** | `{total, returned, extent, date_range, sources_breakdown, layers_present[], places[], resource_uri, provenance}` |
| **Errors** | `RESULT_TOO_LARGE` (with `total` + narrowing suggestions), `FILTER_REJECTED`, `INVALID_GEOMETRY` |

**Design notes.** `match` replaces the `name`/`containsname`/`fuzzyname` trap — one parameter
with three honest values rather than three parameters with precedence rules. `include_undated`
is explicit rather than a silent exclusion. `count_only` (§8.1 ④) answers "how big is this?"
without building features, which is how the slice decides whether a region is tractable before
committing to the fetch. There is **no `limit`-as-random-sample**: sampling, if ever needed, is
a separate `sample` parameter, never the default reading of "limit".

`layers_present` is a convenience the slice leans on — the distinct layers contributing to a
result, so an agent can go straight to their rights without paging every record.

#### ● `tlcmap_get_layer`

A layer's records and metadata.

| | |
| --- | --- |
| **In** | `layer_id`, `sort`, `nulls` (`first`\|`last`), `line` (`none`\|`route`\|`time`), `bbox` |
| **Out** | `{metadata, record_count, extent, date_range, undated_count, resource_uri, provenance}` |
| **Errors** | `LAYER_PRIVATE` (403), `LAYER_NOT_FOUND` (404) |

**Design notes.** Summary plus a resource handle; never inline features (rule 2). `nulls=last`
(§8.2 ⑦) stops sorting from silently deleting undated records — a 216-record layer returning 55
is the current behaviour. `undated_count` is returned explicitly so the exclusion can be
reported rather than discovered.

#### ● `tlcmap_get_layer_metadata`

Rights, licence, citation and warning, without building any features.

| | |
| --- | --- |
| **In** | `layer_id` |
| **Out** | `{name, creator, publisher, contact, citation, license, rights, doi, warning, temporal_extent, spatial_extent, record_count, provenance}` |
| **Errors** | `LAYER_PRIVATE` (403), `LAYER_NOT_FOUND` (404) |

**Design notes.** Cheap by design, so an agent can check rights *before* fetching data — which
is what makes DESIGN.md §3.3's ordering enforceable rather than aspirational.

`license`, `rights` and `warning` are returned **verbatim as strings**. The server does not
normalise them, does not infer permission, does not omit an unparseable one, and does not
helpfully convert the literal string `"None"` in layer 206's warning field into an absent
warning. If §8.3 ⑮ adds a structured identifier it appears as an *additional* field, never as a
replacement.

This is the most-called tool in the slice: 36 layers, 36 calls, every one of them before any
data is used.

#### ● `tlcmap_list_vocabulary`

Valid values for the exact-match filters.

| | |
| --- | --- |
| **In** | `field` (`feature_term`\|`state`\|`lga`\|`parish`\|`record_type`), `bbox` (optional, to scope to a region) |
| **Out** | `{field, values[{value, record_count}], provenance}` |

**Design notes.** Requires §8.3 ⑪. Small, unglamorous, and the difference between a filter that
works and one that silently matches nothing — `lga=CESSNOCK` matches and `lga=Cessnock Council`
does not, and nothing today exposes which is which. The optional `bbox` is what lets the slice
group a region's records by the feature terms that actually occur in it.

#### ● `tlcmap_layer_statistics`

The numbers, as numbers.

| | |
| --- | --- |
| **In** | `layer_id`, or `bbox` + `layer_ids[]` for a region |
| **Out** | `{count, extent, centroid, convex_hull, area_km2, density_per_km2, date_range, median_date, mean_date, undated_count, feature_term_breakdown, provenance}` |
| **Errors** | `LAYER_PRIVATE`, `INSUFFICIENT_RECORDS` |

**Design notes.** Requires §8.3 ⑫. Today `basicstatistics/json` returns geometry only and the
statistics exist solely inside an HTML page, so every client recomputes what the server already
calculated. `feature_term_breakdown` is what the brief groups by.

---

The remaining six, designed but not built in the slice. Three belong to the resolver and land
with Phase D:

#### `tlcmap_resolve_candidates`

Given a placename and what is known around it, return ranked candidates **with evidence** —
`match_score` (the trigram similarity the database already computes, §8.2 ⑨), `matched_on`
naming which field matched, and `distance_from_prior_km` turning a spatial prior into a number
the model can weigh rather than a filter that silently excludes. It returns candidates and
never a decision. The most important tool outside the slice, and the one that most repays care
when Phase D reaches it.

#### `tlcmap_get_places`

Bulk fetch by TLCMap ID, up to 200 per call. Requires §8.3 ⑩; without it the resolver makes one
request per candidate.

#### `tlcmap_get_text_layer`

A geoparsed text layer with its mention offsets, the text itself as a resource rather than a
field — layer 2377's is 1.07 MB and must never pass through the model.

#### `tlcmap_cluster_layer`

`method` (`dbscan`\|`kmeans`\|`temporal`) with method-specific parameters. Distance in **metres
against geography**, per §8.3 ⑬ — not the current degrees-divided-by-100.

#### `tlcmap_compare_layers`

Closeness analysis between two public layers. Warns on cost before running, since it is a cross
join.

#### `tlcmap_build_view_url`

Given a feed and an intent, return a correctly percent-encoded TLCMap Views URL, and refuse
combinations the data cannot support — a timeline needs `udatestart`/`udateend`, a journey needs
`LineString` features. Trivial, and it removes a whole class of broken embed.

### 5.4 Resources

Bulk payloads are MCP resources with stable URIs, not tool results:

| Resource | Holds |
| --- | --- |
| `tlcmap://search/{query_hash}` | The full feature collection for a search |
| `tlcmap://layer/{id}/features` | A layer's records |
| `tlcmap://layer/{id}/text` | An uploaded text's full content |
| `tlcmap://layer/{id}/crate` | The RO-Crate snapshot |

Content-addressed where practical, so a client can cache and a provenance record can cite a
specific state of the data.

### 5.5 Tool surface stability

The least reversible thing in the programme (§2), so it is governed explicitly:

- **Additive changes only** after first publication: new tools, new optional parameters, new
  output fields. Removing a field or changing its meaning is breaking.
- **Version the surface, not the API.** The TLCMap API stays unversioned (DESIGN.md §2.4); the
  MCP server declares a version and can keep a deprecated tool alive through a transition.
- **The slice is the design review.** Tools 6–12 are not published until one real workflow has
  exercised tools 1–5, because that is the only evidence that will exist about whether the
  shapes are right.

### 5.6 What the server must not do

- No model calls.
- No judgement: no "best match", no confidence, no ranking that encodes a decision rather than
  a measurement.
- No capability beyond the HTTP API (rule 3).
- **No summarising or normalising of `license`, `rights` or `warning`.** Verbatim or not at
  all.
- No write, until scopes and tokens exist.

---

## 6. Layer 3 — skills as orchestration

The seven skills in [DESIGN.md §5](./DESIGN.md#5-the-skill-set) survive as capability
definitions, but **not uniformly**, and it would be wrong to claim otherwise. Four are
unchanged or strengthened, two are meaningfully reshaped, and one is at risk of dissolving
into the tool layer entirely.

### 6.1 What a skill becomes

Before: instructions plus a Python toolkit that speaks HTTP, builds queries, normalises
responses and caches a catalogue.

After: instructions plus **tool calls**, with Python only where the work is genuinely local. A
skill narrows to what it was always best at — deciding what to ask, judging what comes back,
and producing the output.

### 6.2 The test for whether something should be a skill at all

MCP changes this test, and the change is easy to miss. A host with the server configured and
**no skills installed** can already search TLCMap, read a layer and check a licence. So a skill
can no longer justify itself by knowing how to call an endpoint — that knowledge now lives in
the tool schema, available to everyone.

> **If a competent model holding the tool list would do this correctly unaided, it does not
> need a skill.**

What still earns a skill: multi-step pipelines, judgement boundaries, ethics gates, local
artefact production, and provenance. What no longer does: parameter selection, endpoint choice,
and workarounds for behaviour §8 has fixed.

### 6.3 Skill by skill

#### Unchanged or strengthened

**`tlcmap-resolve`** keeps its shape exactly. Candidate generation moves into
`tlcmap_resolve_candidates`, but everything that made it a skill stays: the adjudication
prompt, confidence banding, review-bucket routing, coordinate verification, the audit log. A
multi-step workflow with a judgement in the middle is precisely what a tool cannot be.

**`tlcmap-geoparse`** is barely touched. Chunking, extraction and anchoring operate on the
researcher's document, not on TLCMap, and have no tool at all. Only resolution delegates.

**`tlcmap-cite`** gets *stronger* relative to the tools. `tlcmap_get_layer_metadata` hands it
the data cheaply, but the rules — never parse a licence into a permission, warnings travel into
every derived output, stop at the ethics gate — are pure judgement. A tool can return
*"Do not use without permission."*; only a skill can be relied on to stop when it sees it.

**`tlcmap-prepare`** is largely unaffected, since validation is local: date forms, heading
sanitisation, coordinate plausibility, the `ghap_id` round-trip. §8.3 ⑰'s dry-run endpoint
moves part of it server-side later, and the write phase gives it `push`.

#### Reshaped — still skills, different content

**`tlcmap-analyse`** changes more than §6.1 implies. Its DESIGN.md definition carried a
server-side-or-local decision table plus a lot of trap knowledge: DBScan's `distance` being
degrees ÷ 100, KMeans needing `withinRadius=` present but empty, `basicstatistics` returning
geometry only. With `tlcmap_cluster_layer` taking metres and `tlcmap_layer_statistics`
returning real numbers, **all of that evaporates.** What remains is better material for a
skill: choosing which analysis answers the question, interpreting the result, drafting prose
strictly from computed numbers, reporting exclusions, emitting the notebook.

**`tlcmap-visualise`** thins on the TLCMap Views side, where `tlcmap_build_view_url` absorbs
the percent-encoding and the view-compatibility rules. The local half — Leaflet, QGIS, GPX,
matplotlib, storymap scaffolds — is untouched.

#### At risk of dissolving

**`tlcmap-search`** does not survive in its current form. Its decision procedure was mostly
*which of `name`/`containsname`/`fuzzyname`, which endpoint, what to do at the ceiling, and how
to discover layers when the catalogue cannot*. With `match: exact|contains|fuzzy` as one honest
parameter, `tlcmap_list_layers` doing discovery, and §8.1 ③④ removing the ceiling as a design
concern, a model holding the tool schema does most of that unaided — it fails the §6.2 test.

What is left is thin: the research implications (undated records silently excluded, gazetteer
versus contributed), the cached artefact with provenance, and composition into other skills.
**That may be better as a reference file the other skills load than as a skill of its own.**

**The seven-skill split is therefore reopened, and Phase A should settle it.** Current
expectation is six, with `tlcmap-search` folded into shared reference material — but that is a
question for evidence from a real workflow, not for a decision now.

### 6.4 Where guidance lives

A new failure mode arrives with the tool layer: **guidance duplicated between tool descriptions
and skill instructions will drift**, and the two will eventually disagree in front of a
researcher. One rule:

> Tool descriptions carry what you need to **call it correctly**.
> Skills carry what you need to **decide whether to call it**.

*"Never use `limit` to truncate a result set"* belongs in a description. *"A dated search
excludes undated gazetteer records, which usually matters more than the researcher expects"*
belongs in a skill. Neither belongs in both.

### 6.5 The slice skills

Phase A builds four, across the two milestones in §3.6.

**A1 — `tlcmap-search` and `tlcmap-cite`.** Discovery and rights.

| Step | Where it runs |
| --- | --- |
| Turn a region description into a query | **Model** |
| Discover layers intersecting the region | **`tlcmap_list_layers`** |
| Check the size before committing | **`tlcmap_search_places`** (count mode) |
| Retrieve gazetteer records and layer contents | **`tlcmap_search_places`**, **`tlcmap_get_layer`** |
| Read rights for every layer touched | **`tlcmap_get_layer_metadata`** |
| Gate on restricted layers | **Model proposes, human decides** |
| Assemble the attribution block, verbatim | **Local Python** |
| Report exclusions | **Local Python** |

**A2 — `tlcmap-analyse` and `tlcmap-visualise`.** Synthesis and field outputs.

| Step | Where it runs |
| --- | --- |
| Deduplicate across layers | **Local Python** |
| Characterise the region | **`tlcmap_layer_statistics`** |
| Group by feature type | **`tlcmap_list_vocabulary`** + **Local Python** |
| Draft the brief from computed numbers | **Model** |
| Produce GPX, KML and the markdown brief | **Local Python** |
| Carry warnings into every output | **Local Python** |

The determinism boundary reads clearly here: **the model appears three times**, and never
produces a fact. It turns a region description into a query, it proposes which layers need a
human decision, and it writes prose from statistics computed elsewhere. Every coordinate,
count and distance comes from the tools or from local code.

### 6.6 Packaging

Unchanged in shape from [DESIGN.md §4.2](./DESIGN.md#42-packaging) — a plugin with `lib/` and
PEP 723 scripts — but `lib/` is roughly half the size, and the plugin now depends on the MCP
server being configured. For a remote server that is one line of configuration; the skills
document it, and fail with a clear message rather than a stack trace when it is absent.

Since §6.3 reopens the skill count, the plugin manifest should not be treated as settled until
Phase A reports.

---

## 7. Plan

### Phase A — the vertical slice

Run as two milestones (§3.6), so the first is demonstrable before the second begins.

**A0 — spike, first days.** Confirm the PHP MCP tooling holds up (§9). It is the only open
question that could redirect the architecture, and it should be answered before the API work
commits to a tool-shaped contract.

**A1 — discovery and rights.**
*API:* §8.1 ① ② ④, §8.2 ⑦.
*MCP:* `tlcmap_list_layers`, `tlcmap_search_places`, `tlcmap_get_layer`,
`tlcmap_get_layer_metadata` — HTTP transport, provenance envelope, typed errors.
*Skills:* `tlcmap-search`, `tlcmap-cite`.
*Evidence:* ground truth from a one-off exhaustive extent harvest of all 2,118 public layers,
held in `evals/` and never shipped (§3.5).

**Deliverable:** a rights-complete inventory of everything TLCMap holds for the Hunter Valley —
36 layers, 273 contributed records plus the gazetteer — with attribution and warnings verbatim,
the restricted layer gated, and exclusions stated. Layer discovery scored for recall and
precision against the harvest.

**A2 — synthesis and field outputs.**
*API:* §8.3 ⑫.
*MCP:* `tlcmap_list_vocabulary`, `tlcmap_layer_statistics`.
*Skills:* `tlcmap-analyse`, `tlcmap-visualise`.

**Deliverable:** the fieldwork brief — deduplicated across the 36 layers, grouped by feature
term, characterised with computed statistics — plus GPX and KML carrying the advisory warnings
into the field files.

**Decision gate**, read at the end of each milestone rather than once at the end:

- *A1 fails on discovery recall* — §8.1 ① is wrong, and that is the finding. It is the item the
  rest of the programme leans on hardest, so learning it here rather than at Phase C is the
  point of the slice.
- *A1 fails on attribution or warning propagation* — more serious than it sounds. Those criteria
  are binary and automatable, and a failure means the rights machinery does not survive contact
  with real contributor data. Fix before anything else proceeds.
- *A2 disappoints* — the synthesis needs work, but A1 has already shipped something obtainable
  no other way. Proceed to Phase B and revisit the brief.
- *All pass* — proceed to Phase B with the largest API change and the rights machinery both
  validated against a real workflow.

### Phase B — API programme

§4.2's items: ① ③ ④ ⑤ ⑥ ⑦ ⑫, then the non-blocking remainder. Valuable independently of
everything above them.

### Phase C — full read-only MCP server

Tools 6–12, designed against what Phase A learned. Published at `mcp.tlcmap.org`, with rate
limits (§8.3 ⑱) in place before it is public.

**Deliverable:** TLCMap usable from Claude Desktop and any other MCP host, with no skills
involved. The first deliverable that reaches users outside Claude Code.

### Phase D — the skill set

The remaining six skills as orchestration over the tool surface, plus the evaluation harness
and the demonstrations in [USE-CASES.md](./USE-CASES.md).

### Phase E — write

§8.4, scoped tokens, write tools behind the scope separated in Phase A, and `push` in
`tlcmap-prepare`. Unblocks
[use case 7](./USE-CASES.md#7-comparative--longitudinal-mapping-of-a-single-concept).

### Sequencing notes

- **Phases B and C can overlap.** Tools whose API prerequisites have landed can be built while
  the rest of the API work proceeds.
- **Phase A is not a prototype to throw away.** It is the first increment of all three layers,
  built to the standards in DESIGN.md §3.
- **Time to visible value** is the metric to watch. Phase A exists so that something is
  demonstrable early; if it slips past a few months, the scope is wrong, not the plan.

---

## 8. Measurement

Carried from [DESIGN.md §10](./DESIGN.md#10-how-we-will-know-it-works), re-pointed at this
sequence:

| What | When | Measure |
| --- | --- | --- |
| **Layer discovery recall** | Phase A1 | Against the offline extent harvest. Target 1.0 — it is what §8.1 ① is for |
| **Attribution completeness** | Phase A1 | Every layer touched, verbatim. **Must be 1.0** |
| **Warning propagation** | Phase A1 | Into every derived output including GPX/KML. **Must be 1.0** |
| **Rights gating** | Phase A1 | Restricted layers stop the pipeline rather than entering the brief |
| **Resolution accuracy** | Phase D | Precision, recall and **abstention rate**. Deferred with the resolver (§3.5) |
| **Extraction accuracy, anchor drop rate** | Phase D | Deferred with geoparsing (§3.5) |
| **Provenance integrity** | Phase A | Every coordinate traces to a fetched record. Automated, and tested as though it could fail |
| **Tool surface churn** | Phase C | Breaking changes after publication. Target: zero |
| **API compatibility** | Continuous | The production suite catching silent platform changes |
| **Skill triggering** | Phase D | Fixtures from the example prompts in USE-CASES.md, including the push-back prompts |
| **Courtesy** | Continuous | Requests per run, kept visible and low |

---

## 9. Risks

**The tool surface is designed on partial evidence.** Phase A validates five tools against one
workflow; seven more are inferred. Mitigated by §5.5's additive-only rule and by holding
publication of tools 6–12 until Phase A reports.

**Nothing visible for a while.** The main cost of bottom-up. Phase A is the answer, and its
scope discipline (§3.1) is what keeps it from becoming a phase. A1 shortens the wait further —
a rights-complete regional inventory is demonstrable without any synthesis at all.

**The slice leaves resolution unproven.** The largest risk this choice creates, and it is a
deferral rather than a mitigation: adjudication quality — the claim behind use cases 1, 2 and 6
— gets no evidence until Phase D. Two things follow. Phase D should be planned as a second
proving exercise rather than as routine build-out, with its own gold set and its own gate. And
nobody should read a successful Phase A as evidence that resolution works, because it contains
none.

**§8.1 ① is the long pole and the slice depends on it entirely.** A1 cannot start without the
computed catalogue facet. It is the largest item in the programme, which is the honest cost of
choosing the slice that proves it. The compensation is that it is the item most of the rest
depends on, so the risk is being taken early rather than avoided.

**PHP MCP tooling.** The SDK ecosystem is thinner in PHP than in TypeScript or Python. This now
sits on the critical path rather than at Phase 3, so it should be spiked **first**, in the
opening days of Phase A, before the API work commits to a tool-shaped contract.

**The ethics gate is tested, and that is a reason for confidence rather than a risk.** The
Hunter Valley query surfaces two layers carrying an Aboriginal and Torres Strait Islander
advisory, one licensed `"Closed (subject to the access condition details)"`, and 24 with no
licence at all (§3.3) — without anyone asking for sensitive material. A1's binary criteria make
that machinery pass or fail visibly. The residual gap is
[use case 4](./USE-CASES.md#4-indigenous--colonial-name-co-mapping) proper, where a researcher
*seeks out* Indigenous-name layers; the slice covers incidental exposure, not deliberate use.

**API-first becomes API-only.** Eighteen items is enough to absorb a team indefinitely. §4.4's
rule — the slice's four first — is the guard.

---

## 10. Open questions

1. **Does the PHP MCP tooling hold up?** Spike in week one of Phase A. It is the only question
   that could redirect the architecture.
2. **Who runs and owns the ground-truth harvest?** One polite pass over 2,118 layer feeds,
   computing true extents, held in `evals/`. It is the only thing standing between "discovery
   recall 1.0" and an unfalsifiable claim — and it must stay an evaluation artefact rather than
   drifting into the product (§3.5).
3. **Where does `mcp.tlcmap.org` run**, and does it share the application's infrastructure?
   Relevant because there is no rate limiting today and a public MCP endpoint makes that
   pressing.
4. **Is the tool surface right at twelve?** Fewer and richer, or more and finer? Phase A gives
   evidence for six of them; the instinct to resist is one tool per endpoint.
5. **Is the Hunter Valley the right region?** It is dense enough to be interesting and small
   enough to stay under the ceiling. A second region with different characteristics — sparse,
   or remote, or carrying more restricted layers — would test generalisation cheaply once A1
   works.
6. **Do the skills require the MCP server, or degrade without it?** Requiring it is simpler and
   honest; degrading gracefully costs the HTTP client this plan was designed to avoid building.
   Recommend requiring it.
7. **Write API timeline** — sets Phase E, unchanged from DESIGN.md §12.
8. **Licensing and governance** of the three artefacts — unchanged from DESIGN.md §12.
9. **How many skills, once the tools exist?** §6.3 reopens the seven-skill split, because
   `tlcmap-search` largely fails the §6.2 test once the tool layer carries what it used to know.
   Current expectation is six, with search folded into shared reference material. Phase A should
   settle it with evidence rather than assertion, and the plugin manifest stays unsettled until
   it does.

---

## 11. Recommendation

Adopt it. The dependency ordering is right, it avoids building an HTTP client that a later
phase makes redundant, and both lower layers have value independent of the layer above them.

The amendment that matters is Phase A. Strict three-layer sequencing would design an API and a
tool surface against workflows nobody has built — and the tool surface is the least reversible
thing here. One thin slice through all three layers, on a real region with rights messy enough
to be honest, buys the evidence before the commitment.

The trade it makes is explicit: it proves discovery, retrieval and the rights machinery, and
proves nothing about resolution. Phase D inherits that debt and should be resourced as a second
proving exercise, not as build-out.
