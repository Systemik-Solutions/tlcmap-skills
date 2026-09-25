# TLCMap AI — design

**Status:** design for build · **Date:** 2026-09-25

Companion document: **[USE-CASES.md](./USE-CASES.md)** — the eight research use cases in full,
with example prompts and worked examples from real TLCMap data.

---

## Contents

| | |
| --- | --- |
| [1. What we are building](#1-what-we-are-building) | The proposition |
| [2. The platform today](#2-the-platform-today) | What the API offers, and what breaks |
| [3. Design principles](#3-design-principles) | Eight rules the rest follows from |
| [4. Architecture](#4-architecture) | Three layers, and why in this order |
| [5. Layer 1 — the API](#5-layer-1--the-api) | The enhancement programme |
| [6. Layer 2 — the MCP server](#6-layer-2--the-mcp-server) | Tool surface and design rules |
| [7. Layer 3 — the skills](#7-layer-3--the-skills) | Workflow and judgement |
| [8. Use cases](#8-use-cases) | Mapping to USE-CASES.md |
| [9. The vertical slice](#9-the-vertical-slice) | The proof of concept |
| [10. Plan](#10-plan) | Phases A–E |
| [11. Measurement](#11-measurement) | How we know it works |
| [12. Risks and ethics](#12-risks-and-ethics) | |
| [13. Open questions](#13-open-questions) | |

---

## 1. What we are building

TLCMap holds curated Australian historical place data that nothing else holds: the ANPS
Gazetteer's colonial-era name variants, two thousand contributed layers, and the texts that
have been geoparsed into them. It publishes all of it over plain HTTP with no key and no client
library.

What it does not have is anything that turns a research question into a query, or a query
result into an answer. That gap is the opportunity. A researcher who knows exactly which of
`name`, `containsname` and `fuzzyname` to use, that `sort` silently deletes every undated
record, and that `limit` returns a *random sample* rather than the first N, can get a great deal
out of TLCMap today. Nobody knows those things without reading the developer documentation end
to end.

**The proposition:** open TLCMap to AI agents in three layers — an API that is honest about what
it returns, an MCP server that exposes it as discrete tools to any agent platform, and a set of
skills that shape those tools into research workflows — so that a natural-language research
request produces a citable, reproducible result.

Each layer is useful on its own. The API work improves every existing TLCMap client, including
the browser interface. The MCP server reaches Claude Desktop, a chatbot on tlcmap.org and any
other agent framework, with no skills installed. The skills add judgement, ethics gating and
finished outputs on top.

---

## 2. The platform today

Grounded in the developer documentation and in live checks against production, 2026-09-25.

### 2.1 What the API offers

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

### 2.2 The catalogue, measured

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
them (78, 284, 1341, 1333); all four fields are null on the rest, and the per-layer feed's
`?metadata` has the same gap. The information exists but is not exposed in aggregate. Layer 152,
for instance:

```
46 records, every one with coordinates
DECLARED bbox : null, null, null, null
ACTUAL bbox   : lon 4.7978..116.2544   lat -34.3568..53.0548
```

So **layer discovery by region is impossible from the catalogue**, and there is no `bbox`
parameter on `/layers/json` to do it server-side either. That blocks
[use case 3](./USE-CASES.md#3-regional-knowledge-synthesis-for-fieldwork) and half of
[use case 4](./USE-CASES.md#4-indigenous--colonial-name-co-mapping). The fix is one
`GROUP BY dataset_id` with `ST_Extent` exposed as a catalogue facet — **§5.1 ①**, the highest
value change in the programme, which fixes the same gap in the browser interface at the same
time.

::: warning
**Compute the extents; do not promote the declared fields.** The four that exist cannot be
trusted either: layer 284 declares `latitude_from: -10, latitude_to: -30` — "from" is *north* of
"to" — and layer 1341 declares `longitude_to: 182.167965`, outside the valid ±180 range. Keep
the contributor-supplied values as a separate hint, and derive the facet from geometry.
:::

### 2.3 Behaviours that break a naive client

Every one of these is a silent failure — wrong data, not an error. This table is the evidence
base for §5: each row is something the API should stop doing, not something a client should
learn to survive (§3.8).

| Trap | Consequence |
| --- | --- |
| `limit=N` takes a **random sample** (`shuffle()` then `take()`) | Two identical requests return different records. Never use it to truncate a result set. |
| `sort=anything` drops records with no start *or* end date | A 216-record search returns 55 with `sort=title`. Same for `line=time`. |
| Dated searches exclude undated records entirely | Most gazetteer records are undated; adding a date bound can cut results by an order of magnitude. |
| >5,000 matches → `302` to `/maxpaging` (HTML) | A client following redirects gets HTML with a `200` where it expected GeoJSON. `paging=1` does not help — the ceiling is checked against the total. |
| Private or missing layer → `200` with a FeatureCollection, no `features` | Detect by the **absence of `features`**; the warning key is misspelled `warnnig` here. |
| Unparseable `extended_data` expression is **discarded silently** | See below. The most dangerous behaviour in the API. |
| Search output sets `udateend = udatestart` | Every record looks like a single instant on a timeline. Layer feeds are correct. |
| `name` matches `title` only, never `placename` | The parameter people reach for first is the one that misses. |
| Extended data is merged into `properties` | A contributor's column named `description` silently replaces the built-in one. |
| Import sanitises field headings | `Catalogue no. 3` → `Catalogue no ` (trailing space significant); `Area m2` → `Area m`. |
| One unparseable date aborts an entire upload | No partial import, and only the first error is reported. |
| `bbox`/`polygon` take **longitude first**; polygon rings must be closed | Silently wrong area, or a PostGIS error. |
| DBScan `distance` is divided by 100 and passed as **degrees** | Labelled km in the UI. `distance=100` ≈ 111 km N–S, less E–W. |
| Missing analysis parameter → `500` | Send every parameter, including empty ones (`withinRadius=`). |
| `basicstatistics/json` returns **geometry only** | The actual statistics exist only in the HTML page. |
| `id=` redirects to the path form | So a client must follow redirects — but following redirects on an oversized query yields `/maxpaging` HTML with a `200`. |
| No rate limiting, no versioning | Politeness and caching are the client's responsibility. |

The `extended_data` case deserves its own illustration, because it is easy to underestimate.
Layer 461 holds 17,917 records:

```
extended_data=Years > 50    →    612 records      (filter applied)
extended_data=Years>50      →  17,917 records     (filter silently discarded)
```

A single missing space returns 29× the data as though it were a filtered result. Here it
happens to exceed the ceiling and fail loudly-but-wrongly as HTML; on any layer under 5,000
records it returns a plausible, complete, entirely unfiltered answer that no client can
distinguish from a correct one.

### 2.4 What does not exist

- **Any write.** No create, update or delete.
- **Private layers over the API.** No token exposes them to a script, by design.
- **Catalogue filtering or search.** Fetch 2.7 MB and filter locally.
- **Statistics as numbers.** Only as a rendered page.
- **Change feeds.** No `updated_since`, no `ETag` — a scheduled re-run cannot ask what changed.
- **Versioning.** The API documents and serves the current production release.

---

## 3. Design principles

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

Enforced mechanically, not by instruction: every exported coordinate is validated against the
cached response it came from, and an export whose coordinates do not trace back to a fetched
record fails. Every model judgement is written to an audit log with the evidence it was shown.

### 3.2 Provenance by construction

Every artefact carries how it was made: the query, the fetch timestamp, a checksum of the
response, the TLCMap UID of every record, and — for model judgements — the candidates
considered and the confidence. A result a reviewer cannot retrace is not a research output.

### 3.3 Attribution travels, permission is never guessed

`license` and `rights` are free text. A layer may say "CC BY 4.0", or a sentence, or nothing.
**No layer of this system ever parses those fields into a yes or no.** They are surfaced
verbatim, alongside `creator`, `citation` and — critically — `warning`, the field contributors
use for cultural sensitivity notices. If data is republished, the warning is republished with
it. Layers relating to Indigenous knowledge are routed to a human decision, never an automated
one (§12).

Of 2,118 public layers, 1,414 carry no licence at all. Absence of a licence is not permission,
and the system says so rather than defaulting to open.

### 3.4 Report what was excluded

The API reports its own failures honestly once §5.1 ② lands — a rejected filter is a `400`, a
private layer a `403`, an oversized result a `413` — so clients check status codes rather than
sniffing content types. That is the whole of error handling.

What remains is not error handling but honesty about scope, and it stays:

- **Exclusions are reported, always.** A dated search leaves out undated records; a sorted one
  leaves out records with no date to sort by. "216 records, 55 of them dated" is the truthful
  version of a temporal claim, and every summary carries it.
- **Contributor-supplied metadata is evidence, not fact.** Declared extents can be malformed
  (§2.2), licences are free text (§3.3), extended-data values are untyped strings. These are
  properties of community-contributed data, not API defects, and no endpoint will fix them.
  Validate, surface, never silently repair.
- **Model output is verified against the cache** (§3.1).

### 3.5 Handles and files, not context

TLCMap responses are large — a 2.7 MB catalogue, feeds of thousands of features. Nothing of
that size passes through the model. Tools return summaries and resource handles; scripts write
to disk and report what they wrote.

### 3.6 Courtesy is a feature

One machine serves both the application and the API, and there is no rate limiting. Caching is
not an optimisation but a requirement — and it doubles as the provenance store and the
reproducibility snapshot.

### 3.7 Reproducibility as an output

Where an analysis produces numbers, the skill emits the notebook or script that produced them
alongside the figure. The researcher can re-run it, a reviewer can check it, and the model is
visibly not the source of the statistics.

### 3.8 Fix it upstream

TLCMap owns the API and this capability both, and there is exactly **one deployment** —
tlcmap.org. So where the agent layers need something the API does not do, the API changes.

This design contains **no workarounds, no fallbacks and no feature detection**. A client-side
workaround would be code written, tested, documented and maintained indefinitely, to defend
against behaviour we control, on behalf of an installed base of one — while every other TLCMap
client keeps hitting the same wall. Layer discovery by region is the clearest case: either a
2,118-request harvest plus a recursive bbox-subdivision algorithm in every client, or one
`GROUP BY dataset_id` on the server that fixes the browser interface at the same time.

§5 is accordingly a list of **prerequisites**, not of gaps to route around. Each item names what
depends on it and the phase it is needed by. The upper layers are written against the API as it
will be, and a capability whose prerequisite has not landed does not ship yet — it does not ship
with a workaround.

One boundary this does not cross: §5.5 sets out what genuinely belongs in the client. Judgement,
disambiguation and ethics do not become endpoints.

---

## 4. Architecture

### 4.1 Three layers

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

| | Layer 1 — API | Layer 2 — MCP | Layer 3 — Skills |
| --- | --- | --- | --- |
| **Owns** | Data, query semantics, correctness | Tool surface, agent ergonomics | Workflow, judgement, output |
| **Makes model calls** | No | **No** | Yes |
| **Knows about agents** | No | Tool shapes only | Entirely |
| **Structurally reversible?** | **Hardest — unknown clients, no negotiation** | Easy — tools are re-read each session | Easy — rewrite freely |
| **Semantically reversible?** | Hard | **Hardest — changes are invisible** | Easy |
| **Built in** | PHP / Laravel | PHP / Laravel | Markdown + Python |

The two reversibility rows say different things.

**Structurally, the API is hardest to change.** It has real clients today — scripts, QGIS users,
embeds — none of which re-read anything or adapt. MCP is the opposite: the protocol is
self-describing and the tool list is fetched per session, so a renamed tool or a new parameter is
visible and a capable agent adapts without anyone shipping a fix.

**Semantically, MCP is hardest**, and this is the row to act on. An agent remediates *errors*; it
does not remediate *meanings that changed quietly*. If `date_from` begins excluding undated
records where it used to include them, or `bbox` starts matching declared extents rather than
computed ones, nothing fails — the tool returns a well-formed, plausible, wrong answer. That is
§2.3's failure class reintroduced one layer up. The server is the smallest layer and the one
whose mistakes are quietest, so it deserves the most design scrutiny per line — and the
discipline it needs is about meaning, not about freezing names (§6.5).

### 4.2 Why this order

Building API → MCP → skills, rather than the reverse:

- **It matches the dependency order.** §3.8 makes API changes prerequisites; the delivery order
  should agree with that rather than fight it.
- **It avoids building the same thing twice.** A skills-first approach needs a Python HTTP
  client, query builder, response normaliser and catalogue cache — all of which an MCP server
  later makes redundant. Built in this order, that layer is never written.
- **Each layer stands alone.** If work stops after two layers, TLCMap still has a better API and
  an AI gateway serving every agent platform.
- **It plays to the team in the right order.** The lower two layers are PHP and Laravel in a
  codebase the team owns; the unfamiliar part comes last, on solid ground.

The risk this creates is designing an API and a tool surface for workflows nobody has built yet —
and the classic failure mode is specific: tool surfaces designed without workflow experience
mirror the data model rather than the task. §9 is the answer: one vertical slice through all
three layers, first.

### 4.3 Packaging and distribution

Three artefacts, in two places.

**The API and the MCP server** live in the TLCMap application. Same stack, same deployment, same
database, and at the write phase the same authentication and permission model.

**The skills** live in this repository, built to the **Agent Skills open standard**
([agentskills.io](https://agentskills.io)) — a skill is a folder containing `SKILL.md` with
`name` and `description` frontmatter, optionally bundling `scripts/`, `references/` and
`assets/`. Around forty-five clients read that format, including Claude Code, Codex, Gemini CLI,
Cursor, GitHub Copilot and VS Code, Goose, Kiro and OpenCode — and Laravel Boost, a useful
precedent for a Laravel project publishing skills.

**Target clients for verification: Claude Code, Codex and Gemini CLI.** Reading the format and
running a skill correctly are different claims — script execution and working-directory handling
vary — so portability is something we test on three, not something we assert for forty-five.

#### Distribution: `gh skill`, not something we build

There is no open standard for *plugins*: the specification covers one skill folder, and bundling
several with hooks and commands is a single vendor's concept. Rather than invent packaging, use
the tooling the ecosystem has converged on.

**`gh skill`** (GitHub CLI v2.90.0+) discovers, installs, manages and publishes skills straight
from a GitHub repository — `search`, `install`, `list`, `preview`, `update --all`, and
`publish --dry-run` to validate before release. It covers all three target clients, routes each
install to the right per-client directory, and carries supply-chain provenance.

That settles the distribution question entirely:

| Concern | Answer |
| --- | --- |
| How does a researcher install these? | `gh skill install` |
| How do we publish? | Push to the repository; `gh skill publish` |
| Per-client install directories | Handled by the tooling, not by us |
| Do we need a plugin manifest? | **No.** `gh skill` serves Claude Code as well as the others |
| Do we need per-vendor builds? | **No.** The same `SKILL.md` runs everywhere; only the install path differs |

`gh skill` is in public preview, so confirm it is still the right bet before Phase D. The fallback
is unexciting rather than alarming — a repository of standard skill folders can be copied into any
client's skills directory by hand.

**Validation in CI.** Adopt an existing linter rather than writing one; `skill-lint`, `skillmd`
and `agent-skills-lint` all validate `SKILL.md` against the spec, and the last checks per-client
schemas. Treat the choice as replaceable and do not build CI around one tool's output format.

#### Repository layout

Skills are self-contained, which is what the standard requires and what makes a folder work when
copied into any client:

```
tlcmap-skills/
├─ skills/                  # the deliverable — standard-compliant, self-contained
│  ├─ tlcmap-search/
│  │  ├─ SKILL.md           # name + description frontmatter; portable core only
│  │  ├─ scripts/           # entry points, PEP 723 inline dependencies
│  │  └─ references/        # cheatsheets this skill loads on demand
│  ├─ tlcmap-resolve/
│  ├─ tlcmap-geoparse/
│  ├─ tlcmap-analyse/
│  ├─ tlcmap-visualise/
│  ├─ tlcmap-prepare/
│  └─ tlcmap-cite/
│
├─ tests/                   # unit tests + the production compatibility check
├─ evals/                   # gold sets, fixtures, ground-truth artefacts (§11)
└─ examples/                # the worked demonstrations
```

**Keep `SKILL.md` to the portable core.** `name` and `description` are the fields every client
reads. Client-specific frontmatter — tool allowlists, model hints — is what makes a skill stop
being portable, so it stays out.

**Shared code is a deferred decision, measured rather than assumed.** A portable skill cannot
reach a library at the repository root, because that root does not exist once the folder is
copied elsewhere; sharing would mean vendoring one source copy into each skill at build time, and
no existing tool does that. But the MCP server absorbs the HTTP client, query building,
normalisation and catalogue caching, and most of what remains is skill-specific — chunking and
anchoring belong only to `tlcmap-geoparse`, exports only to `tlcmap-visualise`, validation only to
`tlcmap-prepare`. The plausibly shared surface is provenance manifests and coordinate
verification.

So: **start with no build step.** After A1 we will know what two real skills actually share. If it
is a couple of hundred lines, add `src/` and a short vendoring script; if it is negligible,
self-contained skills stay simpler and more standard-compliant. Deciding now would be guessing.

**Dependencies without an install step.** Each script declares its own dependencies with PEP 723
inline metadata, so `uv run scripts/analyse.py` resolves them per-script into an ephemeral
environment. The heavier skills want `pandas`, `shapely` and `matplotlib`; nobody should install
that stack to run a search. It is a plain shell invocation, so it travels as well as the skill
format does.

**MCP carries the reach; skills are portability on top.** MCP is the mature standard in this
stack, and a client with no skills support still gets full tool access through the server. That is
the other reason Layer 2 comes before Layer 3 (§4.2): its portability is not in question.

**Versioning against an unversioned API.** The API is not versioned (§2.4) and the upper layers
depend on documented behaviours, so `tests/` includes a compatibility check run against production
that fails loudly when one of them changes. When the `udateend` bug is fixed, we find out from a
red test rather than from a wrong timeline.

---

## 5. Layer 1 — the API

Ordered by what depends on each item. §11's compatibility suite asserts every one of these
against production once it lands.

### 5.1 Tier 1 — required first

**① Layer extent and facets on the catalogue.**

```
GET /layers/json?bbox=&datefrom=&dateto=&q=&recordtype=&page=&per_page=
```

with computed `bbox`, `date_range` and `record_count` on every entry — one PostGIS query
(`ST_Extent` and min/max over dates, grouped by `dataset_id`), refreshed on layer change.
**Computed, not declared**: the contributor-filled fields are populated on 4 of 2,118 layers and
two of those four are malformed (§2.2), so they stay a separate hint.

> **Required by:** `tlcmap_list_layers`, and therefore all regional discovery. Fixes the same gap
> in the browser interface. **The highest-value change in the programme.** Needed by Phase A.

**② Honest errors instead of HTML redirects.**

| Situation | Now | Should be |
| --- | --- | --- |
| Over the 5,000 ceiling | `302` → `/maxpaging` HTML | `413` + JSON `{error, total, suggestions}` |
| Layer private | `200`, no `features`, key `warnnig` | `403` + JSON |
| Layer missing | `200`, no `features` | `404` + JSON |
| Unparseable `extended_data` | `200`, filter silently dropped | `400` + JSON naming the expression |
| Missing analysis parameter | `500` | `400` + JSON naming the parameter |
| Bad `format` on `/api` | `302` → home page | `400` + JSON |
| `id=` on `/places` | `302` → the path form | Serve it directly |

> **Required by:** every tool, because it is the difference between a server that reads a status
> code and one that guesses from content types — guesswork that would otherwise be baked into the
> layer whose mistakes are quietest. Needed by Phase A.

**③ Extend `/api` to contributed layers.** `/api` pages properly through any number of matches
and reports `total`, but serves the two gazetteers only. Contributed layers — the research data —
are reachable only through `/places`, the endpoint with the ceiling.

> **Required by:** any question spanning contributed layers at scale, which is most of them. It is
> what makes arbitrary regions tractable without a client ever subdividing a bounding box.
> Probably the best value-to-effort ratio on the list, since the paging machinery already exists.
> Needed by Phase B.

**④ A cheap count.** `count_only=true` returning `{total}` without building features, or `total`
in `/places` output the way `/api` already has it. Today a client cannot ask how big a result set
is without running the query, and running it is exactly what fails when it is large.

> **Required by:** any workflow that must decide whether a query is tractable before spending it.
> Needed by Phase A.

### 5.2 Tier 2 — data correctness

**⑤ Namespace extended data in GeoJSON.** Return it under `properties.extended_data{}` rather
than merged into `properties`, where a contributor's column named `description` or `source`
silently overwrites the built-in field and nothing says which is which. Changes an existing
response shape, so it needs a transition. *Phase B.*

**⑥ Fix `udateend` in search output.** It is computed from the start date, so every record
reports `udateend == udatestart` and every timeline built from a search shows instants. *Phase B.*

**⑦ Stop `sort` and date filters silently discarding records.** Add `nulls=last` to `sort`, and
`include_undated=true` to date filtering. A brief for "the Hunter Valley, 1820–1860" otherwise
loses every undated record, and most gazetteer records are undated. *Phase A.*

**⑧ Make `limit` deterministic; add `sample`.** `limit=N` should take the first N; the existing
shuffle-then-take behaviour moves to `sample=N`. Both uses are legitimate; one parameter should
not silently mean the surprising one. *Non-blocking.*

**⑨ Return the match score on `fuzzyname`.** The trigram similarity is computed to rank results
and then discarded. Exposing it, with a threshold, gives the resolver a server-computed signal it
cannot obtain at any price — **the one item on this list that cannot be worked around
client-side at all.** *Phase B, ahead of Phase D.*

### 5.3 Tier 3

**⑩ Bulk fetch by ID.** `ids=a1353c,n77b93,…`. Today `id=` takes one and comma-separated values
redirect to a nonsense path, so a resolver makes one request per candidate. *Phase B.*

**⑪ Controlled vocabulary endpoints.** `state`, `lga`, `feature_term` and `parish` are **exact**
matches against the gazetteer's vocabulary — `lga=CESSNOCK` works, `lga=Cessnock Council` does
not — and nothing exposes the valid values. *Phase A (region-scoped), extended in Phase B.*

**⑫ Statistics as JSON.** `basicstatistics/json` returns geometry only while the numbers — count,
area, density, date range, median, mean — are computed for the HTML page and thrown away.
Advanced statistics has no JSON endpoint at all. *Phase A.*

**⑬ DBScan distance in real units.** Accept metres against geography rather than
degrees-divided-by-100, whose meaning changes with latitude. *Non-blocking.*

**⑭ Conditional requests and a change feed.** `ETag` / `Last-Modified` on feeds, and
`updated_since` on layers and the catalogue. Without it a scheduled pipeline refetches and diffs
everything on every run. *Phase E.*

**⑮ A structured licence identifier** alongside the free-text field. The free text stays and is
still what gets displayed; the identifier lets a client *assist* a republication decision without
ever making it (§3.3). *Non-blocking.*

**⑯ Import fixes.** Stop stripping digits from field headings (`Area m2` → `Area m`). Report every
date error rather than aborting on the first. Expose the validator as
`POST /layers/{id}/validate` for dry runs. *Phase D.*

**⑰ Small defects.** `warnnig` → `warning`; the RO-Crate metadata declaring `TLCMLayer_{id}.json`
while shipping `tlcmap_output.json`; `chunks` returning `500`. *Non-blocking.*

**⑱ Published rate limits.** More pressing once a public MCP endpoint exists. *Phase C.*

### 5.4 The write API

Needed for use cases 1, 3 and 7, and the precondition for authenticated MCP tools.

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

What matters more than the routes:

- **Scoped personal access tokens** (`layer:read`, `layer:write`), revocable, not the session
  cookie.
- **Idempotency** via an `Idempotency-Key` header and upsert on a client-supplied key. Without
  these, a scheduled pipeline that fails halfway and retries duplicates the whole layer. The
  single most important requirement here.
- **A validation dry run** returning the same errors as a real import.
- **All errors at once**, not the first.
- **Provenance preservation**, so extended-data fields written by a pipeline survive round-trips
  unmodified.

### 5.5 What stays in the client

Not everything belongs in the API. These are genuinely the agent's work, and no endpoint should
absorb them:

| Stays client-side | Why |
| --- | --- |
| Turning a research question into a query | Judgement, not a parameter |
| Choosing among exact / contains / fuzzy matching | Depends on what the researcher knows about their own data |
| LLM disambiguation of candidates | The candidates come from the API; the choice does not |
| Confidence banding and review routing | A research-workflow decision |
| Coordinate provenance validation (§3.1) | Guards against the *model*, not the API |
| Attribution presentation and ethics gating | Requires human judgement by design (§3.3, §12) |
| Reproducible notebook emission | An output format, not a data source |
| Longitude-first `bbox` | Correct GeoJSON convention, not a defect |
| The date formats themselves | A genuine feature — mixed `1856` / `1856-03` / `-400` is right for historical data |

---

## 6. Layer 2 — the MCP server

### 6.1 Architecture

**PHP and Laravel, inside the TLCMap application** — same stack, same deployment, same database,
and at the write phase the same authentication and permission model.

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

Served over HTTP rather than stdio, because the point is reach: a remote endpoint needs no local
install, which keeps the skills' setup to one line of configuration.

The server shares no code with the skills' Python, and does not need to. Most of what a
client-side toolkit would do is speak HTTP to TLCMap and cope with what comes back; a server
inside the application queries the database instead. What both are clients of is §5 — one
contract, two transports.

### 6.2 Design rules

1. **No model calls, ever.** The server exposes data and leaves judgement to the client that
   holds the model. This keeps prompts, adjudication and confidence banding in one place rather
   than reimplemented in PHP — and it is why `tlcmap_resolve_candidates` returns candidates
   rather than a choice.
2. **Handles and summaries, never bulk.** A tool result returns
   `{count, extent, date_range, resource_uri}`; features live behind a resource the client
   fetches only if it needs them (§3.5).
3. **The tools expose the API contract, nothing more.** A server with direct database access
   *could* answer questions the HTTP API cannot, and must not: one contract, two transports, so a
   question has the same answer whichever way it is asked. Anything worth adding is added to §5
   and reaches both.
4. **Every result carries provenance** — the equivalent query, the fetch time, the record UIDs.
   An MCP-driven answer must be as retraceable as a skill-driven one.
5. **Errors are typed and actionable.** `RESULT_TOO_LARGE` carries the total and a suggestion;
   `FILTER_REJECTED` names the expression it could not parse. This is §5.1 ② surfacing at the tool
   layer.
6. **Read and write are separate scopes from day one**, so the public read-only deployment and an
   authenticated one are the same server configured differently.

### 6.3 The tool surface

Twelve read tools. Six are specified in full — the ones the vertical slice builds (§9) — and six
are sketched.

#### `tlcmap_list_layers`

Faceted catalogue search. The tool regional discovery is impossible without.

| | |
| --- | --- |
| **In** | `query`, `bbox`, `polygon`, `date_from`, `date_to`, `include_undated`, `record_type`, `creator`, `has_license`, `has_warning`, `min_records`, `limit`, `cursor` |
| **Out** | `{total, returned, layers[{layer_id, name, description, creator, record_count, bbox, date_range, has_warning, license_present}], cursor, provenance}` |
| **Errors** | `INVALID_GEOMETRY`, `RESULT_TOO_LARGE` |

`bbox` matches the **computed** extent (§5.1 ①), never the contributor-declared fields. Returns
layer summaries only; records come from `tlcmap_get_layer`. `has_license` and `has_warning` report
presence, **not** an interpretation of content — they let an agent find a region's no-licence
layers without the server ever deciding what a licence permits.

#### `tlcmap_search_places`

Search places across all four sources.

| | |
| --- | --- |
| **In** | `name_query`, `match` (`exact`\|`contains`\|`fuzzy`), `sources[]`, `bbox`, `polygon`, `date_from`, `date_to`, `include_undated`, `state`, `lga`, `feature_term`, `record_type`, `layer_ids[]`, `extended_data[]`, `count_only`, `limit`, `cursor` |
| **Out** | `{total, returned, extent, date_range, sources_breakdown, layers_present[], places[], resource_uri, provenance}` |
| **Errors** | `RESULT_TOO_LARGE` (with `total` + narrowing suggestions), `FILTER_REJECTED`, `INVALID_GEOMETRY` |

`match` replaces the `name`/`containsname`/`fuzzyname` trap — one parameter with three honest
values rather than three parameters with precedence rules. `include_undated` is explicit rather
than a silent exclusion. `count_only` (§5.1 ④) answers "how big is this?" without building
features. There is **no `limit`-as-random-sample**. `layers_present` gives the distinct layers
contributing to a result, so an agent can go straight to their rights without paging every record.

#### `tlcmap_get_layer`

A layer's records and metadata.

| | |
| --- | --- |
| **In** | `layer_id`, `sort`, `nulls` (`first`\|`last`), `line` (`none`\|`route`\|`time`), `bbox` |
| **Out** | `{metadata, record_count, extent, date_range, undated_count, resource_uri, provenance}` |
| **Errors** | `LAYER_PRIVATE` (403), `LAYER_NOT_FOUND` (404) |

Summary plus a resource handle; never inline features. `nulls=last` (§5.2 ⑦) stops sorting from
silently deleting undated records. `undated_count` is explicit so the exclusion can be reported
rather than discovered.

#### `tlcmap_get_layer_metadata`

Rights, licence, citation and warning, without building any features.

| | |
| --- | --- |
| **In** | `layer_id` |
| **Out** | `{name, creator, publisher, contact, citation, license, rights, doi, warning, temporal_extent, spatial_extent, record_count, provenance}` |
| **Errors** | `LAYER_PRIVATE` (403), `LAYER_NOT_FOUND` (404) |

Cheap by design, so an agent can check rights *before* fetching data — which is what makes §3.3's
ordering enforceable rather than aspirational.

`license`, `rights` and `warning` are returned **verbatim as strings**. The server does not
normalise them, does not infer permission, does not omit an unparseable one, and does not
helpfully convert a warning field containing the literal string `"None"` into an absent warning.
If §5.3 ⑮ adds a structured identifier it appears as an *additional* field, never a replacement.

#### `tlcmap_list_vocabulary`

Valid values for the exact-match filters.

| | |
| --- | --- |
| **In** | `field` (`feature_term`\|`state`\|`lga`\|`parish`\|`record_type`), `bbox` (optional) |
| **Out** | `{field, values[{value, record_count}], provenance}` |

Requires §5.3 ⑪. Small, unglamorous, and the difference between a filter that works and one that
silently matches nothing. The optional `bbox` is what lets a brief group a region's records by the
feature terms that actually occur in it.

#### `tlcmap_layer_statistics`

The numbers, as numbers.

| | |
| --- | --- |
| **In** | `layer_id`, or `bbox` + `layer_ids[]` for a region |
| **Out** | `{count, extent, centroid, convex_hull, area_km2, density_per_km2, date_range, median_date, mean_date, undated_count, feature_term_breakdown, provenance}` |
| **Errors** | `LAYER_PRIVATE`, `INSUFFICIENT_RECORDS` |

Requires §5.3 ⑫.

---

The remaining six:

**`tlcmap_resolve_candidates`** — given a placename and what is known around it, return ranked
candidates **with evidence**: `match_score` (the trigram similarity the database already computes,
§5.2 ⑨), `matched_on` naming which field matched, and `distance_from_prior_km` turning a spatial
prior into a number the model can weigh rather than a filter that silently excludes. It returns
candidates and never a decision. The most important tool outside the slice.

**`tlcmap_get_places`** — bulk fetch by TLCMap ID, up to 200 per call. Requires §5.3 ⑩.

**`tlcmap_get_text_layer`** — a geoparsed text layer with its mention offsets, the text itself as
a resource rather than a field.

**`tlcmap_cluster_layer`** — `method` (`dbscan`\|`kmeans`\|`temporal`). Distance in metres against
geography, per §5.3 ⑬.

**`tlcmap_compare_layers`** — closeness analysis between two public layers. Warns on cost before
running, since it is a cross join.

**`tlcmap_build_view_url`** — a correctly percent-encoded TLCMap Views URL, refusing combinations
the data cannot support: a timeline needs `udatestart`/`udateend`, a journey needs `LineString`
features.

### 6.4 Resources

Bulk payloads are MCP resources with stable URIs, not tool results:

| Resource | Holds |
| --- | --- |
| `tlcmap://search/{query_hash}` | The full feature collection for a search |
| `tlcmap://layer/{id}/features` | A layer's records |
| `tlcmap://layer/{id}/text` | An uploaded text's full content |
| `tlcmap://layer/{id}/crate` | The RO-Crate snapshot |

Content-addressed where practical, so a client can cache and a provenance record can cite a
specific state of the data.

### 6.5 Tool surface stability

The surface is cheap to change structurally and expensive to change semantically (§4.1), so the
rules govern meaning rather than shape.

**Rename rather than redefine.** The central rule. A renamed or removed tool fails loudly, the
agent re-reads the tool list and adapts, and nothing silently wrong reaches a researcher. A tool
that keeps its name while its parameters change meaning succeeds quietly and corrupts the output.
So when the semantics of `bbox`, `date_from`, `include_undated` or a returned field genuinely
have to change, **ship a new name and deprecate the old one** — never redefine in place, however
tempting the continuity looks.

**Additive changes are free; semantic changes are not.** New tools, new optional parameters and
new output fields cost nothing, because an agent that has not heard of them does not use them.

**Version the surface, not the API.** The API stays unversioned (§2.4); the MCP server declares a
version and can keep a deprecated tool alive through a transition.

**Remember what does not re-read.** The self-describing argument covers agents in a session, not
the artefacts built around the tools: SKILL.md files naming tools, a site chatbot's fixed prompt,
eval fixtures, provenance records citing a tool whose behaviour has since moved. Those need the
deprecation window that agents do not.

**The slice is the design review.** The six sketched tools are not published until one real
workflow has exercised the six specified ones.

### 6.6 What the server must not do

- No model calls.
- No judgement: no "best match", no confidence, no ranking that encodes a decision rather than a
  measurement.
- No capability beyond the HTTP API (rule 3).
- **No summarising or normalising of `license`, `rights` or `warning`.** Verbatim or not at all.
- No write, until scopes and tokens exist.

---

## 7. Layer 3 — the skills

### 7.1 What a skill is here

Instructions plus **tool calls**, with Python only where the work is genuinely local. A skill's
job is deciding what to ask, judging what comes back, and producing the output.

What stays local, in each skill's own `scripts/` (§4.3):

| Local | Why |
| --- | --- |
| Text chunking and offset anchoring | Operates on the researcher's document, not on TLCMap |
| Export to QGIS, GPX, KML, Leaflet, notebooks | File production on the researcher's machine |
| Provenance manifests | Records what *this* run did |
| Coordinate verification against fetched records | Guards the model (§3.1) |
| Deduplication and composition across results | Needs the whole result set in hand |

There is no HTTP client, query builder, response normaliser or catalogue cache. Those are the
server's job.

### 7.2 The skill boundary test

A host with the MCP server configured and **no skills installed** can already search TLCMap, read
a layer and check a licence. So a skill cannot justify itself by knowing how to call an endpoint.

> **If a competent model holding the tool list would do this correctly unaided, it does not need
> a skill.**

What earns a skill: multi-step pipelines, judgement boundaries, ethics gates, local artefact
production, provenance. What does not: parameter selection, endpoint choice, and workarounds for
behaviour §5 has fixed.

### 7.3 The skills

**`tlcmap-search`** — find and retrieve places and layers. Turns a research question into a query,
runs it, and reports what it found *and what it excluded*. Chooses sources (gazetteers for
authoritative placenames, contributed layers for research data, text-derived places for
geoparsed corpora), handles region and date framing, and never confuses a result set with a
sample.

*Provisional.* This is the skill most exposed to §7.2: once the tool layer carries the parameter
choices, what remains is the research implications — undated records silently excluded, gazetteer
versus contributed — and the cached artefact with provenance. It may be better as reference
material the other skills load. Phase A settles it.

**`tlcmap-resolve`** — placename strings to TLCMap records. Normalises input, applies priors
(state, region, date, feature type), calls `tlcmap_resolve_candidates`, and has the model
adjudicate with the row's full context and the candidates — *and nothing else* — returning
`{uid | "unknown", confidence, reasoning, evidence_used}`. Bands by confidence, routes the
uncertain to a reviewer page, and copies coordinates from the TLCMap record. "Unknown" is a
correct answer, not a failure.

**`tlcmap-geoparse`** — map the places mentioned in a document. Chunks with stable offsets, has
the model return **verbatim surface forms and their sentences** — never character offsets — then
anchors each by exact string search in the source. Anchoring yields precise offsets *and* acts as
a hard hallucination guard: a surface form not present verbatim is dropped and counted. Extraction
is model-based rather than using an NER library, because statistical taggers are trained on modern
news text and degrade exactly where this corpus lives — colonial orthography, OCR noise, and the
context that distinguishes a river from a surname.

**`tlcmap-analyse`** — spatial and temporal distribution. Chooses which analysis answers the
question, calls `tlcmap_layer_statistics` and `tlcmap_cluster_layer`, computes locally what the
API does not expose, and drafts prose strictly from the computed numbers with the exclusion counts
stated. Emits the notebook that reproduces the figures.

**`tlcmap-visualise`** — maps, timelines, charts and embeds. Two routes: TLCMap Views URLs via
`tlcmap_build_view_url`, matching view to data; and local artefacts — Leaflet or kepler.gl HTML,
matplotlib figures, a QGIS project, KML and GPX for field devices, a static storymap scaffold.

**`tlcmap-prepare`** — build and validate a TLCMap-ready layer. Validates dates against all eight
accepted forms (one bad date aborts an entire import, so every row is checked and every failure
listed at once), previews heading sanitisation, flags collisions with built-in fields, checks
coordinates for plausibility, and runs a `ghap_id` dry-run diff against the current layer. Gains
`push` when §5.4 lands.

**`tlcmap-cite`** — attribution, licensing and citable packaging. Collects
`tlcmap_get_layer_metadata` for every layer touched and assembles an attribution block with
`warning` reproduced verbatim and prominently. States what the licence says; never decides what it
permits. Where metadata, keywords or warning indicate Indigenous knowledge, cultural material or
sensitive sites, it stops and surfaces the question. Invoked automatically by every export path.

### 7.4 Where guidance lives

Guidance duplicated between tool descriptions and skill instructions will drift, and the two will
eventually disagree in front of a researcher.

> Tool descriptions carry what you need to **call it correctly**.
> Skills carry what you need to **decide whether to call it**.

*"Never use `limit` to truncate a result set"* belongs in a description. *"A dated search excludes
undated gazetteer records, which usually matters more than the researcher expects"* belongs in a
skill. Neither belongs in both.

### 7.5 What is deliberately not built

Recorded so the omissions read as decisions rather than oversights. None of these exists in this
design, because §5 fixes each at the source (§3.8):

- A client-side layer extent index, or any harvest of all 2,118 layer feeds as a product feature.
- Recursive bbox subdivision to stay under the result ceiling.
- Content-type sniffing to detect HTML error pages.
- Running a query twice to detect whether a filter was applied.
- Re-deriving `udateend` from `dateend`.
- Hardcoded vocabulary lists for `state`, `lga` or `feature_term`.

---

## 8. Use cases

Full descriptions — example prompts, the researcher, the pipeline step by step, what the skills do
*not* do, and a worked example from real TLCMap data for each — are in
**[USE-CASES.md](./USE-CASES.md)**. This table is the mapping.

| # | Use case | Skills | Prerequisites | Phase |
| --- | --- | --- | --- | --- |
| [1](./USE-CASES.md#1-historical-text--mapped-corpus) | Historical text → mapped corpus | geoparse → resolve → visualise → prepare | ②③⑨⑩⑪ | D |
| [2](./USE-CASES.md#2-place-based-corpus-enrichment) | Place-based corpus enrichment | resolve → cite | ⑨⑩⑪ | D |
| [3](./USE-CASES.md#3-regional-knowledge-synthesis-for-fieldwork) | Regional fieldwork brief | search → analyse → visualise → cite | ①②④⑦⑫ | **A** |
| [4](./USE-CASES.md#4-indigenous--colonial-name-co-mapping) | Indigenous / colonial co-mapping | search → cite → visualise | ⑮ (to *assist* only) | any — **human-gated** |
| [5](./USE-CASES.md#5-toponym-pattern-analysis) | Toponym pattern analysis | search (harvest) → analyse → visualise | ③ | B |
| [6](./USE-CASES.md#6-environmental--event-history-overlay) | Environmental / event history overlay | geoparse → resolve → analyse → visualise | ⑥⑦ | B or D |
| [7](./USE-CASES.md#7-comparative--longitudinal-mapping-of-a-single-concept) | Longitudinal managed layer | prepare → *(write API)* → visualise | ⑭ + §5.4 | E |
| [8](./USE-CASES.md#8-teaching--public-facing-storymaps) | Teaching storymaps | search → visualise | ⑥ (improves) | B |

Circled numerals are the API items in §5.

### Additional capabilities

Not in the original brief, but cheap, high-value and directly useful to TLCMap's own community —
described in full under [Additional capabilities](./USE-CASES.md#additional-capabilities).

- **Layer health report** *(in `tlcmap-prepare`)* — unparseable dates, implausible coordinates,
  duplicates, headings mangled on import, missing licence or citation. The import is strict and
  its failures are opaque; this turns them into a list a contributor can act on.
- **Cross-layer duplicate detection** *(in `tlcmap-analyse`)* — so a merged multilayer does not
  triple-count the same township.
- **Resolution evaluation harness** *(in `evals/`)* — precision, recall and abstention rate.
- **Reproducible notebook emission** *(in `tlcmap-analyse`)*.
- **Saved-search awareness** — a saved search is a stored query, not a stored result. Prefer a
  re-runnable query over a frozen extract where the question is ongoing.

---

## 9. The vertical slice

One workflow, cut through all three layers, before any layer is built out. It exists to produce
evidence about the tool surface and the largest API change while both are still cheap to alter.

### 9.1 Target: use case 3, scoped to one region

[Regional knowledge synthesis for fieldwork](./USE-CASES.md#3-regional-knowledge-synthesis-for-fieldwork).
A bounding box in; a consolidated brief, field files and a complete attribution block out.

Three things make it the right slice:

1. **It does something researchers cannot do at all today.** Not "does it faster" — cannot do.
   Regional discovery is impossible from the catalogue (§2.2).
2. **It exercises the largest API change end to end.** §5.1 ① is the item everything else depends
   on for discovery and the biggest single piece of platform work. Proving it in a real workflow
   first is a better use of a slice than proving four small fixes.
3. **It puts the rights machinery under real load** (§9.3), which is the part of this design that
   is hardest to retrofit and most costly to get wrong.

The discipline that keeps it a slice rather than a phase is narrow scope within the use case. Note
that the left column is what the slice *verifies*, not what the skill *supports* — the workflow
takes a region as a parameter and nothing in it is Hunter-Valley-specific:

| Verified in the slice | Explicitly out |
| --- | --- |
| **One region**, end to end | Dense regions that exceed the ceiling, and multi-region composition (§9.5) |
| Discovery → retrieval → rights → dedupe → brief → field files | Resolution, geoparsing, upload, write |
| 6 MCP tools | The remaining six |
| API ① ② ④ ⑦ ⑫ — five items | The other thirteen |
| Markdown brief, GPX, KML | QGIS project styling, PDF typesetting, storymaps |
| Rights surfaced and gated | Automated rights *decisions* — never in scope at all |

### 9.2 The region: the Hunter Valley

```
bbox 150.8,-33.1,151.4,-32.6
  → 273 records across 36 distinct contributed layers
```

Chosen because it is real, dense enough to be interesting without exceeding the 5,000-record
ceiling, and carries the colonial and convict history that makes a brief worth reading. Wollombi,
Cessnock and the Singleton parishes sit inside it, so the gazetteer half is substantial too.

The 36 layers are a genuine long tail: weather stations (461) contributes 69 records, polling
places (717) nineteen, and a dozen layers contribute one record each. That tail is what makes
deduplication and grouping non-trivial rather than decorative.

### 9.3 What is actually there — measured

The 36 layers break down like this:

| | |
| --- | --- |
| Carry a **warning** | 8 |
| Carry a **licence** | 12 — in six different spellings |
| Carry a **citation** | 8 |
| **No licence at all** | **24** |

That distribution is the slice's real test, and four layers make it concrete.

**Layers 2749 and 2849 — "Public schools attended by Aboriginal and/or Torres Strait Islander
students"**, licensed `CCBY-NC-ND`, each carrying:

> *"Aboriginal and/or Torres Strait Islander Peoples are advised that this map may contain links
> to images and words…"*

An ordinary regional query surfaces these without anyone asking for sensitive material. The
advisory has to travel into the brief, the GPX file and any figure — not be summarised, and not be
dropped because an output format is inconvenient.

**Layer 1125 — "50 words project"**, licensed:

> `"Closed (subject to the access condition details)"`

A client matching known identifiers finds none and treats it as unrestricted; one that
pattern-matches "Closed" and guesses is worse for being confident. Correct behaviour is to surface
it verbatim and stop.

**Layer 206 — "Music communities"**, whose `warning` field contains the literal string:

> `"None"`

A present warning that says nothing, which is not the same as an absent one. Small, and exactly
the kind of contributor-data reality that separates a design that works from one that demos.

**And 24 layers carry no licence**, which is the majority. The brief must say that absence of a
licence is not permission, for each of them, without editorialising further.

### 9.4 Success criteria

Three of the five are numbers rather than judgements.

1. **Layer discovery recall and precision.** Ground truth computed once, offline, by harvesting all
   2,118 public layers and calculating true extents — an *evaluation artefact, not a product
   feature* (§9.5). `tlcmap_list_layers` with a bbox must find every layer with a record in the box
   and no others. **Target: recall 1.0.** Anything less means §5.1 ① is wrong, which is precisely
   what the slice exists to find out.
2. **Attribution completeness — must be 1.0.** All 36 layers appear in the attribution block with
   `creator`, `licence`, `citation` and `warning` reproduced verbatim. Automatable, binary, and a
   single omission fails the slice.
3. **Warning propagation — must be 1.0.** The two advisory warnings appear in every derived output,
   including the GPX and KML files, not only the markdown brief.
4. **The restricted layer stops the pipeline.** Layer 1125 routes to a human decision rather than
   into the brief, and the reason shown is the licence text itself.
5. **Exclusions are stated.** The brief says what it could not see: undated records dropped by any
   date bound, layers whose licence is absent, records without coordinates.

Plus one qualitative criterion, reviewed rather than scored: **a domain reader finds the brief
usable in the field.** Worth doing, and worth not pretending is a metric.

### 9.5 What the slice does not prove

- **It does not test resolution.** No adjudication, no candidate ranking, no abstention. The
  model's role here is synthesis and grouping, not judgement over evidence — so the central claim
  behind use cases 1, 2 and 6 stays unproven until Phase D. This is the main cost of the choice,
  and it should be planned for rather than discovered.
- **It does not test extraction or anchoring**, and so gives no evidence on the decision to drop
  NER libraries (§7.3).
- **It does not validate the tool surface for the resolver-driven use cases.** Six tools exercised,
  six inferred.
- **It does not prove scale**, which is a larger gap than it looks, because *the skill applies to
  any region from the moment it exists* — the constraint is on what the slice measures, not on what
  the workflow accepts. What a dense or very large region additionally needs:

  - **Paging, not chunking.** A region exceeding the 5,000-record ceiling is §5.1 ③'s problem, not
    the skill's. Recursive bbox subdivision in the client is exactly what §3.8 forbids and §7.5
    records as deliberately not built. ③ lands in Phase B and is what actually unlocks arbitrary
    regions.
  - **Composition that is not concatenation.** Records near a boundary appear in more than one
    request, so deduplication runs over the union. Attribution is unioned, not repeated — 36 layers
    across four sub-queries is 36 entries. And **the statistics do not compose**: density over a
    union is not the mean of densities, the convex hull of a union is not the union of hulls, and a
    point's nearest neighbour may lie outside its own sub-query. Since deduplication changes the
    counts too, statistics are recomputed over the combined set or they are wrong.
  - **A deliverable that changes shape.** 273 records make a brief someone reads in the field;
    50,000 across 400 layers make one nobody does. Past some size the right output becomes an index,
    a prioritisation or a filtered selection rather than a longer document.
- **It does not make the harvest a product.** The exhaustive extent harvest exists once, in
  `evals/`, to generate ground truth. It is not shipped, not run by users and not a fallback path —
  that would reintroduce exactly what §3.8 removes.

---

## 10. Plan

Two tracks run in parallel and are equally first-class: the **platform track** (§5) and the
**capability track** (server, skills). Per §3.8 the platform track is on the critical path — the
upper layers are written against the API as it will be, and each phase names what must be deployed
to tlcmap.org first.

### Phase A — the vertical slice

Run as two milestones, so the first is demonstrable before the second begins.

**A0 — spike, first days.** Confirm the PHP MCP tooling holds up (§12). It is the only open
question that could redirect the architecture, and it should be answered before the API work
commits to a tool-shaped contract.

**A1 — discovery and rights.**
*API:* ① ② ④ ⑦.
*MCP:* `tlcmap_list_layers`, `tlcmap_search_places`, `tlcmap_get_layer`,
`tlcmap_get_layer_metadata` — HTTP transport, provenance envelope, typed errors.
*Skills:* `tlcmap-search`, `tlcmap-cite`.
*Evidence:* ground truth from a one-off exhaustive extent harvest, held in `evals/` and never
shipped (§9.5).

**Deliverable:** a rights-complete inventory of everything TLCMap holds for the Hunter Valley — 36
layers, 273 contributed records plus the gazetteer — with attribution and warnings verbatim, the
restricted layer gated, and exclusions stated. Layer discovery scored for recall and precision.

**A2 — synthesis and field outputs.**
*API:* ⑫.
*MCP:* `tlcmap_list_vocabulary`, `tlcmap_layer_statistics`.
*Skills:* `tlcmap-analyse`, `tlcmap-visualise`.

**Deliverable:** the fieldwork brief — deduplicated across the 36 layers, grouped by feature term,
characterised with computed statistics — plus GPX and KML carrying the advisory warnings into the
field files.

**Decision gate**, read at the end of each milestone rather than once at the end:

- *A1 fails on discovery recall* — §5.1 ① is wrong, and that is the finding. It is the item the
  rest of the programme leans on hardest, so learning it here rather than at Phase C is the point
  of the slice.
- *A1 fails on attribution or warning propagation* — more serious than it sounds. Those criteria
  are binary and automatable, and a failure means the rights machinery does not survive contact
  with real contributor data. Fix before anything else proceeds.
- *A2 disappoints* — the synthesis needs work, but A1 has already shipped something obtainable no
  other way. Proceed to Phase B and revisit the brief.
- *All pass* — proceed with the largest API change and the rights machinery both validated against
  a real workflow.

### Phase B — API programme, and scale

③ (`/api` over contributed layers), ⑤ (namespaced extended data), ⑥ (`udateend`), then the
resolver's ⑨ ⑩ ⑪ ahead of Phase D, then the non-blocking remainder. All valuable independently of
everything above them.

③ is the one that changes what the Phase A skill can *do* rather than how cleanly it does it: it
removes the 5,000-record ceiling as a design concern and so **unlocks arbitrary regions** without
the client ever subdividing a bounding box (§9.5). Phase B therefore also carries the composition
work dense regions need — deduplication over the union, unioned attribution, statistics recomputed
rather than combined — and the question of what the deliverable becomes when a region returns tens
of thousands of records.

Use cases 5 and 8 become deliverable here.

### Phase C — full read-only MCP server

The six sketched tools, designed against what Phase A learned. Published at `mcp.tlcmap.org`, with
rate limits (⑱) in place before it is public.

**Deliverable:** TLCMap usable from Claude Desktop and any other MCP host, with no skills involved.
The first deliverable that reaches users outside Claude Code.

### Phase D — the resolver and the remaining skills

`tlcmap-resolve`, `tlcmap-geoparse`, `tlcmap-prepare`, plus the evaluation harness. Use cases 1, 2
and 6 become deliverable.

**Plan this as a second proving exercise rather than routine build-out.** Phase A deliberately
leaves resolution unproven (§9.5), so Phase D carries its own gold set, its own precision / recall
/ abstention figures, and its own gate.

### Phase E — write

§5.4, scoped tokens, write tools behind the scope separated in Phase A, and `push` in
`tlcmap-prepare`. Unblocks use case 7. Building the server inside the Laravel application pays off
here: scoped tokens extend the existing authorisation model rather than standing up a parallel one.

### Sequencing notes

- **Design and construction can proceed in parallel; *finishing* cannot.** A capability is not done
  until its API prerequisite is live on tlcmap.org.
- **§5.1 Tier 1 is the long pole for the whole programme.** Phase A waits on it.
- **Time to visible value is the metric to watch.** A1 exists so something is demonstrable early;
  if it slips past a few months, the scope is wrong, not the plan.

---

## 11. Measurement

A proof of concept that cannot be measured is a demonstration, not a proof.

| What | When | Measure |
| --- | --- | --- |
| **Layer discovery recall** | A1 | Against the offline extent harvest. Target 1.0 — it is what §5.1 ① is for |
| **Attribution completeness** | A1 | Every layer touched, verbatim. **Must be 1.0** |
| **Warning propagation** | A1 | Into every derived output including GPX/KML. **Must be 1.0** |
| **Rights gating** | A1 | Restricted layers stop the pipeline rather than entering the brief |
| **Provenance integrity** | A1 onward | Every coordinate traces to a fetched record. Automated, and tested as though it could fail |
| **API compatibility** | Continuous | The production suite catching silent platform changes (§4.3) |
| **Tool surface churn** | C onward | Semantic changes after publication. Target: zero (§6.5) |
| **Resolution accuracy** | D | Precision, recall and **abstention rate** — a resolver that correctly says "unknown" is worth more than one that guesses well |
| **Extraction accuracy, anchor drop rate** | D | Against a hand-annotated sample; the drop rate is the prompt-drift detector |
| **Skill triggering** | D | Fixtures from the example prompts in USE-CASES.md, including the *push-back* prompts, which test that a skill declines to fabricate a coordinate or proceed past a rights gate |
| **Courtesy** | Continuous | Requests per run, kept visible and low |

---

## 12. Risks and ethics

**Cultural sensitivity is the first-order risk, not a compliance footnote.** TLCMap holds
Indigenous placenames, massacre sites and mission records, and 165 public layers carry an explicit
contributor warning.

- Warnings travel with data, always, including into derived outputs, field files and figures.
- Licence and rights text is surfaced, never interpreted into a permission.
- Any pipeline touching Indigenous knowledge stops at a human decision. The system does not decide;
  it presents what the contributor said and asks.
- CARE principles (Collective benefit, Authority to control, Responsibility, Ethics) are stated in
  the skills' own instructions, not only in this document.
- Aggregation is itself a risk: combining layers can reveal sensitive site locations that no single
  layer disclosed. `tlcmap-analyse` flags composition across layers carrying warnings.

The slice tests this machinery rather than deferring it — the Hunter Valley query surfaces two
advisory-carrying layers, one `"Closed"` licence and 24 with none, without anyone asking for
sensitive material (§9.3). The residual gap is
[use case 4](./USE-CASES.md#4-indigenous--colonial-name-co-mapping) proper, where a researcher
*seeks out* Indigenous-name layers; the slice covers incidental exposure, not deliberate use.

**Fabrication.** Addressed architecturally in §3.1 and tested in §11, because instructing a model
not to invent coordinates is necessary and not sufficient.

**The slice leaves resolution unproven.** A deferral rather than a mitigation: adjudication quality
gets no evidence until Phase D. Nobody should read a successful Phase A as evidence that resolution
works, because it contains none.

**§5.1 ① is the long pole and Phase A depends on it entirely.** The honest cost of choosing the
slice that proves it. The compensation is that it is the item most of the rest depends on, so the
risk is taken early rather than avoided.

**PHP MCP tooling.** The SDK ecosystem is thinner in PHP than in TypeScript or Python. This sits on
the critical path, so it is spiked in the opening days of Phase A, before the API work commits to a
tool-shaped contract.

**Semantic drift in the tool surface.** Agents adapt to renamed tools; they cannot detect a
parameter whose meaning changed. §6.5's rename-rather-than-redefine rule is the mitigation.

**Coverage bias.** The gazetteers are uneven, contributed layers reflect who contributed. Analyses
must report what was excluded rather than presenting a partial distribution as a complete one.

**Load.** No rate limiting means we set our own. Cache aggressively, harvest once, never poll.

**Over-automation.** The temptation is to make the pipeline run end to end without stopping. The
review bucket and the human gates are the product, not friction in it.

---

## 13. Open questions

1. **Does the PHP MCP tooling hold up?** Spike in week one of Phase A. The only question that could
   redirect the architecture.
2. **Who runs and owns the ground-truth harvest?** One polite pass over 2,118 layer feeds computing
   true extents, held in `evals/`. It is the only thing standing between "discovery recall 1.0" and
   an unfalsifiable claim — and it must stay an evaluation artefact rather than drifting into the
   product (§9.5).
3. **Where does `mcp.tlcmap.org` run**, and does it share the application's infrastructure?
   Relevant because there is no rate limiting today and a public MCP endpoint makes that pressing.
4. **Is the tool surface right at twelve?** Fewer and richer, or more and finer? Phase A gives
   evidence for six of them; the instinct to resist is one tool per endpoint.
5. **Which second region tests generalisation?** The skill is region-parameterised from the start,
   so this is about what to *verify*, not what to support. A sparse or remote region, and one
   carrying more restricted layers, would each probe a different edge cheaply once A1 works. A
   region that *exceeds* the ceiling is Phase B's, because the honest answer there is ③ rather than
   client-side chunking.
6. **Do the skills require the MCP server, or degrade without it?** Requiring it is simpler and
   honest; degrading gracefully costs the HTTP client this architecture exists to avoid building.
   Recommend requiring it.
7. **How many skills, once the tools exist?** §7.3 flags `tlcmap-search` as provisional, because it
   largely fails the §7.2 test once the tool layer carries what it used to know. Current expectation
   is six, with search folded into reference material the others carry. Phase A settles it, and the
   shipped skill set stays unsettled until it does.
8. **Write API timeline** — sets Phase E, and it is the only thing standing between use case 7 and a
   working managed layer.
9. **Licensing and governance** of the three artefacts, which third parties will install. What
   licence, and what support expectation?
10. **Does shared skill code justify a build step?** Answerable only after A1, when two real
   skills exist and the duplication can be measured rather than guessed (§4.3). Until then, no
   build step.
