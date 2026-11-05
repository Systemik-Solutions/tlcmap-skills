# A1 — API v2: discovery and rights

**Status:** spec for build · **Date:** 2026-11-05 · **Repository:** TLCMap application
(`tlcmap-ghap`, branch `tlcmap-api-enhancement`) and `tlcmap-docs` (branch `api-v2`)

The API half of milestone A1 in [DESIGN.md §10](../DESIGN.md#phase-a--the-vertical-slice):

> *API:* ① ② ③ ④ ⑤ ⑥ ⑦ ⑧ — the first v2 endpoints: the layer catalogue, the layer feed and its
> metadata, and place search.

This document expands that line, and the items in
[DESIGN.md §5.2](../DESIGN.md#52-tier-1--required-by-a1), into something buildable. The MCP and
skills halves of A1 have their own specs. DESIGN.md remains the authority on principles; where this
spec and DESIGN.md disagree, one of them is wrong and is fixed.

---

## Contents

| | |
| --- | --- |
| [1. Purpose and scope](#1-purpose-and-scope) | What this delivers, what it leaves out |
| [2. Facts this spec rests on](#2-facts-this-spec-rests-on) | Read from the source and measured locally |
| [3. Architecture](#3-architecture) | Routing, code layout, the one shared edit |
| [4. Database objects](#4-database-objects) | Functions and indexes beneath both versions |
| [5. Request rules](#5-request-rules) | Strict parameters, echo, paging, ordering, `limit`, `sample` |
| [6. Dates](#6-dates) | `v2_udate`, periods, undated records |
| [7. Record filters](#7-record-filters) | The filters place search and the catalogue share |
| [8. The summary](#8-the-summary) | Whole-match-set figures, computed in SQL |
| [9. Features](#9-features) | The v2 GeoJSON feature |
| [10. Endpoints](#10-endpoints) | Place search, catalogue, layer detail, layer features |
| [11. Errors](#11-errors) | Status codes and the error body |
| [12. Rate limit and CORS](#12-rate-limit-and-cors) | v2's own throttle key |
| [13. Testing](#13-testing) | Test database, suites, the v1 guard |
| [14. Documentation](#14-documentation) | The v2 developer docs |
| [15. Acceptance](#15-acceptance) | When this half of A1 is done |
| [16. What the MCP server can rely on](#16-what-the-mcp-server-can-rely-on) | The contract handed to the MCP half |
| [17. Changes to DESIGN.md](#17-changes-to-designmd) | Made with this spec |

---

## 1. Purpose and scope

v2 is the contract the MCP server and the skills are written against
([DESIGN.md §5.1](../DESIGN.md#51-two-versions)). This milestone delivers its first four endpoints,
with every §5.2 Tier 1 behaviour pinned by a test, so that the MCP half of A1 has a fixed target.

**In scope**

- Four endpoints under `/api/v2/`: place search, the layer catalogue, layer detail, and a layer's
  features (§10).
- DESIGN §5.2 items ① to ⑧, each delivered and each covered by named tests.
- The database functions and indexes v2 needs (§4), added by migration beneath both versions.
- v2's own error rendering, throttle key and CORS headers.
- A test harness for the application: a dedicated test database built from migrations, factories,
  and a `V2` suite, plus characterisation tests guarding v1.
- The v2 developer documentation, on the `api-v2` branch of `tlcmap-docs`.

**Out of scope**

- Deployment. This half of A1 is done when it is merged on `tlcmap-api-enhancement`; releasing it to
  tlcmap.org is a separate step.
- Any change to v1: its routes, controllers, `FileFormatter`, `GeneralFunctions`, middleware,
  responses or defects. The only shared file edited is the exception handler, by one branch that
  non-v2 requests never enter (§3.3).
- Any change to stored data, including v1's `udatestart`/`udateend` columns, which stay as they are.
- Moving the MCP server to v2, the skills, and the ground-truth harvest (the other halves of A1).
- Items ⑨ onward, including real rate limits (⑱), match scores (⑨) and `ETag` (⑭).
- `line` on layer features, and the v1 parameters v2 does not carry forward (`names`, `subquery`,
  `searchdescription`, `anps_id`, `from`/`to`, `viewBbox`, `chunks`, `download`, Trove aliases).

---

## 2. Facts this spec rests on

Read from the application source on `tlcmap-api-enhancement` at `310deea`, and measured against the
local database (`temp_tlcmap`: PostgreSQL 14.5, PostGIS 3.2.2, pg_trgm 1.6; 187 layers, 687,235
records — a subset of production), 2026-11-05. Facts about production are DESIGN's, with their dates.

`temp_tlcmap` is the local database to develop against: all 36 migrations applied, both gazetteers
loaded, records changed as recently as 2026-11-02. The other local database, `tlcmap`, is a
December 2022 snapshot (62 layers, 31,331 records, no gazetteers) that predates 21 migrations — it has
no `datasource_id` or `geom` — and cannot run the current application's search.

**Platform.** Laravel 5.8.38, PHPUnit 7.5.20. Production runs PHP 7.4
([DESIGN.md §10](../DESIGN.md#sequencing-notes)); Laragon has PHP 7.4.19 and 8.2.9. The CLI on the
path is 8.2, so the tests are run with 7.4 explicitly.

**Tables.** Everything lives in the `tlcmap` schema, which no migration creates.

- `dataitem`: `id` bigint, `uid` (prefix + hex id: `a` ANPS, `n` NCG, `t` contributed), `dataset_id`
  (NULL for gazetteer records), `datasource_id` (default 1), `recordtype_id`, `title` (NOT NULL),
  `placename`, `description`, `latitude`/`longitude` double, `geom geometry(Point,4326)` and `geog`,
  `datestart`/`dateend` **text**, `udatestart`/`udateend` bigint, `state`, `lga`, `parish`,
  `feature_term`, `source`, `external_url`, `extended_data` **text holding KML XML**, `image_path`,
  `glycerine_url`, `dataset_order`, timestamps.
- `geom` and `geog` are maintained by the `update_geom_geog_trigger` BEFORE INSERT OR UPDATE trigger
  (migration `2024_01_25_083329`). Locally, every record with both coordinates has `geom`, and none
  without has it.
- `dataset`: `public` boolean, the metadata fields (`creator`, `publisher`, `contact`, `citation`,
  `doi`, `source_url`, `linkback`, `license`, `rights`, `language`, `warning`), the declared
  `latitude_from`/`_to`, `longitude_from`/`_to`, `temporal_from`/`_to`, `created`, `recordtype_id`.
- `datasource`: four rows, found by `search_param_name` — `searchpublicdatasets` (id 1, contributed
  layers), `searchausgaz` (2, ANPS), `searchncg` (3, NCG), `searchgeocoder` (4, places added from a
  text). No migration or seeder creates rows 1–3, or any `recordtype` row.
- `subject_keyword` and `dataset_subject_keyword` hold layer keywords.
- Indexes: btree on `dataitem.id`, `dataset_id`, `recordtype_id`, `uid`, `original_id`,
  `datasource_id`. **No GIST and no trigram index.**
- **The base schema probably did not come from the migrations.** The older indexes are named
  `idx_<number>_<name>` (`idx_18848_primary`, `idx_18848_fk_dataitem_recordtype_id`,
  `idx_18867_dataset_subject_keyword_subject_keyword_id_foreign`) — pgloader's pattern for indexes it
  copies from MySQL, around MySQL-style names (`PRIMARY`, Laravel's MySQL-only `_foreign` indexes). The
  15 migrations up to 2022-07 are recorded in `public.migrations` as one batch; every later one ran
  natively and created Laravel-named indexes. An inference, not a confirmed history — but enough that a
  database built from the migrations alone may differ, which §13.1 checks first.

**v1's stored dates are incomplete and partly wrong.** `udatestart`/`udateend` were filled once by
migration `2025_02_05_034149` and are set afterwards only by the web forms' add and edit paths
(`AjaxController.php:561-562, 664-665`). Bulk import and the ANPS/NCG console imports never set them;
locally, 20 records have a `datestart` and no `udatestart`. The parser,
`GeneralFunctions::dataToUnixtimestamp`, appends `-01-01` only to strings with no `-`, so `-400`
reaches `strtotime` as a relative expression and comes back as roughly the current time; it works in
`Australia/Sydney` local time. v1's date filter reads these columns, so v2 neither uses nor repairs
them.

**The data's dates.** [Formats](https://guide.tlcmap.org/developers/formats) (docs source,
`formats.md:255-262`) lists eight accepted forms, with `-YYYY` meaning BCE and "the year may be any
number of digits". Local data includes `-51021` and the impossible `-16112-11-31`. PostgreSQL
timestamps end at 4713 BC, so v2 computes epoch milliseconds arithmetically (§6).

**Separators.** 94 local `feature_term` values contain a comma, so vocabulary filters take one value.

**Extended data.** 344,350 local records carry `extended_data`, gazetteer records among them. v1
filters it with `(xpath(…, extended_data::xml))`, so one malformed stored value fails the whole
query; v2 reads it through a function that cannot (§4).

**Cost of a per-layer summary.** `GROUP BY dataset_id` with `ST_Extent`, min/max dates and counts over
the 56k public contributed records takes about 200 ms locally. Production has about eleven times the
layers; if records scale with layers, an unfiltered catalogue request takes a couple of seconds. That is accepted:
v2 computes at query time and stores no derived data (§4).

**Throttle keys are shared.** The `web` group applies `throttle:40,10` (commit `b34d303`) and the
`api` group `throttle:60,1`. Laravel 5.8's `ThrottleRequests::resolveRequestSignature` returns
`sha1(domain|ip)` regardless of group, so both share one counter per address. v2 uses its own key
(§12).

**Routes.** `routes/web.php` has `GET /api` (exact) and nothing matching `/api/v2/…`;
`routes/api.php` has only `GET /api/user`. `app/Exceptions/Handler::render` defers to the parent.

**Tests.** `tests/` holds two example tests; `phpunit.xml` sets no database, so tests use `.env`'s.
The only factory is `UserFactory`.

---

## 3. Architecture

### 3.1 Routing

v2 has its own route file and middleware group, registered in `RouteServiceProvider::map()` beside
the existing two:

```php
Route::prefix('api/v2')
    ->middleware('api_v2')
    ->namespace($this->namespace . '\Api\V2')
    ->group(base_path('routes/api_v2.php'));
```

```
GET /api/v2/places                    PlaceController@search
GET /api/v2/layers                    LayerController@index
GET /api/v2/layers/{id}               LayerController@show
GET /api/v2/layers/{id}/features      LayerController@features
```

The `api_v2` group in `app/Http/Kernel.php` holds the v2 throttle and CORS middleware (§12) and
`bindings`; no session, cookies or CSRF. `{id}` is constrained to digits at the route, so
`/api/v2/layers/abc` is a `404 NOT_FOUND`. Only `GET` is routed; any other method is
`405 METHOD_NOT_ALLOWED` (§11).

### 3.2 Code layout

All v2 code is new, under the `TLCMap\Api\V2` namespace and the v2 controllers. It reads the
application's tables and may use its Eloquent models for relations, but calls none of v1's
controllers, `FileFormatter`, `GeneralFunctions` or `Dataset::json()`, so no v1 change can alter a
v2 response and no v2 change can alter a v1 one.

```
app/Http/Controllers/Api/V2/
  PlaceController.php        parse → query → serialise; no SQL, no formatting
  LayerController.php
app/Api/V2/
  Params/                    request parsing and validation into value objects
    ParameterSet.php           the allowed names per endpoint; unknown and repeated → 400
    Bbox.php, Polygon.php, DateBound.php, Sources.php, LayerIds.php
    ExtendedDataFilter.php     the grammar of §7.8
    Sort.php, Paging.php, Cursor.php
  Query/
    RecordQuery.php            the record filters of §7, shared by search and catalogue
    PlaceSearch.php            RecordQuery + order + keyset page
    LayerCatalogue.php         RecordQuery → matching layers → per-layer aggregates
    Summary.php                the aggregate SQL of §8
  Serialise/
    FeatureSerialiser.php      §9
    LayerSerialiser.php        §10.2, §10.3
    QueryEcho.php              §5.2
  Errors/
    ApiException.php           code, status, message, details
    ErrorRenderer.php          any Throwable on a v2 path → §11 JSON
config/api_v2.php            rate limit, page sizes, fuzzy threshold, polygon vertex limit
routes/api_v2.php
```

`RecordQuery` is the single place a record filter is defined. Place search returns its records; the
catalogue groups them by layer. So **the catalogue lists exactly the layers that the equivalent place
search draws records from** — the property [DESIGN.md §9.4](../DESIGN.md#94-success-criteria)'s
recall and precision depend on, and a test of its own (§13.4).

### 3.3 The one shared edit

`app/Exceptions/Handler::render` gains one branch, first:

```php
if ($request->is('api/v2', 'api/v2/*')) {
    return ErrorRenderer::render($request, $exception);
}
```

Every other request takes the path it takes today. `ApiException` is added to `$dontReport`, since a
`400` is the client's error, not the server's; everything else is still reported and logged.

---

## 4. Database objects

Added by migrations, beneath both versions. None changes a table's columns, triggers or data, so no
v1 response changes (§13.5 guards this). v2 stores **no derived data**: dates, extents, counts and
summaries are computed at query time.

| Object | Definition |
| --- | --- |
| `tlcmap.v2_udate(text) → bigint` | The first instant of the period a date string names, as epoch milliseconds UTC; `NULL` if empty or unparseable (§6). `IMMUTABLE`, `STRICT`, `PARALLEL SAFE`. Pure arithmetic — no `timestamp` type, so no 4713 BC limit. |
| `tlcmap.v2_udate_end(text) → bigint` | The last millisecond of that period (§6.2). Same properties. |
| `tlcmap.v2_extended_value(text, text) → text` | The value of the named field in a record's `extended_data` XML, or `NULL` if the field is absent, the column is empty, or the XML is malformed. Catches the XML error rather than raising it. `IMMUTABLE`. |
| GIST index on `dataitem.geom` | For `bbox`, `polygon` and extents. |
| GIN `gin_trgm_ops` indexes on `dataitem.title` and `dataitem.placename` | For `match=fuzzy` and for `exact`/`contains`, which are written as `ILIKE` so the index serves them. |

The migration creates the indexes with `CREATE INDEX IF NOT EXISTS`, so a database where an operator
has already added one is not an error, and drops only what it created on rollback. Index builds on
production's table are a deployment consideration recorded for the release, not part of this plan.

Indexes are not derived data in the sense §1 rules out: PostgreSQL maintains them on every write
path, so they cannot go stale.

---

## 5. Request rules

Every endpoint follows these.

### 5.1 Strict parameters

- **Unknown parameters are rejected.** Each endpoint has a fixed parameter list (§10); any other
  query parameter is `400 INVALID_PARAMETER` naming it. A misspelt `date_form` silently ignored is
  [§2.3](../DESIGN.md#23-behaviours-that-break-a-naive-client)'s failure class again.
- **Repeated or array parameters are rejected** (`?sources=anps&sources=ncg`, `?layers[]=1`).
- **Values are validated**: type, range and allowed values, each failure a `400` naming the parameter,
  the rule and what it accepts.
- **Empty values.** `?bbox=` is a `400`, not an absent filter: an empty value is an error, never a
  silent no-op. (The global `ConvertEmptyStringsToNull` middleware turns it into `null`; v2 checks
  key presence, so a present key with a null value is the empty case.)
- **Booleans** are `true` or `false`, nothing else.
- **Names are snake_case**: `date_from`, `date_to`, `include_undated`, `record_type`, `feature_term`,
  `extended_data`, `summary_only`, `per_page`, `has_license`, `has_warning`, `min_records`.

### 5.2 The effective query is echoed

Every successful response carries `query`: every parameter of the endpoint with its effective value,
defaults applied, and `null` for filters not given. Lists are arrays; numbers are numbers. A reader
always sees that `match` was `exact` or `include_undated` was `true` without knowing the defaults.
`cursor` is echoed as given; `seed` as used.

### 5.3 Paging

Keyset cursors, never offsets, on every list endpoint.

- `per_page`: default 500 and maximum 5,000 on features; default 100 and maximum 1,000 on the
  catalogue. Above the maximum is `400 INVALID_PARAMETER`.
- Every list response carries
  `page: {returned, per_page, next_cursor, next}`. `next` is the full URL of the next page — this
  request with `cursor` set. Both are `null` on the last page.
- **The cursor is opaque**: base64url of a small JSON object holding a format version, a hash of the
  query, the last row's sort key and id, and the number of rows served so far (for `limit`, §5.5).
  Clients must not construct or inspect it.
- **A cursor belongs to its query.** The hash covers every parameter except `cursor` and `per_page`;
  a cursor presented with different parameters is `400 INVALID_PARAMETER`. `per_page` may change
  between pages. A cursor that does not decode is `400 INVALID_PARAMETER`.
- **Stability.** Paging visits every matching row exactly once while the data is unchanged, at any
  depth. A row inserted or deleted mid-traversal affects only its own position, never which other
  rows are served — the guarantee offsets cannot give.
- `total` and the summary are recomputed on every page and describe the match set as it is at that
  request.

### 5.4 Ordering

- `sort` chooses the order (values per endpoint, §10). `direction` is `asc` (default) or `desc`.
  `nulls` is `first` or `last` (default `last`) and places records whose sort key is null.
- **The id always breaks ties**, in the same direction as the sort, so every order is total and
  stable.
- **No sort removes a record** (⑦). A record with no usable date sorts by `start` or `end` with a
  null key, and appears where `nulls` puts it.
- Keyset comparison honours `nulls`: the cursor records whether the last key was null, and the next
  page continues within the null group or crosses into it as the order requires.

### 5.5 `limit` and `sample`

- **`limit=N`** (1 or more) caps the result at the first N rows in the current order, across pages:
  with `limit=1200&per_page=500`, pages of 500, 500 and 200, then `next_cursor: null`. `total` and
  the summary still describe the whole match set (⑧). There is no random behaviour in `limit`.
- **`sample=N`** (1 to 5,000) returns N rows chosen pseudo-randomly from the match set, ordered by
  `md5(id::text || seed)`, in one page with no cursor. `seed` is an optional string up to 64
  characters; without it the server generates one and echoes it, so the same sample can be requested
  again while the data is unchanged. `sample` cannot be combined with `sort`, `direction`, `nulls`,
  `limit` or `cursor` (`400`). Fewer than N matches returns them all. `total` and the summary
  describe the match set, not the sample.

### 5.6 Responses

JSON, `Content-Type: application/json; charset=utf-8`, encoded with unescaped slashes and Unicode,
not pretty-printed. A v2 response is **never a redirect** (②).

---

## 6. Dates

### 6.1 `v2_udate`

Every date in v2 — a record's `datestart` or `dateend`, a `date_from`/`date_to` bound, an
extended-data `before`/`after` operand — is read by one function. After trimming whitespace it
accepts exactly these forms:

| Form | Example | Period named |
| --- | --- | --- |
| `Y` (one or more digits) | `1856`, `400` | that year (CE) |
| `-Y` (one or more digits) | `-400`, `-51021` | that year BCE |
| `[-]Y-MM` | `1856-03` | that month |
| `[-]Y-MM-DD` | `1856-03-04` | that day |
| `[-]Y-MM-DDThh:mm[:ss[.fff]]` | `1856-03-04T14:30` | that instant |
| `D/M/YYYY` (one or two digits for D and M) | `4/3/1856` | that day — day first |
| `[-]Y-00-00` | `1856-00-00` | that year |
| `00/00/Y` | `00/00/1856` | that year |

- **Years.** There is no year zero: `0` and `-0` are unparseable. `-Y` is Y BCE, astronomical year
  `1 − Y`. Years of any length are accepted.
- **Calendar.** Proleptic Gregorian, UTC, leap years by the Gregorian rule on the astronomical year.
  Months 01–12, days valid for the month (`-16112-11-31` is unparseable), hours 00–23, minutes and
  seconds 00–59; fractional seconds are truncated to milliseconds.
- **Result.** Signed epoch milliseconds of the first instant of the period. Anything else — any other
  form, an out-of-range field, text — is `NULL`.

Reference values, computed independently of PostgreSQL:

| Input | `v2_udate` |
| --- | --- |
| `1970` | 0 |
| `1856` | −3,597,523,200,000 |
| `1856-03` | −3,592,339,200,000 |
| `1856-03-04` | −3,592,080,000,000 |
| `4/3/1856` | −3,592,080,000,000 |
| `400` | −49,544,438,400,000 |
| `1` | −62,135,596,800,000 |
| `-1` | −62,167,219,200,000 |
| `-400` | −74,758,377,600,000 |
| `-51021` | −1,672,202,880,000,000 |
| `2000-02-29` | 951,782,400,000 |

The largest magnitudes stay well inside a bigint and inside JavaScript's safe-integer range
(±9×10¹⁵, about 285,000 years either side of 1970). Beyond that range the function returns `NULL`.

v1 differs: it uses Sydney local time, misreads `-Y`, and returns `0` or `null` for failures. The
v2 documentation lists this (§14).

### 6.2 Periods

A date string names a period, not an instant: `1856` is all of 1856. `v2_udate_end` gives the
period's last millisecond (`1856` → 1856-12-31T23:59:59.999Z); for a value with a time it equals
`v2_udate`.

For each record:

```
s = v2_udate(datestart)        e = v2_udate(dateend)          -- first instants, or NULL
S = v2_udate_end(datestart)    E = v2_udate_end(dateend)      -- last instants, or NULL
lo = least(coalesce(s, e), coalesce(e, s))
hi = greatest(coalesce(E, S), coalesce(S, E))
```

A missing side is filled from the other, as v1's layer feeds already do. A record whose end precedes
its start still covers `[lo, hi]`; its stored text is returned unchanged.

### 6.3 Dated, undated, unparseable

- A record is **dated** if `lo` is not null — at least one side parses.
- It is **undated** otherwise, in one of two kinds:
  - `no_date` — both `datestart` and `dateend` are null or blank;
  - `unparseable_date` — at least one has text and neither parses.
- Separately, a record has an **unparseable date** if either side has text that does not parse, even
  if the other side does. Such a record is dated from its good side, and is counted (§8), so a
  half-garbled date is reported rather than quietly treated as a single point.

### 6.4 Date filters

`date_from` and `date_to` are validated with `v2_udate`; one that does not parse is
`400 INVALID_PARAMETER`. Either may be given alone. A dated record matches if its period overlaps the
range of periods:

```
(date_to   absent or lo <= v2_udate_end(date_to))
and (date_from absent or hi >= v2_udate(date_from))
```

So `date_from=1820&date_to=1860` keeps a record dated `1860-06` and one dated `1820`. `date_from`
later than `date_to` is a `400`.

`include_undated` (default **`true`**) decides undated records when a date filter is present: they
are kept unless it is `false` (⑦). With no date filter it has no effect, and is echoed as given.
Every summary reports the undated count, so its effect is visible either way.

### 6.5 Output values

Each feature carries `udatestart = s` and `udateend = e` — each from its **own** column (⑥),
`null` where that side is empty or unparseable. The infill of §6.2 is for filtering, sorting and
summaries only; output never invents a value the record does not hold.

---

## 7. Record filters

`RecordQuery` applies these, in place search and in the catalogue's record-level filters. Every given
filter must match (logical AND).

### 7.1 Visibility

A record is visible to v2 if `dataset_id` is null (a gazetteer record) or its dataset has
`public = true`. Nothing in v2 ever returns a record outside that set, and no parameter widens it.

### 7.2 `sources`

A comma list of `layers`, `anps`, `ncg`, `text`; default all four. Each maps to a `datasource` row by
`search_param_name` (`searchpublicdatasets`, `searchausgaz`, `searchncg`, `searchgeocoder`), looked up
once per request — never by hard-coded id. An unknown value or an empty list is a `400`. Duplicates
are allowed and collapse.

### 7.3 `name` and `match`

`name` is a non-empty string up to 200 characters. `match` is `exact` (default), `contains` or
`fuzzy`; giving `match` without `name` is a `400`.

Every mode tests **both `title` and `placename`** — unlike v1, whose `name` misses `placename`:

| `match` | A record matches when |
| --- | --- |
| `exact` | `title` or `placename` equals `name`, case-insensitively (`ILIKE` with `%`, `_` and `\` escaped — no wildcards) |
| `contains` | `title` or `placename` contains `name`, case-insensitively (escaped as above) |
| `fuzzy` | `similarity(title, name) ≥ 0.3` or `similarity(placename, name) ≥ 0.3` |

The fuzzy threshold is v2's, from `config/api_v2.php`, set for the query with
`set_config('pg_trgm.similarity_threshold', '0.3', true)` inside a transaction so the `%` operator and
the trigram index serve it. It does not depend on the database's default. Results are not ranked by
similarity in A1; the score itself is ⑨, Phase B.

### 7.4 `layers`

A comma list of up to 100 positive integer layer ids. Records must belong to one of them. Each id is
checked first: a missing layer is `404 LAYER_NOT_FOUND`, a private one `403 LAYER_PRIVATE`, naming
the id (the first offending id, in the order given). A requested layer is never silently dropped from
a result. Gazetteer records have no layer, so `layers` excludes them.

### 7.5 `bbox`

`minLon,minLat,maxLon,maxLat`, four finite numbers, longitudes in [−180, 180], latitudes in [−90, 90],
`minLat ≤ maxLat`. A record matches if `geom && ST_MakeEnvelope(…, 4326)` — boundary included.
`minLon > maxLon` means the box crosses the antimeridian and is matched as two envelopes. Records
without coordinates never match a spatial filter.

### 7.6 `polygon`

`lon lat,lon lat,…`: at least three distinct vertices, at most 1,000 (configurable), each a finite
pair in range. The ring is closed if the last vertex differs from the first. It is built with bound
values (`ST_MakePolygon(ST_MakeLine(…))` over parameters), never interpolated into SQL. If
`ST_IsValid` fails, the request is `400 INVALID_GEOMETRY` with `ST_IsValidReason`. A record matches if
`ST_Intersects(geom, polygon)`. A polygon may not cross the antimeridian; `bbox` can. `bbox` and
`polygon` together must both match.

### 7.7 `state`, `lga`, `feature_term`, `record_type`

One value each, up to 200 characters, matched exactly and case-insensitively against the record's
field (`record_type` against `recordtype.type`). Exact means the whole value: `lga=CESSNOCK` matches
`Cessnock`, and `lga=Cessnock Council` matches neither. Vocabulary endpoints that list the valid
values are ⑪, A2.

### 7.8 `extended_data`

A strict grammar. Any input that does not parse is `400 FILTER_REJECTED` with the expression, the
character position where parsing failed, and what was expected there. Nothing is ever discarded.

```
filter   := clause ( "AND" clause )*            -- AND is upper case and surrounded by whitespace
clause   := field op value
field    := quoted | bare                       -- bare: everything before the operator, trimmed
op       := "=" | ">" | "<"                     -- whitespace optional around these
          | "textmatch" | "before" | "after"    -- whitespace required around these, any case
value    := quoted | bare                       -- bare: up to the next " AND " or the end, trimmed
quoted   := '"' chars '"' | "'" chars "'"       -- no escapes; a value containing its quote uses the other
```

`Years>50`, `Years > 50` and `"Years" > "50"` parse to the same clause. A field name containing an
operator character, or a value containing ` AND `, must be quoted. An empty field or value is
rejected. At most 10 clauses.

| Operator | Value must be | A record matches when its field value |
| --- | --- | --- |
| `=` | any string | equals it exactly (case-sensitive) |
| `textmatch` | any string | contains it, case-insensitively |
| `>` / `<` | a decimal number (`-12`, `3.5`) | is a decimal number (after trimming) and is greater / less |
| `before` | a date `v2_udate` accepts | is such a date and its period ends before the operand's begins: `v2_udate_end(v) < v2_udate(x)` |
| `after` | a date `v2_udate` accepts | is such a date and its period begins after the operand's ends: `v2_udate(v) > v2_udate_end(x)` |

A value of the wrong kind on the request side (`Years > many`) is `400 FILTER_REJECTED`. On the data
side, a record whose field is absent, empty, or not of the operator's kind does not match — that is
what the filter asks — and the field is read with `v2_extended_value`, so malformed stored XML makes
that record not match rather than failing the request.

---

## 8. The summary

Every place search and layer-features response carries `summary`, computed in SQL over the **whole
match set** — not the page, not the `limit`, not the sample (④). Layer detail carries the same
figures for the layer's records, without `by_layer`.

```json
{
  "total": 273,
  "extent": [150.8012, -33.0974, 151.3990, -32.6011],
  "no_coordinates_count": 0,
  "date_range": { "start": -4417977600000, "end": 1609459199999 },
  "undated": { "total": 191, "no_date": 188, "unparseable_date": 3 },
  "unparseable_dates": 5,
  "by_source": { "layers": 273, "anps": 0, "ncg": 0, "text": 0 },
  "by_layer": [
    { "layer_id": 461, "name": "…", "count": 69, "warning": null },
    { "layer_id": 2749, "name": "…", "count": 2, "warning": "Aboriginal and/or Torres Strait Islander Peoples are advised…" }
  ]
}
```

(Values illustrative.)

| Field | Rule |
| --- | --- |
| `total` | Count of the match set. |
| `extent` | `[minLon, minLat, maxLon, maxLat]` of `ST_Extent(geom)` over matching records with coordinates; `null` if none have any. |
| `no_coordinates_count` | Matching records without coordinates — present in results with `geometry: null` (§9), and counted so the gap is stated. |
| `date_range` | `{start: min(lo), end: max(hi)}` over dated matching records (§6.2) — the span of periods covered, so `end` is a period's last millisecond; `null` if none is dated. |
| `undated` | Matching records with no usable date, split as §6.3. |
| `unparseable_dates` | Matching records with at least one side present but unparseable (§6.3), dated or not. |
| `by_source` | Count per source, all four keys always present. |
| `by_layer` | One entry per contributing layer, ordered by `layer_id`: its name, its count in the match set, and its `warning` **verbatim** — `null` only when the field is null or blank, so `"None"` is returned as `"None"`. Every layer a search draws on appears with its warning; no result travels without it. |

It is two queries — the totals and the per-layer group — over the same filtered set. With
`summary_only=true`, place search runs only these and builds no features.

---

## 9. Features

Records are GeoJSON features. Geometry is `Point [longitude, latitude]`, or `null` for a record
without both coordinates — included, not dropped (v1's layer feed drops them). Coordinates are
numbers. Properties, all always present, `null` where the record has nothing:

| Property | Value |
| --- | --- |
| `id` | The record's `uid`, e.g. `t1a2b` — the citable identifier |
| `name` | `title` |
| `placename` | `placename` |
| `description` | `description`, verbatim (it may hold HTML from the contributor; v2 does not alter it) |
| `datasource` | `layers`, `anps`, `ncg` or `text` |
| `layer_id` | The layer's id, or `null` for gazetteer records |
| `record_type` | `recordtype.type` |
| `state`, `lga`, `parish`, `feature_term` | As stored |
| `datestart`, `dateend` | As stored, verbatim text |
| `udatestart`, `udateend` | §6.5 — each from its own column (⑥) |
| `source` | The record's `source` text, verbatim |
| `external_url` | `external_url` |
| `image_url` | The public URL of `image_path`, as v1 builds it (`Storage::disk('public')->url('images/' . $path)`) — the URL, not an `<img>` element |
| `glycerine_url` | `glycerine_url` — the URL, not an anchor |
| `url` | The record's TLCMap page, `{APP_URL}/places/{uid}` |
| `created_at`, `updated_at` | ISO 8601 |
| `extended_data` | An object of field name → value, every field the record's XML holds, values as **plain strings** — no HTML escaping, no link wrapping. `{}` when there is none. Malformed XML yields `{}` and the record's `extended_data_unreadable: true` |
| `extended_data_unreadable` | `true` only when the stored XML could not be parsed; otherwise `false` |

Extended data is namespaced under `extended_data` (⑤), so a contributor's field named `description`
or `source` never replaces the built-in one. No property is derived by guessing.

---

## 10. Endpoints

### 10.1 `GET /api/v2/places` — place search

Searches visible records across all four sources (③ ④ ⑤ ⑥ ⑦ ⑧).

| Parameter | |
| --- | --- |
| `name`, `match` | §7.3 |
| `sources` | §7.2 |
| `layers` | §7.4 |
| `bbox`, `polygon` | §7.5, §7.6 |
| `date_from`, `date_to`, `include_undated` | §6.4 |
| `state`, `lga`, `feature_term`, `record_type` | §7.7 |
| `extended_data` | §7.8 |
| `summary_only` | `true` returns `{query, summary}` only; default `false` |
| `sort` | `id` (default), `title`, `start` (key `coalesce(s, e)`), `end` (key `coalesce(e, s)`) — first instants, §6.2; undated records have a null key |
| `direction`, `nulls` | §5.4 |
| `limit`, `sample`, `seed` | §5.5 |
| `cursor`, `per_page` | §5.3; `per_page` default 500, max 5,000 |

Response:

```json
{
  "type": "FeatureCollection",
  "query":   { "name": "Wollombi", "match": "exact", "sources": ["layers","anps","ncg","text"], "…": "…" },
  "summary": { "total": 4, "…": "…" },
  "page":    { "returned": 4, "per_page": 500, "next_cursor": null, "next": null },
  "features": [ { "type": "Feature", "geometry": { "type": "Point", "coordinates": [151.14, -32.94] },
                  "properties": { "id": "a1353c", "…": "…" } } ]
}
```

A search with no filters at all is allowed: it pages through every visible record.

### 10.2 `GET /api/v2/layers` — the catalogue

Lists public layers (①).

| Parameter | |
| --- | --- |
| `q` | Contains, case-insensitive, across layer `name`, `description` and subject keywords |
| `bbox`, `polygon`, `date_from`, `date_to`, `include_undated` | **Record-level**: a layer matches if at least one of its visible records matches all the given record filters, through `RecordQuery` (§3.2). `bbox` therefore tests records, never an extent — a layer whose extent overlaps the box but which has no record in it does not match |
| `record_type` | The layer's record type, exact, case-insensitive |
| `creator` | Contains, case-insensitive |
| `has_license`, `has_warning` | `true`/`false`: whether `license` / `warning` is non-blank — presence only, never what it says (§3.3); `"None"` is present |
| `min_records` | Layers with at least this many records (whole layer) |
| `sort` | `id` (default) or `name` |
| `direction` | §5.4 |
| `cursor`, `per_page` | §5.3; `per_page` default 100, max 1,000 |

Response:

```json
{
  "query": { "…": "…" },
  "total": 36,
  "page": { "returned": 36, "per_page": 100, "next_cursor": null, "next": null },
  "layers": [
    {
      "id": 461,
      "name": "…",
      "description": "…",
      "creator": "…",
      "record_type": "Other",
      "record_count": 17917,
      "matching_count": 69,
      "bbox": [112.9, -43.6, 153.6, -10.4],
      "date_range": null,
      "undated": { "total": 17917, "no_date": 17917, "unparseable_date": 0 },
      "has_license": true,
      "has_warning": false,
      "url": "https://tlcmap.org/layers/461"
    }
  ]
}
```

(Values illustrative.)

- `record_count`, `bbox`, `date_range` and `undated` describe the **whole layer**, computed from its
  records — never the declared fields.
- `matching_count` is the layer's records matching the record-level filters; present only when one
  of `bbox`, `polygon`, `date_from`, `date_to` is given, otherwise `null`.
- `bbox` is `null` for a layer with no coordinates; `date_range` `null` for one with no dated record.
- Layers with no records appear (with `record_count: 0`) unless a record-level filter or
  `min_records` excludes them.
- `total` counts matching layers. The rights fields themselves are on layer detail; the catalogue is
  for discovery and reports their presence.

### 10.3 `GET /api/v2/layers/{id}` — layer detail

Metadata and a record summary, without features. `403 LAYER_PRIVATE` if not public,
`404 LAYER_NOT_FOUND` if absent.

```json
{
  "layer": {
    "id": 206,
    "name": "Music communities",
    "description": "…",
    "subject_keywords": ["music"],
    "record_type": "Other",
    "creator": "…", "publisher": null, "contact": null,
    "citation": null, "doi": null, "source_url": null, "linkback": null,
    "license": null, "rights": null, "language": "en",
    "warning": "None",
    "declared": {
      "latitude_from": null, "latitude_to": null, "longitude_from": null, "longitude_to": null,
      "temporal_from": null, "temporal_to": null
    },
    "created": "2021-03-01",
    "created_at": "2021-03-01T02:11:09Z",
    "updated_at": "2023-05-12T07:40:51Z",
    "url": "https://tlcmap.org/layers/206"
  },
  "summary": { "total": 46, "extent": ["…"], "…": "…" }
}
```

- **Every field is always present**; `null` where the stored value is null or blank. (v1 removes
  empty keys, so absence and emptiness look alike.)
- `license`, `rights`, `citation`, `warning`, `creator` and the other text fields are **verbatim** —
  not trimmed, normalised or interpreted (§3.3). Blank means only empty or whitespace.
- `declared` holds the contributor-entered extents and temporal fields as stored: a hint, never used
  for filtering, known to be malformed on some layers ([§2.2](../DESIGN.md#22-the-catalogue-measured)).
- `summary` is §8 over the layer's records, without `by_layer`.

### 10.4 `GET /api/v2/layers/{id}/features` — a layer's records

`403`/`404` as §10.3. Parameters: `sort` — `order` (default; `dataset_order`, the contributor's
order), `id`, `title`, `start`, `end` — with `direction`, `nulls`, `limit`, `cursor` and `per_page`
(default 500, max 5,000). No record filters, no `sample`, no `line`: a layer's records within a region
are a place search with `layers` and `bbox`, and a route or timeline line is drawn by the client from
the ordered points.

The response is place search's envelope with the layer's metadata added:

```json
{
  "type": "FeatureCollection",
  "layer":   { "…": "the §10.3 layer object" },
  "query":   { "…": "…" },
  "summary": { "…": "§8 over the whole layer, without by_layer" },
  "page":    { "…": "…" },
  "features": [ "…" ]
}
```

So the feed carries its rights fields and warning with it, on every page.

---

## 11. Errors

Every non-2xx v2 response is JSON with the same body, and no v2 response is a redirect (②):

```json
{ "error": { "code": "INVALID_PARAMETER", "message": "…", "details": { "parameter": "per_page", "rule": "max", "accepts": "1–5000" } } }
```

| Status | Code | Raised when | `details` |
| --- | --- | --- | --- |
| 400 | `INVALID_PARAMETER` | Unknown, repeated, empty, ill-typed or out-of-range parameter; invalid combination; bad cursor | `parameter`, `rule`, `accepts` |
| 400 | `INVALID_GEOMETRY` | `polygon` invalid per PostGIS | `parameter`, `reason` (from `ST_IsValidReason`) |
| 400 | `FILTER_REJECTED` | `extended_data` does not parse, or an operand is the wrong kind | `expression`, `position`, `expected` |
| 403 | `LAYER_PRIVATE` | A requested layer exists and is not public | `layer_id` |
| 404 | `LAYER_NOT_FOUND` | A requested layer does not exist | `layer_id` |
| 404 | `NOT_FOUND` | No v2 route matches | `path` |
| 405 | `METHOD_NOT_ALLOWED` | A method other than `GET` (and `OPTIONS`, §12) | `method` |
| 429 | `RATE_LIMITED` | The v2 throttle (§12) | `limit`, `retry_after`; plus the `Retry-After` header |
| 500 | `INTERNAL_ERROR` | Anything else | none — no message text, no stack trace; the exception is logged |

Messages are Australian English, written for a person reading a log. `details` carries what a client
needs to act on. The codes match the names the MCP server already maps
([A0 spec §7](./A0-SERVER-SKELETON.md#7-errors)).

**`413` is reserved.** DESIGN §5.2 ② lists `413` for a request too large to answer at once. Every A1
request is bounded by `per_page` or `sample`, both capped and validated, so no A1 request can be too
large and nothing raises `413`; an oversized `per_page` is a `400`. The code stays reserved for a
future bound.

---

## 12. Rate limit and CORS

**Throttle.** A subclass of Laravel's `ThrottleRequests` whose request signature is prefixed with
`api-v2|`, so v2 requests never count against v1's per-address counter, nor v1's against v2's. The
limit is `config('api_v2.rate_limit')`, from `API_V2_RATE_LIMIT`, default `60,1` — the value the
`api` group already uses. It is an interim value: deliberate limits, and a quota for the MCP server
granted on something a client cannot claim, are ⑱ in Phase C. The MCP server's upstream must not move
to v2 on a deployment where this limit would throttle it; that is recorded for the MCP spec (§16).

**CORS.** A v2 middleware sets `Access-Control-Allow-Origin: *`, `Access-Control-Allow-Methods: GET,
OPTIONS` and `Access-Control-Allow-Headers: Content-Type` on every v2 response, errors included, and
answers `OPTIONS` preflight with `204` through an explicit `OPTIONS` route in `routes/api_v2.php` —
Laravel's automatic `OPTIONS` response bypasses route middleware, so it would carry no CORS headers.
No credentials header: v2 is public and cookieless.

---

## 13. Testing

### 13.1 The test database

A dedicated database, `tlcmap_test`, on the local PostgreSQL. `phpunit.xml` gains a `V2` testsuite
(`tests/V2`) and points tests at it (`DB_DATABASE=tlcmap_test`). `DB_SCHEMA` stays unset, as in the
application: the extensions live in `public`, and every table is named with its `tlcmap.` schema.

An artisan command, `tlcmap:test-db`, (re)builds it and is the documented first step:

1. Create `tlcmap_test` if it does not exist; drop and recreate its `tlcmap` and `public` schemas.
2. `CREATE EXTENSION IF NOT EXISTS postgis` and `pg_trgm`, in `public`.
3. Run every migration from scratch.
4. Seed the four `datasource` rows (with their production `search_param_name` values) and the
   `recordtype` rows (`Other`, `Placename`, `Media`, `Text`, `Event`, `Person`, `Journey`, `Site`,
   `Organisation`).

The command refuses to run unless the target database name ends in `_test`. Each test runs inside a
transaction rolled back at the end (`DatabaseTransactions`) and builds its own layers, records and
keywords with fixture builders that insert rows directly, with explicit ids — sequences do not roll
back, so ids from them would make golden files unrepeatable, and the `Dataitem` model's date mutator
would normalise the malformed dates the tests need. No test depends on data it did not create.

Whether every migration runs from scratch is **unproven** — the base schema appears to have come from
a MySQL conversion rather than the migrations (§2). The plan's first task establishes it, and fixes or works around a migration that does not run
only inside the test bootstrap, never by changing a migration production has already run.

Tests run with PHP 7.4 (`C:\laragon\bin\php\php-7.4.19-Win32-vc15-x64\php.exe vendor/bin/phpunit`),
the production version, so PHP 8-only syntax fails locally.

### 13.2 Unit tests

The parameter parsers, `ParameterSet`'s unknown/repeated/empty handling, `Cursor` encode/decode and
query binding, `Sort`'s keyset predicates, and the `extended_data` grammar — every row of §7.8,
including each rejection with its position.

### 13.3 SQL function tests

`v2_udate` and `v2_udate_end` against a table of every form in §6.1, every reference value, the
boundary cases (leap days, `0`, `-0`, month 13, day 31 in a 30-day month, `-16112-11-31`, blank,
text), and values beyond the safe-integer range. Expected values are computed in PHP by an
independent days-from-civil routine in the test, not by calling the function under test.
`v2_extended_value` with present, absent, empty and malformed XML.

### 13.4 Feature tests

HTTP tests against the four endpoints, organised so every DESIGN item is pinned and named:

| Item | Asserted |
| --- | --- |
| ① | Catalogue `bbox` finds a layer with one record in the box and not a layer whose extent overlaps it without a record; computed `bbox`, `date_range`, `record_count` match the records; declared fields ignored; every filter of §10.2 |
| ② | Private → `403`, missing → `404`, bad `extended_data` → `400 FILTER_REJECTED`, unknown parameter → `400`, unknown route → `404` JSON; no response is a `3xx` |
| ③ | Paging a match set larger than `per_page` visits every record exactly once, in order, for each sort and both `nulls`; cursor bound to its query |
| ④ | Summary figures equal those computed from the fixture by hand, independent of the page; `summary_only` |
| ⑤ | A contributor field named `description` sits under `extended_data` and does not replace `description` |
| ⑥ | `udateend` comes from `dateend`, not `datestart` |
| ⑦ | Sorting keeps undated records (placed by `nulls`); date filters keep undated records by default and drop them only with `include_undated=false` |
| ⑧ | `limit` returns the same first N on repeated requests and across pages; `sample` with a seed is repeatable |

Plus the **catalogue–search invariant**: for a set of filter combinations over a fixture of several
layers, the catalogue's layer ids equal the distinct `layer_id`s of the place search with the same
record filters, paged to the end.

### 13.5 The v1 guard

Before any v2 migration is added, characterisation tests capture v1's responses on fixture data —
`/places?…&format=json` for a name, a bbox and a date search; `/layers/{id}/json`; `/layers/json` —
as committed golden files. They must still pass, byte for byte, after every v2 change. Volatile parts
(none are expected on fixed fixtures beyond the request URL) are normalised in the comparison and
listed in the test.

### 13.6 Performance notes

Not a gate. Against the local `temp_tlcmap` copy, `EXPLAIN ANALYZE` and wall time are recorded for:
the unfiltered catalogue; the catalogue and place search for the Hunter Valley bbox
(`150.8,-33.1,151.4,-32.6`); a gazetteer-wide fuzzy search; a deep page of an unfiltered search; an
`extended_data` filter without `layers`. They go in the acceptance record, scaled by the layer count
as an estimate for production, so a slow path is known before release.

---

## 14. Documentation

On the `api-v2` branch of `tlcmap-docs`, following that repository's conventions (source code as the
primary reference, Australian English, never "GHAP"):

- **A v2 section in the developer docs** — an overview (base URL, JSON only, no redirects, strict
  parameters, the query echo, paging, errors), and a page per endpoint with every parameter, the
  response shape and examples.
- **Dates** — the accepted forms, periods, BCE, UTC, undated and unparseable.
- **Errors** — the table of §11.
- **Differences from v1** — every behaviour that answers the same question differently, required by
  [DESIGN.md §5.1](../DESIGN.md#51-two-versions): `name` matches `placename`; dated searches keep
  undated records; `sort` drops nothing; `limit` is deterministic; extended data is namespaced and
  plain text; `udateend` is correct; dates in UTC with BCE read correctly; records without
  coordinates included; honest status codes; no 5,000 ceiling.
- The developer index links to v2 and states that v2 is unstable until declared stable.

Examples in the docs are produced by running the documented request against the local application
(`temp_tlcmap`), not written by hand, and trimmed only by removing features or array entries.

---

## 15. Acceptance

This half of A1 is done when:

1. The code, migrations and tests are merged on `tlcmap-api-enhancement`.
2. `tlcmap:test-db` builds the test database from nothing, and the `V2` suite and the v1 guard pass
   under PHP 7.4.
3. Every item ① to ⑧ has the tests of §13.4, and the catalogue–search invariant test passes.
4. The documentation of §14 is committed on `api-v2` in `tlcmap-docs`.
5. An acceptance record, `documents/acceptance/A1-API-V2.md` in this repository, maps each of ① to
   ⑧ to the tests that pin it, records the commits, and holds the §13.6 performance notes.

---

## 16. What the MCP server can rely on

The MCP half of A1 is written against this contract. In summary:

- **Upstream URLs** become `/api/v2/layers`, `/api/v2/layers/{id}`, `/api/v2/layers/{id}/features`
  and `/api/v2/places`; each result's provenance records the v2 request.
- **Errors by status**: `403 LAYER_PRIVATE` and `404 LAYER_NOT_FOUND` go live
  ([A0 spec §14](./A0-SERVER-SKELETON.md#14-api-dependencies)); `400` codes carry the parameter or
  expression for `FILTER_REJECTED` and `INVALID_GEOMETRY`; nothing returns `413` in A1.
- **Tool inputs map one to one** onto §10's parameters, with v2's snake_case names. The MCP `cursor`
  is v2's cursor, passed through.
- **Summaries come from v2**: `total`, `extent`, `date_range`, `by_source` and `by_layer` (the tool's
  `sources_breakdown` and `layers_present`) — the server computes none of them.
- **Layer features are paged**, so `tlcmap://layer/{id}/features` is assembled from pages; the layer
  object, with its warning, is on every page.
- `line` is not offered; `tlcmap_get_layer` drops it (§17).
- **Rate limit**: the interim v2 limit is 60 a minute per address (§12). The server's upstream must
  not move to v2 where that would throttle it; raising it is configuration until ⑱.

---

## 17. Changes to DESIGN.md

Made in the same commit as this spec:

- **§5.2 ①** — the catalogue signature uses v2's names and paging:
  `GET /api/v2/layers?q=&bbox=&polygon=&date_from=&date_to=&include_undated=&record_type=&creator=&has_license=&has_warning=&min_records=&cursor=&per_page=`.
- **§5.2 ②** — a note under the table that `413` is reserved: A1 bounds every request with
  `per_page`, so nothing raises it yet.
- **§6.3 `tlcmap_get_layer`** — `line` removed from its inputs; layer features are paged, and a client
  draws lines from the ordered points.
- **§10 A1** — a pointer to this spec, as A0 has.
- The README's documents table gains this spec and its plan.
