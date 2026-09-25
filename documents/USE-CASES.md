# TLCMap AI Skills — use cases

**Companion to [DESIGN.md](./DESIGN.md)** · **Date:** 2026-09-25 · **Status:** companion to the design

This document describes each use case in full: who it is for, what the pipeline actually
does, which parts the skills cover, **which parts they do not**, and what has to exist before
it works. §8 of the design document holds the summary table; this is the detail behind it.

Every example is drawn from real TLCMap data, checked against production on 2026-09-24. Layer
IDs resolve at `https://tlcmap.org/layers/{id}`.

**Each use case opens with example prompts** — what a researcher would actually type, the
skills that should fire, and the phrasings where the skill is expected to *push back* rather
than comply. Those are not decoration: they become the trigger-accuracy fixtures in `evals/`
([DESIGN.md §11](./DESIGN.md#11-measurement)). A skill that activates on
someone else's prompt is as much a defect as one that fails to activate on its own, and the
push-back prompts are how the determinism and ethics boundaries get tested rather than merely
asserted.

**The skills**, in brief — full definitions in [DESIGN.md §7.3](./DESIGN.md#73-the-skills):

| | |
| --- | --- |
| `tlcmap-search` | Find and retrieve places and layers |
| `tlcmap-resolve` | Placename strings → TLCMap records |
| `tlcmap-geoparse` | Places mentioned in a document → mentions with offsets |
| `tlcmap-analyse` | Spatial and temporal distribution |
| `tlcmap-visualise` | Maps, timelines, charts, embeds |
| `tlcmap-prepare` | Build and validate a TLCMap-ready layer |
| `tlcmap-cite` | Attribution, licensing, citable packaging |

Two rules run through all eight, and are worth holding in mind while reading:

> **The model chooses; code computes; TLCMap supplies the facts.** No coordinate, count or
> distance is ever produced by the model. ([DESIGN.md §3.1](./DESIGN.md#31-the-determinism-boundary))

> **Attribution travels; permission is never guessed.** Licence and rights text is surfaced
> verbatim, never parsed into a yes or no. ([DESIGN.md §3.3](./DESIGN.md#33-attribution-travels-permission-is-never-guessed))

---

## Contents

| # | Use case | Status |
| --- | --- | --- |
| [1](#1-historical-text--mapped-corpus) | Historical text → mapped corpus | Phase D |
| [2](#2-place-based-corpus-enrichment) | Place-based corpus enrichment | Phase D |
| [3](#3-regional-knowledge-synthesis-for-fieldwork) | Regional knowledge synthesis for fieldwork | **Phase A — the vertical slice** |
| [4](#4-indigenous--colonial-name-co-mapping) | Indigenous / colonial name co-mapping | Human-gated by design |
| [5](#5-toponym-pattern-analysis) | Toponym pattern analysis | Phase B |
| [6](#6-environmental--event-history-overlay) | Environmental / event history overlay | Phase B or D |
| [7](#7-comparative--longitudinal-mapping-of-a-single-concept) | Longitudinal mapping of a single concept | Phase E — blocked on the write API |
| [8](#8-teaching--public-facing-storymaps) | Teaching / public-facing storymaps | Phase B |
| [+](#additional-capabilities) | Additional capabilities not in the original brief | — |

---

## 1. Historical text → mapped corpus

### Who and what

A literary or history researcher holds a corpus — colonial diaries, convict records, Trove
newspaper articles, explorers' journals — and wants a map of every place mentioned in it,
with frequency, date and the source passage behind each pin.

The work is currently manual and does not scale: read, note the placename, look it up, find a
coordinate, record which page it came from. For a corpus of any size it simply does not
happen, which is why this is the use case researchers ask about first.

### Example prompts

> *"I've got a folder of OCR'd convict diaries. Map every place they mention, with how often
> each one comes up and the passage it came from."*

`tlcmap-geoparse` → `tlcmap-resolve` → `tlcmap-visualise` → `tlcmap-cite`

Other phrasings that should reach the same pipeline:

- *"Pull the place names out of these 200 Trove bushfire articles and map them by decade."*
- *"Which places does this explorer's journal actually name, and where are they?"*
- *"I want a map of everywhere mentioned in this book, linked back to the page it's on."*

**Where it pushes back.** *"Just geocode everything you find."* — it will not. The output
reports how many mentions resolved confidently, how many went to review, and how many were
dropped at anchoring because the surface form was not in the source text. Forcing a match on
an ambiguous colonial name is how you get `GREECE` at a Sydney coordinate.

**Where it offers something simpler.** *"Map the places in this one diary."* — for a single
moderate document the skill says so and offers TLCMap's own text upload first, which produces
a proper text layer and the Full Text view without any of this.

### A real example

**Layer 170, "19th Century Australian Bushfire Reporting"** (Fiannuala Morgan, CC BY) is this
pipeline already run once, by hand and by earlier tools. It holds **7,454 records** drawn
from 19th-century newspapers, every one dated, spanning **1824-05-14 to 1899-12-30**, with
`Newspaper`, `Newspaper Place of Publication`, `Article Link` and `Article Word Count` carried
as extended data.

It is also an honest illustration of where the difficulty lies. The layer's own warning says
so plainly:

> *"Placenames may be incorrectly spelled due to the quality of source material. Geographic
> coordinates have been computationally generated and may be imprecise. Recorded dates
> correspond to date of publication and not necessarily to the date on which a bushfire
> occurred."*

Across 2,824 distinct placenames the common ones are exactly right — `MELBOURNE` (522),
`SYDNEY` (434), `BALLARAT` (121), `GEELONG` (89) — but the tail contains `GREECE` (4) and
`EGYPT` (9). The layer's very first record is a poem, *"TRIBUTARY LINES, Addressed to
LIEUTENANT GOVERNOR…"*, with placename `GREECE` and a coordinate in Sydney.

That is the residual error class this use case targets: a classifier working on the token
`Greece` has no way to know it is a rhetorical flourish in verse rather than a place the
article is about. A model reading the sentence does. The point is not that the existing layer
is poor — its creator documented the limitation precisely — but that **sentence context is
the thing that fixes it, and sentence context is what an LLM brings.**

### The pipeline

1. **Ingest and normalise** — PDF, OCR text, Trove XML or plain text. Chunk into overlapping
   windows with stable byte offsets. *Deterministic, in code.*
2. **Extract** (`tlcmap-geoparse`) — the model returns, per mention, the **verbatim surface
   form** and its sentence. Never a character offset.
3. **Anchor** — exact string search locates each surface form in the source. This yields
   precise offsets *and* discards anything the model invented, since a fabricated placename
   cannot be found verbatim. The drop count is reported.
4. **Resolve** (`tlcmap-resolve`) — candidates from TLCMap by `containsname`/`fuzzyname`,
   narrowed by state, date and region priors. The model adjudicates using the surrounding
   paragraph, and may answer **"unknown"**. Coordinates are copied from the TLCMap record.
5. **Band and route** — high confidence accepted, low confidence and unknowns sent to a
   reviewer page rather than into the output.
6. **Assemble** (`tlcmap-visualise`, `tlcmap-prepare`) — GeoJSON with mention counts, layers
   by decade, a `textcontexts` sidecar for TLCMap's Full Text view, and an upload-ready layer
   file.
7. **Attribute** (`tlcmap-cite`) — the corpus's own rights, plus every TLCMap layer touched.

### What the skills do **not** do

- **Acquire the corpus.** Trove API keys, library agreements and copyright clearance are the
  researcher's. The skills read what they are given.
- **Perform OCR.** They work with OCR output and can repair obvious errors in context, but
  they do not run an OCR engine over page images.
- **Decide what counts as a mention.** Whether "the colony" or "the diggings" is a place is a
  research judgement; the skill asks rather than assuming.
- **Upload the result.** Until the write API (Phase E) the researcher uploads the prepared
  file through the browser.
- **Replace TLCMap's own geoparser for simple cases.** For a single moderate document,
  uploading the text to TLCMap is better — it produces a proper text layer and the Full Text
  view for free. The skill says so and offers that route first.

### Prerequisites and status

Needs §5.1 ②③, §5.2 ⑨ (match score), §5.3 ⑩⑪ (bulk ID fetch, vocabularies).
**Phase D**, with the resolver.

---

## 2. Place-based corpus enrichment

### Who and what

The reverse direction, and the most immediately useful of the eight. A researcher has a
spreadsheet — shipwrecks, mission stations, police posts, land grants — with place-name
strings in a column and no coordinates. They need a clean, citable geodataset.

This is the use case where the model's contribution is clearest and most measurable, because
the task is exactly "given this evidence, which of these candidate records is the right one,
or is it none of them?"

### Example prompts

> *"This spreadsheet has 400 mission stations with place names but no coordinates. Can you add
> them?"*

`tlcmap-resolve` → `tlcmap-cite`

Other phrasings that should reach the same pipeline:

- *"Add TLCMap IDs and lat/long to the Location column in shipwrecks.csv — the Colony and Year
  columns should help narrow it down."*
- *"Which of these police post names can you match to the gazetteer, and which are too
  ambiguous to call?"*
- *"Geocode this list of historical Australian places."*

**Naming the priors helps a lot.** The second prompt is the better one, because it tells the
resolver which columns are evidence. Without that it has to infer them, and it will ask.

**Where it pushes back.** *"Fill in the ones you can't find with your best guess."* — it
declines. Unresolved rows go to the review bucket with their candidates and map thumbnails,
because a fabricated coordinate in a citable geodataset is worse than a gap.

### A real example

**Layer 475, "Military Mounted Police Posts, Major Nunn's Report"** (Bill Pascoe) — 29
records, all dated 1839, carrying `mounted troopers`, `dismounted troopers`, `officers`,
`division`, `date note` and `location note` as extended data.

Its input was an 1839 military report listing posts by name, with troop numbers and a
division. Turning that into the layer meant resolving each post name to a location — against
a gazetteer where the name may have drifted, the spelling is inconsistent, and several places
share a name.

That is the shape of the problem. A row reading `Post: Boggy Creek, Division: Liverpool
Plains, Officers: 1, Mounted: 12` gives the resolver three pieces of evidence beyond the name
itself: an approximate region, an era, and a feature type. TLCMap holds **94,615 records
matching `containsname=creek`** — the name alone is useless, and the row metadata is what
makes the match decidable.

### The pipeline

1. **Profile the input** — identify the place column, the date column and any columns usable
   as priors (colony, region, district, feature type). The skill reports what it found and
   asks where it is unsure.
2. **Normalise** — strip qualifiers, expand abbreviations, keep the original string intact
   for the audit trail.
3. **Generate candidates** — cascade `name` → `containsname` → `fuzzyname`, filtered by the
   priors. `rapidfuzz` pre-ranks; it does not decide.
4. **Adjudicate** — the model sees the row's full context and the candidate records, and
   returns `{uid | "unknown", confidence, reasoning, evidence_used}`.
5. **Band** — high accepted, medium accepted-with-flag, low and unknown to the review bucket.
   Thresholds are configurable and reported.
6. **Export** — enriched CSV, GeoJSON, a `needs-review.html` reviewer page with candidates and
   map thumbnails per uncertain row, and `decisions.jsonl` recording every judgement with the
   evidence behind it.
7. **Validate** — a post-export check re-reads the cache and **fails the run** if any
   coordinate does not match a fetched TLCMap record exactly.

### What the skills do **not** do

- **Invent coordinates.** Ever. The model ranks records TLCMap returned; it never produces a
  latitude. A place TLCMap does not hold resolves to "unknown", and unknown is a correct
  answer rather than a failure.
- **Resolve non-Australian places.** TLCMap's gazetteers are Australian. Overseas placenames
  need a different gazetteer, and the skill says so rather than forcing a bad match.
- **Clear the review bucket.** Low-confidence rows are a human's to decide. The reviewer page
  makes that fast; it does not make it automatic.
- **Clean the spreadsheet.** Deduplication, merged cells and inconsistent date formats in the
  source are flagged, not silently repaired.

### Prerequisites and status

Needs §5.2 ⑨ (match score — the one signal that cannot be obtained any other way),
§5.3 ⑩ (bulk fetch by ID, or it is one request per candidate) and ⑪ (vocabularies, or a
`state`/`lga` filter cannot be constructed at all).
**Phase D.** Fully supported by today's data; the gold set (§11 of the design) measures it.

---

## 3. Regional knowledge synthesis for fieldwork

### Who and what

An archaeologist, ecologist or oral historian is preparing fieldwork in a region and needs a
consolidated brief: gazetted places, relevant contributed layers, and the rights attached to
each — in a form they can take offline, on a device, with no signal.

### Example prompts

> *"I'm doing fieldwork in the Hunter Valley next month. What does TLCMap have for that area,
> and can I get it onto my GPS?"*

`tlcmap-search` → `tlcmap-analyse` → `tlcmap-visualise` → `tlcmap-cite`

Other phrasings that should reach the same pipeline:

- *"Build me a briefing for the area around Wollombi — everything gazetted, plus any
  contributed layers, grouped by type."*
- *"Which TLCMap layers cover the Coorong between 1840 and 1880?"*
- *"What's within 20km of these coordinates, and who do I credit for it?"*

**Where it pushes back.** *"Include everything, don't worry about the licences."* — the
attribution block is not optional and warnings travel with the data (§3.3). It will produce
the brief and it will carry each layer's rights statement into it.

**What it says about its own limits.** The brief states its coverage: it reports what TLCMap
holds for that region, which is not the same as what is there.

### A real example

A real query against the Hunter Valley, run on 2026-09-24:

```
GET /places?format=json&bbox=150.8,-33.1,151.4,-32.6&searchpublicdatasets=on
  → 273 records across 36 distinct contributed layers
```

Thirty-six layers, from weather stations (461) to convict landscapes (1270), that a
researcher had no way of knowing existed. **None of them could be found through the layer
catalogue**, because only 4 of 2,118 public layers declare a bounding box
([DESIGN.md §2.2](./DESIGN.md#22-the-catalogue-measured)). This use case is the
clearest argument for §5.1 ①, the catalogue extent facet.

The brief this produces groups those 273 records by type — water sources, stations, heritage
sites, language boundaries — cites each by TLCMap UID, and carries each contributing layer's
licence and warning alongside its data.

### The pipeline

1. **Define the region** — from a placename, a bbox, or a polygon. Longitude first, ring
   closed; the skill handles the convention so the researcher does not have to.
2. **Discover layers** (`tlcmap-search`) — one faceted catalogue request once §5.1 ① exists.
3. **Retrieve** — gazetteer records plus the contributed layers that intersect, cached and
   checksummed.
4. **Deduplicate** (`tlcmap-analyse`) — the same waterhole in three layers is one feature with
   three sources, not three pins.
5. **Group and characterise** — by feature term, record type, date range and source layer,
   with counts computed in code.
6. **Draft the brief** — prose grouped by theme, every claim citing a TLCMap UID, every number
   taken from the computed statistics rather than from the model.
7. **Package for the field** (`tlcmap-visualise`) — KML and GPX for GPS devices, a QGIS project
   with styling, and a markdown or PDF briefing document.
8. **Rights** (`tlcmap-cite`) — every layer's creator, licence, citation and **warning**,
   reproduced verbatim.

### What the skills do **not** do

- **Judge cultural appropriateness of visiting a site.** Layers carry warnings for a reason;
  the brief surfaces them prominently and stops for a human decision where a layer indicates
  Indigenous knowledge or sensitive sites.
- **Obtain permits or landholder permission.** Out of scope entirely.
- **Guarantee completeness.** It reports what TLCMap holds for that region, which is not the
  same as what exists there. The brief states its own coverage limits.
- **Fetch non-TLCMap sources.** AIATSIS boundaries, cadastral data and heritage registers are
  the researcher's to supply; the skills will overlay them but do not go and get them.
- **Work offline.** Building the brief needs the network; the output is what goes offline.

### Prerequisites and status

Needs §5.1 ① above all — without the extent facet, regional discovery does not work.
**Phase A: this is the vertical slice.**

---

## 4. Indigenous / colonial name co-mapping

### Who and what

Produce a publication figure showing Indigenous placenames alongside colonial gazetteer names
for a defined region, with correct attribution to every contributing layer.

**This is the use case where the skills deliberately do less.** The technical work is easy;
the ethical work is not, and the design routes it to a human rather than automating it.

### Example prompts

> *"I need a publication figure showing Indigenous and colonial place names for the Pilbara.
> Which layers can I use, and what do I need to credit?"*

`tlcmap-search` → `tlcmap-cite` → **human decision** → `tlcmap-visualise`

Note where the human step sits: before any data is fetched, not after the map is drawn.

Other phrasings that should reach the same pipeline:

- *"What does TLCMap have on Aboriginal place names in WA, and what are the access
  conditions?"*
- *"Check the licence and attribution on layers 258 and 1091 before I use them."*
- *"Who do I need to contact about using this layer?"*

**Where it stops.** *"Merge all the Aboriginal place name layers for WA into one map."* — the
skill does not proceed. It reports each layer's licence and warning verbatim and asks how you
want to handle them. For layer 258 that means showing you *"Do not use without permission."*
and *"This layer contains historical information about Aboriginal people that may be
distressing. It contains names of people who have passed away."* — and then waiting.

It does not proceed on silence, and it will not summarise a warning into a shorter one.

### A real example

Two layers make the point better than an argument would.

**Layer 258, "WA Journey Ways — Aboriginal Camps Around WA"** (Dr Francesca Robertson, Dr Noel
Nannup, Alison Nannup):

> **Licence:** *"Do not use without permission."*
> **Warning:** *"This layer contains historical information about Aboriginal people that may
> be distressing. It contains names of people who have passed away."*

**Layer 1091, "Ngarinyman materials"**:

> **Licence:** *"Closed (subject to the access condition details)"*

Neither licence is machine-readable, and both say something a parser would get catastrophically
wrong. A naive integration looking for a recognised identifier finds none and treats the layer
as unrestricted. A slightly cleverer one might match "Closed" and guess. **Both are wrong, and
the second is worse for being confident.** This is why
[DESIGN.md §3.3](./DESIGN.md#33-attribution-travels-permission-is-never-guessed) forbids
parsing these fields into a permission decision at all.

A third layer shows what good contributed data looks like here. **Layer 2477, "Southern
Queensland War and Resistance"** (Ray Kerkhove and Bill Pascoe) carries `AboriginalPlaceName`
and `LanguageGroup` as fields alongside the colonial record, plus a `CorroborationRating` on
each event — the contributor encoding their own confidence. Its warning reads *"Colonial
violence. Linked sources and citations may contain racist language and attitudes of the
time."* It has **no licence field at all.** It is part of a series with layers 2509 (Far North
Queensland) and 2754 (Coorong, Adelaide and Yorke).

### The pipeline

1. **Find candidate layers** (`tlcmap-search`) — by region and keyword.
2. **Read the rights first** (`tlcmap-cite`) — `?metadata` on each layer before any data is
   fetched. Creator, licence, rights, citation, warning.
3. **Stop.** Where metadata, keywords or warning indicate Indigenous knowledge, cultural
   material or sensitive sites, the skill presents what the contributor said and **asks the
   researcher how to proceed**. It does not decide, and it does not proceed on silence.
4. **Compose** — only after that decision, assemble the multilayer or merged extract.
5. **Export for cartography** (`tlcmap-visualise`) — GeoJSON styled for QGIS, with the
   Indigenous and colonial name sets kept as distinct layers rather than merged into one
   field.
6. **Attribution block** — every contributing layer's creator, citation and warning, formatted
   for a figure caption and a methods section.

### What the skills do **not** do

- **Decide whether you may use a layer.** They tell you what the contributor wrote. The
  decision is yours, and where a licence says "do not use without permission", obtaining that
  permission is a conversation with a person, not a field lookup.
- **Strip or summarise warnings.** Warnings travel into every derived output, including
  figures. If data is republished, the warning is republished with it.
- **Merge Indigenous and colonial names into one authority.** They are different knowledge
  systems with different provenance; flattening them is a cartographic and ethical error.
- **Aggregate silently.** Combining layers can reveal sensitive site locations that no single
  layer disclosed. `tlcmap-analyse` flags composition across layers carrying warnings.
- **Speak for communities.** No skill drafts text asserting cultural meaning. It can format,
  cite and attribute; it does not interpret.

### Prerequisites and status

Needs §5.3 ⑮ (a structured licence identifier alongside the free text) to *assist* — never to
decide. **Human-gated by design at every phase.** CARE principles are stated in the skills'
own instructions, not only in this document
([DESIGN.md §12](./DESIGN.md#12-risks-and-ethics)).

---

## 5. Toponym pattern analysis

### Who and what

Quantitative historical geography. *"What is the spatial distribution of placenames ending in
-ville, -town, or derived from Indigenous languages across NSW, and how does it correlate with
settlement waves?"*

This is the use case with the least model involvement and the most statistics — and it is the
clearest demonstration of the determinism boundary, because every number in the output comes
from pandas and PostGIS while the model writes the code and the prose around them.

### Example prompts

> *"How are place names ending in -ville distributed across NSW compared with
> Indigenous-derived names, and does that track the settlement frontier?"*

`tlcmap-search` (harvest) → `tlcmap-analyse` → `tlcmap-visualise`

Other phrasings that should reach the same pipeline:

- *"Harvest every gazetteer record with 'creek' in the name and show me density by region."*
- *"Classify NSW place names by likely origin, give me the spatial statistics, and include the
  notebook."*
- *"Is there a pattern to where -town names appear?"*

**What the model does and does not do here.** It proposes the classification heuristics and
writes the code; pandas assigns the labels and computes the numbers. Every figure traces to a
line in the emitted notebook.

**Where it pushes back.** *"Just tell me which names are Aboriginal in origin."* — it produces
candidates with a confidence and an ambiguity bucket. Attributing derivation is a scholarly
claim, and for Indigenous-language toponyms it is a question for the relevant community rather
than a suffix rule.

### A real example

The gazetteers are large enough to make the patterns real. A single generic element:

```
GET /api?format=json&containsname=creek&per_page=100&page=1
  → total: 94,615
```

Ninety-four thousand records for one word. That is well past the 5,000-record ceiling on
`/places`, which is why this use case harvests through `/api` — and why §5.1 ③, extending
`/api` to contributed layers, matters if the analysis is to include them.

The classification step is where the model earns its place: given a sample of names, it
proposes and refines the heuristics (suffix patterns, known-toponym lists, morphological
markers of Indigenous-language origin), and writes the classifier. It does not assign the
labels itself at scale — the code does, reproducibly, and the boundary cases go to review.

### The pipeline

1. **Harvest** (`tlcmap-search`) — page through `/api` following `next` until exhausted.
   Cached and checksummed; a 94,615-record harvest is run once, not per session.
2. **Develop classification heuristics** — the model drafts rules from a sample, the
   researcher reviews them, and they are applied as code.
3. **Classify** — in pandas, deterministically, with an ambiguity bucket.
4. **Join** external data — settlement phases, land-grant dates, railway openings. Supplied by
   the researcher.
5. **Compute** (`tlcmap-analyse`) — kernel density, nearest-neighbour statistics, Ripley's K,
   per-decade and per-LGA breakdowns. All in code.
6. **Visualise** (`tlcmap-visualise`) — choropleths, density surfaces, small multiples by
   period.
7. **Draft** — results prose written strictly from the computed statistics file, with the
   exclusions stated.
8. **Emit the notebook** — the analysis ships with the means to reproduce it.

### What the skills do **not** do

- **Produce the statistics.** The model writes the code; pandas computes the numbers. A figure
  in the output traces to a line in the emitted notebook.
- **Supply the settlement data.** The correlation half of the question needs a historical
  dataset TLCMap does not hold.
- **Assert linguistic origin authoritatively.** A name's derivation is a scholarly claim.
  The classifier produces candidates and a confidence; attributing origin is the researcher's,
  and for Indigenous-language toponyms it is a question for the relevant community, not a
  regex.
- **Hide the gaps.** Gazetteer coverage is uneven by state and era. The analysis reports what
  it could not see.

### Prerequisites and status

Works today for gazetteer-only analysis. Needs §5.1 ③ to include contributed layers.
**Phase B.**

---

## 6. Environmental / event history overlay

### Who and what

Map historical bushfire, flood or drought accounts against modern hazard models, to ask
whether historical events track with modelled risk zones — a question with obvious relevance
to current planning, and one that needs the historical record to be spatial before it can be
asked at all.

### Example prompts

> *"Take the 19th century bushfire layer and tell me whether those fires line up with the
> high-risk zones in this hazard raster."*

`tlcmap-search` → `tlcmap-cite` → `tlcmap-analyse` → `tlcmap-visualise`

Other phrasings that should reach the same pipeline:

- *"Map layer 170 by decade — does the seasonality change over the century?"*
- *"Overlay these historical flood reports on the catchment boundaries in this shapefile."*
- *"How many of these events fall inside the modelled risk area?"*

**The caveat arrives before the analysis, not after.** Ask the second prompt and the skill
surfaces the layer's own warning first: *"Recorded dates correspond to date of publication and
not necessarily to the date on which a bushfire occurred."* A seasonality claim built on 7,454
publication dates needs that addressed, and the skill will apply an offset you specify — it
will not invent one.

**Where it pushes back.** *"So what does this tell us about climate change?"* — it gives you
the containment statistics and states the confounders, principally that colonial newspapers
chose what to report and that choice was not spatially uniform. The causal claim is yours.

### A real example

**Layer 170, "19th Century Australian Bushfire Reporting"** again — 7,454 dated fire reports
from 1824 to 1899, drawn from newspapers, CC BY. It is exactly the historical half of this
overlay, and it already exists.

Its warning carries the caveat that makes or breaks the analysis:

> *"Recorded dates correspond to date of publication and not necessarily to the date on which
> a bushfire occurred."*

A naive temporal analysis treats 7,454 publication dates as 7,454 event dates. The lag between
a fire and its report in a colonial newspaper could be days or weeks, and varied by distance
from the press. **Any seasonality claim drawn from this layer without addressing that is
wrong**, and the skill's job is to surface the warning at the point the researcher asks a
temporal question — not to bury it in a citation.

Note also that the layer exceeds the 5,000-record ceiling, so it must be read as a layer feed
rather than through a search.

### The pipeline

1. **Acquire the historical events** — either an existing layer like 170, or extracted from
   text via use case 1.
2. **Resolve places** (`tlcmap-resolve`) where the source has names rather than coordinates.
3. **Surface the caveats** (`tlcmap-cite`) — the layer's warning is presented before analysis
   begins, not after.
4. **Overlay** — hazard rasters, catchment boundaries or fire-history polygons, supplied by
   the researcher, joined in `geopandas`/`rasterio`.
5. **Compute containment statistics** (`tlcmap-analyse`) — what proportion of historical
   events fall inside modelled high-risk zones, by period and region.
6. **Visualise** (`tlcmap-visualise`) — an interactive map with a time slider, plus
   journal-ready static figures.
7. **Report exclusions** — how many records lacked coordinates, how many lacked usable dates,
   how many fell outside the raster extent.

### What the skills do **not** do

- **Supply the hazard models.** BoM, state fire service and catchment datasets are the
  researcher's to obtain, with their own licences.
- **Reproject silently.** CRS mismatches are flagged, not guessed at.
- **Correct the publication-date lag.** The skill surfaces the caveat and will apply an
  offset the researcher specifies. It will not invent one.
- **Make the causal claim.** Correlation between historical accounts and modelled risk is a
  finding to interpret, and the sampling bias in what colonial newspapers chose to report is
  a serious confounder the analysis must state.

### Prerequisites and status

Needs §5.2 ⑥ (`udateend`) and ⑦ (`include_undated`) for honest temporal work.
**Phase D** where text extraction is involved; **Phase B** where an existing layer is used.

---

## 7. Comparative / longitudinal mapping of a single concept

### Who and what

A research group tracks one theme — mission stations, police camps, pastoral leases — across
sources and over years, and wants a single canonical TLCMap layer they keep updating as new
sources arrive. The layer becomes citable infrastructure for the group rather than a
one-off export.

### Example prompts

> *"I want a TLCMap layer of colonial police camps that I keep adding to as I find new
> sources. Set that up."*

`tlcmap-resolve` → `tlcmap-prepare` → *(write API — blocked)* → `tlcmap-visualise`

Other phrasings that should reach the same pipeline:

- *"I've found 12 more mission stations — add them to my existing layer without duplicating
  what's already there."*
- *"What changed in my layer since last month?"*
- *"Re-run this every week and update the map."*

**It tells you it is blocked at the start, not at the end.** The first prompt gets an honest
answer immediately: the skill can build and validate the layer file and produce a dry-run diff
against the current layer, but it cannot write to TLCMap, so you upload through the browser.
The third prompt gets a clearer refusal still — without idempotent upsert, a scheduled run
that fails halfway and retries duplicates the layer.

**What it can do today.** The second prompt works well: it resolves the new records, diffs
them against the live layer field by field, and hands you an upload file containing only what
changed, with `ghap_id` set so TLCMap updates rather than appends.

### A real example

**Layer 475, "Military Mounted Police Posts, Major Nunn's Report"** is a snapshot of exactly
this theme at one moment: 29 posts, all 1839, from a single report. The longitudinal version
would extend it with every subsequent report, dispatch and muster roll, each new record
carrying its own source and date, the layer growing while its identity and citation stay
stable.

The **"War and Resistance"** series — layers 2477, 2509 and 2754, covering Southern
Queensland, Far North Queensland and the Coorong — shows the same pattern at a larger scale:
a sustained research programme publishing as it goes, with a shared field vocabulary
(`CorroborationRating`, `LanguageGroup`, `Period`, `Stage`) across layers.

### The pipeline

1. **New sources arrive** — a reference manager, a shared drive, a scheduled search.
2. **Extract candidate records** — use case 1 or 2 depending on whether the source is text or
   tabular.
3. **Classify against the theme taxonomy** — the model assigns each candidate to the group's
   own categories, with the taxonomy held in the repository rather than in the prompt.
4. **Resolve and validate** (`tlcmap-resolve`, `tlcmap-prepare`) — including a dry-run diff
   against the current layer: added, changed field-by-field, unchanged, orphaned.
5. **Upsert into the managed layer** — *the blocked step, see below.*
6. **Regenerate views** (`tlcmap-visualise`) — the map and timeline refresh from the live feed.
7. **Provenance** — every appended record carries its source, extraction date and the pipeline
   version that produced it, in extended data.

### What is blocked

**This use case does not work today**, and the reason is specific:

- **No write API.** Step 5 cannot run. `tlcmap-prepare` produces a validated, upload-ready
  file and a checklist, and a human uploads it through the browser. That is a real workflow,
  but it is not the unattended one this use case describes.
- **No change feed.** There is no `updated_since` and no `ETag`, so a scheduled re-run cannot
  ask what changed — it must refetch everything and diff locally.
- **No idempotency.** Without upsert on a client-supplied key, a cron job that fails halfway
  and retries duplicates records. This is the single most important requirement in the write
  API proposal, more than any individual route.

### What the skills do **not** do

- **Own the taxonomy.** The research group defines what counts as a "police camp". The skill
  applies that definition and flags the boundary cases.
- **Resolve disagreement between sources.** Where two sources place the same station
  differently, both are recorded with their provenance and the conflict is surfaced.
- **Run unattended on first setup.** The first pass is reviewed by a human; only a pipeline
  with a measured accuracy figure should be trusted on a schedule.

### Prerequisites and status

**Blocked on [§5.4, the write API](./DESIGN.md#54-the-write-api)**, plus §5.3 ⑭
(conditional requests and `updated_since`). **Phase E.**

---

## 8. Teaching / public-facing storymaps

### Who and what

Build a student-facing interactive story — an expedition, a voyage, a campaign — that walks
through a journey with primary-source excerpts pinned to real locations, comprehension
questions, and accessible alt text.

The lowest-risk use case in the set, and the best shop window: the output is public, visual
and immediately legible to people who will never read this document.

### Example prompts

> *"Build a Year 9 lesson around layer 450 — the early Sydney exploration one — with the route
> on a map and comprehension questions for each stop."*

`tlcmap-search` → `tlcmap-visualise` → `tlcmap-cite`

Other phrasings that should reach the same pipeline:

- *"Make me an embeddable timeline of the De Vergulde Draeck wreck layer."*
- *"Turn this expedition layer into a story I can put on the department site."*
- *"I need a map of this journey with the stops in date order and alt text for each image."*

**Stating the year level matters**, because it sets the reading level of the drafted
annotations. Without it the skill asks.

**Where it pushes back.** *"Write the historical background for each stop."* — it drafts it and
marks it as requiring a teacher's check, because the annotations are a starting point rather
than verified history. It will also raise the framing question on an expedition layer: the
journey went through Country that was already known and named, and layer 450 is titled *Early
Land Exploration around Sydney / Gadigal and Darug Country* for that reason. How to teach it
is the educator's call.

### A real example

TLCMap holds a series of early-exploration layers already shaped for this, several of which
name Aboriginal Country alongside the colonial expedition — which is itself a teaching point:

| Layer | |
| --- | --- |
| **450** | Early Land Exploration around Sydney / Gadigal and Darug Country — 7 records, 1788–1793, `CCBY`, with an `Explorer` field |
| **447** | d'Entrecasteaux in Van Diemen's Land (Bill Pascoe) |
| **456** | James Stirling early land exploration of Noongar country / Swan River |
| **449** | Collett Barker exploration of Kaurna and Ngarrindjeri country |
| **152** | The Wreck of the Ship "De Vergulde Draeck" on the Southland — 46 records, built from a Wikisource text |

Layer 450's seven dated waypoints between 1788 and 1793 are enough for a complete lesson: a
journey view drawing the route in date order, a waypoint per stop, and the `Explorer` field
distinguishing who went where.

TLCMap Views does the map for free. `latest/journey.html` needs `LineString` features, which
a layer feed generates with `line=time`; `latest/timeline.html` needs `udatestart` and
`udateend`, which layer feeds carry correctly — search feeds do not, because of the
`udateend` bug (§5.2 ⑥).

### The pipeline

1. **Select the layer or build one** (`tlcmap-search`, or use case 1 from a journal text).
2. **Order the waypoints** — by date for a chronological story, by `dataset_order` for a known
   route.
3. **Draft annotations** — student-level commentary per waypoint, pitched to a stated year
   level, drawn from the record's description and the linked source.
4. **Generate comprehension questions** — per waypoint and per section.
5. **Write alt text** — for every map and image, as a first-class output rather than an
   afterthought.
6. **Build the embed** (`tlcmap-visualise`) — a TLCMap Views URL with the `load` parameter
   **percent-encoded** (or the feed's own query string is swallowed by the viewer), or a
   static site scaffold with the map embedded.
7. **Attribute** (`tlcmap-cite`) — layer creator and citation on the page, not buried.

### What the skills do **not** do

- **Verify historical claims in the annotations.** Drafted prose is a starting point for a
  teacher or curriculum writer to check, and the skill says so in the output.
- **Publish anywhere.** It produces files. Where they go is the user's decision.
- **Handle the colonial framing on the teacher's behalf.** An expedition layer describes a
  journey through Country that was already known and named. The skill can surface Indigenous
  place names where a layer provides them and will flag the framing question; deciding how to
  teach it is the educator's.
- **Guarantee accessibility compliance.** It generates alt text and semantic structure;
  a WCAG audit is a separate exercise.

### Prerequisites and status

Works with today's API. Improved by §5.2 ⑥ (`udateend`) so timelines can be built from
searches as well as layers. **Phase B.**

---

## Additional capabilities

Not in the original brief, but cheap, high-value, and directly useful to TLCMap's own
community. See [DESIGN.md §8](./DESIGN.md#8-use-cases).

### Example prompts for these

- *"Check layer 170 for data quality problems before I cite it."* → layer health report
- *"Why did my upload fail?"* → health report run against the file rather than a layer, listing
  **every** date error at once rather than the first
- *"Are there duplicate places across these three layers?"* → cross-layer duplicate detection
- *"How accurate is this place matching?"* → the evaluation harness, reporting precision,
  recall and abstention rate rather than a reassurance
- *"Give me something I can re-run next year when there's more data."* → a saved-search URL
  rather than a frozen extract

### Layer health report

Point `tlcmap-prepare` at any public layer and get a data-quality assessment: unparseable
dates, coordinates that are implausible for the stated state, duplicate records, extended-data
headings that were mangled on import (`Area m2` → `Area m`, `Catalogue no. 3` →
`Catalogue no ` with a significant trailing space), collisions with built-in field names, and
missing licence or citation.

**Why it matters:** the import is strict — one unparseable date aborts an entire file — and
its failures are opaque. This turns that into a list a contributor can act on.

**Real example:** layer 170's own warning documents its coordinate imprecision, and its
placename tail contains `GREECE` and `EGYPT`. A health report surfaces that class of issue
without anyone having to read 7,454 records.

**Status:** deliverable now, and the lowest-risk first demonstration in the project.

### Cross-layer duplicate detection

When composing a multilayer, find records that are the same place in different layers, so a
merged map does not triple-count. Combines TLCMap's closeness analysis with local matching.

**Real example:** the Hunter Valley bbox returns 273 records across 36 layers; several of
those layers plausibly hold the same townships and watercourses.

### Resolution evaluation harness

A gold set of hand-checked placename → UID pairs, scoring precision, recall and — most
importantly — **abstention rate**. A resolver that correctly says "unknown" is worth more than
one that guesses well.

**Why it matters:** it is the difference between a demonstration and a proof. Every claim made
for use cases 1, 2 and 6 rests on a resolution accuracy figure, and without a gold set that
figure does not exist.

### Reproducible notebook emission

Any analysis ships with the notebook that produced its numbers. The researcher can re-run it,
a reviewer can check it, and the model is visibly not the source of the statistics.

### Saved-search awareness

A TLCMap saved search is a stored query, not a stored result — re-running it picks up records
added since. Where a research question is ongoing rather than settled, the skills prefer
emitting a re-runnable query URL over a frozen extract.
