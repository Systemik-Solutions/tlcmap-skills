# A1 — MCP server on API v2: discovery and rights

**Status:** spec for build · **Date:** 2026-11-06 · **Repository:** `tlcmap-mcp` (branch `a1-mcp`)

The MCP half of milestone A1 in [DESIGN.md §10](../DESIGN.md#phase-a--the-vertical-slice):

> *MCP:* `tlcmap_list_layers`, `tlcmap_search_places`, `tlcmap_get_layer`,
> `tlcmap_get_layer_metadata` — HTTP transport, provenance envelope, typed errors. The server's
> upstream moves from the current API to v2.

This document expands that line into something buildable, on the server
[A0](./A0-SERVER-SKELETON.md) built and against the contract the
[A1 API spec](./A1-API-V2.md) delivered. Where A0's rules still hold they are not restated; where
this spec changes one, it says so. DESIGN.md remains the authority on principles; where this spec
and DESIGN.md disagree, one of them is wrong and is fixed.

---

## Contents

| | |
| --- | --- |
| [1. Purpose and scope](#1-purpose-and-scope) | What this delivers, what it leaves out |
| [2. Facts this spec rests on](#2-facts-this-spec-rests-on) | Measured against local v2 |
| [3. Components](#3-components) | What changes in the server |
| [4. Rules every tool follows](#4-rules-every-tool-follows) | Inputs, outputs, URLs, response checks |
| [5. The tools](#5-the-tools) | The four tools |
| [6. Resources](#6-resources) | One handle per page, named by content hash |
| [7. Provenance](#7-provenance) | The envelope on every result |
| [8. Cache](#8-cache) | What is held, and for how long |
| [9. Errors](#9-errors) | v2's errors at the tool layer |
| [10. Configuration and deployment](#10-configuration-and-deployment) | Three environments, three datasets |
| [11. Testing](#11-testing) | Unit, integration, the compatibility check |
| [12. Acceptance](#12-acceptance) | Local now; staging and production at v2's release |
| [13. Changes to other documents](#13-changes-to-other-documents) | Made with this spec |

---

## 1. Purpose and scope

A0 fixed the server's shape against the current API. This milestone moves it onto v2 and gives it
the four tools the vertical slice's discovery and rights work needs
([DESIGN.md §9](../DESIGN.md#9-the-vertical-slice)): finding the layers in a region, finding their
records, and reading each layer's rights before its data.

**In scope**

- The upstream moves from v1 (`/layers/{id}/json`) to v2 (`/api/v2/…`). Nothing in the server calls
  v1 afterwards.
- Four read tools: `tlcmap_list_layers`, `tlcmap_search_places`, `tlcmap_get_layer` (rewritten) and
  `tlcmap_get_layer_metadata`.
- Two resource templates for pages of features (§6), replacing A0's.
- v2's error codes at the tool layer (§9).
- Fixtures captured from local v2, unit and integration tests, and a compatibility check that runs
  against any environment.
- An acceptance on the local stack, and the staging and production checks written down for v2's
  release.
- Version 0.2.0.

**Out of scope**

- Deploying 0.2.0 anywhere. Staging keeps serving 0.1.0 until v2 is on its upstream (§10).
- Any change to the TLCMap application or to v2. Where v2 is inadequate, the fix is v2's
  ([§3.8](../DESIGN.md#38-fix-it-upstream)).
- Any tool beyond these four, including `tlcmap_list_vocabulary` and `tlcmap_layer_statistics`
  (A2), and the six sketched tools (Phase C).
- Assembling a layer or a search from several pages into one payload (§6).
- Write tools, authentication, tokens (Phase E).
- The skills `tlcmap-search` and `tlcmap-cite`, and the ground-truth harvest — the other halves of
  A1.

---

## 2. Facts this spec rests on

Verified 2026-11-06.

**v2 is not deployed.** `GET /api/v2/layers/206` is `404` on both `https://tlcmap.org` and
`https://test-ghap.tlcmap.org`. It runs only on the local application: `tlcmap-ghap` at `9209514` on
`tlcmap-api-enhancement`, served by Laragon at `http://tlcmap-ghap.laragon.local` against the local
database `temp_tlcmap` ([A1 API acceptance](../acceptance/A1-API-V2.md)).

**The three environments hold different data.** The same layer id can be a different layer, or
absent:

| Layer | Local (`temp_tlcmap`, v2) | Staging (`test-ghap`, v1) | Production (`tlcmap.org`, v1) |
| --- | --- | --- | --- |
| 206 | Music communities; `warning` `"None"`; no licence | Music communities; `"None"` | Music communities; `"None"` |
| 258 | WA Journey Ways – Aboriginal Camps Around WA; 13 records; licence `"Do not use without permission."`; an advisory warning | missing or private | same name and licence |
| 472 | New begining; 4 records; `warning` `<p>Thus <strong>this</strong> is a <em>warning</em></p>` | missing or private | a different layer (James Cooks 2nd journey) |
| 1125 | missing | Chamorro Materials; licence `"Open (subject to…"` | 50 words project; licence `"Closed (subject to…"` |
| 2749 | missing | missing or private | the HTML warning A0 used |
| 33 | private (29 records) | missing or private | — |
| 99999999 | missing | missing | missing |

Locally there are 155 public layers, with ids up to 475; production has 2,118. Fixtures and
acceptance checks therefore name layers by the property they exercise, and each environment's record
names its own layers (§11, §12).

**v2's responses**, as served locally:

| Endpoint | Top-level keys |
| --- | --- |
| `GET /api/v2/layers` | `query`, `total`, `page`, `layers` |
| `GET /api/v2/layers/{id}` | `layer`, `summary` |
| `GET /api/v2/layers/{id}/features` | `type`, `layer`, `query`, `summary`, `page`, `features` |
| `GET /api/v2/places` | `type`, `query`, `summary`, `page`, `features` |
| `GET /api/v2/places?summary_only=true` | `query`, `summary` |

- `page` is `{returned, per_page, next_cursor, next}`. With `sample`, it is one page with
  `next_cursor: null`.
- `per_page` defaults to 500 on features and 100 on the catalogue (A1 API spec §5.3).
- Layer 206's features at `per_page=30` are two pages (30, then 22), and `summary.total` is 52 on
  both.
- The Hunter Valley box `150.8,-33.1,151.4,-32.6` matches 8 layers in the local catalogue (84, 102,
  140, 170, 206, 379, 461, 462) and 1,318 records in place search (104 from layers, 685 ANPS, 529
  NCG). Production holds 36 layers there ([DESIGN.md §9.3](../DESIGN.md#93-what-is-actually-there--measured)).
- Every error is `{"error": {"code", "message", "details"}}` (A1 API spec §11). Measured locally:
  `404 LAYER_NOT_FOUND` with `details.layer_id`; `403` for layer 33; `400 INVALID_PARAMETER` with
  `parameter`, `rule`, `accepts`; `400 FILTER_REJECTED` with `expression`, `position`, `expected`;
  `400 INVALID_GEOMETRY` with `parameter`, `reason`.
- Responses carry `Content-Type: application/json; charset=utf-8` and `Cache-Control: no-cache,
  private`, and no `ETag` (⑭ is Phase C).
- URLs inside responses (`page.next`, `layer.url`, each feature's `url`) carry the serving host, so
  locally captured bodies name `tlcmap-ghap.laragon.local`.

**v2's number grammar.** `bbox` and `polygon` coordinates must match `-?[0-9]+(\.[0-9]+)?`
(`app/Api/V2/Params/Bbox.php:10`): no exponent, no leading `+`, no bare `.5`.

**v2's rate limit.** 60 requests a minute per client address by default, from `API_V2_RATE_LIMIT`
in `config/api_v2.php` (A1 API spec §12). The local `.env` does not set it. The real quota, ⑱, is
Phase C.

**The SDK and runtime** are A0's, unchanged: `@modelcontextprotocol/server` 2.4.0 with the Express
and Node adapters, zod 4, Node 24 (`package.json`).

---

## 3. Components

The structure A0 built stays: one upstream client, the byte-capped cache, single-flight, the
concurrency limiter, scopes, provenance, typed errors, stateless Streamable HTTP. What changes is
what sits behind the client.

```
src/
  upstream/client.ts       unchanged role; error mapping replaced by v2's (§9)
  v2/url.ts                NEW  typed inputs → canonical v2 URL (§4.3)
  v2/schemas.ts            NEW  zod schemas for v2's objects (§4.4)
  pages/service.ts         NEW  replaces layers/service.ts: fetch, check, cache, provenance
  tools/list-layers.ts     NEW
  tools/search-places.ts   NEW
  tools/get-layer.ts       rewritten
  tools/get-layer-metadata.ts  NEW
  resources/pages.ts       NEW  replaces resources/layer-features.ts (§6)
  summarise/layer.ts       DELETED — v2 computes every summary
```

**`pages/service.ts`** is one generic fetcher. A tool gives it a v2 URL, a response schema and the
checks of §4.4; it returns the validated response without its features, the ids, the exact body
bytes and the provenance, through the cache, single-flight and limiter. It is the only caller of
`upstream/client.ts`, and it keeps A0's upstream statistics for `/healthz` (§9.3).

**`summarise/layer.ts` is deleted.** v2 computes `total`, `extent`, `date_range`, the undated and
no-coordinates counts, `by_source` and `by_layer` in SQL over the whole match set (A1 API spec §8,
§16), so the server computes none of them. A0's `ungeolocated_count` becomes v2's
`no_coordinates_count`, and A0's `date_range` with `as_written` becomes v2's, in epoch milliseconds;
the stored text is on each feature as `datestart` and `dateend`.

**The v1 code is removed**: the `/layers/{id}/json` URL builder, the v1 identity check and the v1
fixtures.

---

## 4. Rules every tool follows

### 4.1 Inputs

- **Every input has v2's name and meaning** (A1 API spec §10, §16). The tool's description and its
  schema say what v2 says; a tool never gives a v2 parameter a different meaning.
- **JSON types where v2 takes text.** `bbox` is an array of four numbers; `polygon` an array of
  `[lon, lat]` pairs, at least three; `sources` an array of `layers`, `anps`, `ncg`, `text`;
  `layers` an array of positive integers; booleans are booleans; integers are integers. Dates
  (`date_from`, `date_to`) are strings, read by v2's date rules (A1 API spec §6.1), so `"1856"`,
  `"-400"` and `"1856-03"` are all dates.
- **Type and form only.** The schema checks types, integer-ness, enum values, the arity of `bbox`
  and of each `polygon` vertex, and `layer_id` in 1 to 2147483647. It does **not** repeat v2's
  range and combination rules — the `per_page` maximum, at most 100 `layers`, `sample` with `sort`,
  a cursor from another query, `date_from` later than `date_to`, a polygon's validity. v2 rejects
  those with a `400` naming the parameter, and the agent sees it (§9). Repeating them would mean
  changing the server whenever v2 changed a rule.
- **Unknown inputs are rejected**, as in A0 (`z.strictObject`).
- **Absent means absent.** An input not given is not sent. The server adds no defaults; v2 applies
  its own, and its `query` echo shows the values that applied.

### 4.2 Outputs

- **v2's objects pass through.** A tool's `structuredContent` carries v2's own objects — `layer`,
  `summary`, `query`, `page`, `layers`, `total` — under v2's names, exactly as parsed from the
  response. The server adds `resource_uri` and `provenance`, and nothing else.
- **Nothing is computed, renamed or dropped.** `license`, `rights`, `citation` and `warning` are
  whatever v2 returned: `"None"` stays `"None"`, HTML stays HTML, `null` stays `null`
  ([§3.3](../DESIGN.md#33-attribution-travels-permission-is-never-guessed),
  [§6.6](../DESIGN.md#66-what-the-server-must-not-do)).
- **Features never appear inline** ([§6.2](../DESIGN.md#62-design-rules) rule 2). They are behind
  the page's resource handle (§6).
- **The output schema** is the tool's zod schema built from §4.4's object schemas, published as the
  tool's JSON Schema.
- **The text content block is the serialised JSON of `structuredContent`.** The MCP specification
  recommends this for backwards compatibility, and it means a client that reads only `content` still
  receives the rights text verbatim. A0's one-line summary is replaced.

### 4.3 URLs

`v2/url.ts` builds every upstream URL. The URL is the cache key (§8) and the provenance `request`
(§7), so it is canonical:

- `{TLCMAP_BASE_URL}` with trailing slashes removed, then `/api/v2/…`.
- Parameters in a fixed order per endpoint — the order of v2's `query` echo (A1 API spec §10) — so
  identical calls produce identical URLs.
- Each value is percent-encoded with `encodeURIComponent`, except that `,` is left literal: it is
  legal in a query, it is v2's list separator, and it keeps provenance readable. A comma inside a
  single-valued string such as `name` is harmless, because v2 does not split those.
- Lists are joined with `,`; a polygon vertex is `lon lat`, vertices joined with `,`; booleans are
  `true`/`false`.
- **Numbers are written as plain decimals**, never in exponent form, because v2's number grammar
  has none (§2). JavaScript writes `1e-7` for `0.0000001`; the builder writes `0.0000001`. Digits are
  the shortest that round-trip the number; `-0` is written `0`.
- The cursor is v2's `next_cursor`, passed through untouched.

### 4.4 Response checks

Every `2xx` body must satisfy, before anything is cached or returned:

- **It is JSON and matches the endpoint's schema** (`v2/schemas.ts`). Each object schema requires
  the keys A1 API spec §8–§10 define, with their types, and **admits extra keys**
  (`z.looseObject`), so a field v2 adds later passes through to the tool result rather than failing
  it ([§6.5](../DESIGN.md#65-tool-surface-stability): additive changes are free).
- **Every feature has a non-empty string `properties.id`** — the record UID provenance cites.
- **It is about what was asked.** For both layer tools, `layer.id` equals the requested `layer_id`
  (A0's identity check, kept). For `tlcmap_get_layer`, every feature's `properties.layer_id` equals
  it too.

A body that fails any of these is `UPSTREAM_CONTRACT_VIOLATION` (§9), and is not cached.

---

## 5. The tools

All four are in the `read` scope (A0 §8).

### 5.1 `tlcmap_list_layers`

The layer catalogue — regional discovery ([DESIGN.md §5.2 ①](../DESIGN.md#52-tier-1--required-by-a1)).

**Upstream:** `GET /api/v2/layers` (A1 API spec §10.2).

| Input | Type |
| --- | --- |
| `q` | string |
| `bbox` | `[minLon, minLat, maxLon, maxLat]` |
| `polygon` | `[[lon, lat], …]` |
| `date_from`, `date_to` | string |
| `include_undated`, `has_license`, `has_warning` | boolean |
| `record_type`, `creator` | string |
| `min_records` | integer |
| `sort` | `id` \| `name` |
| `direction` | `asc` \| `desc` |
| `cursor` | string |
| `per_page` | integer |

**Output:** `{query, total, page, layers, provenance}`. The catalogue entries are summaries, not
bulk, so they are returned in full and there is no resource handle. `bbox`, `polygon` and the date
filters are record-level in v2 — a layer matches when one of its records does, never by a declared
extent — and the description says so.

### 5.2 `tlcmap_search_places`

Place search across all four sources (③ ④ ⑤ ⑥ ⑦ ⑧).

**Upstream:** `GET /api/v2/places` (A1 API spec §10.1).

| Input | Type |
| --- | --- |
| `name` | string |
| `match` | `exact` \| `contains` \| `fuzzy` |
| `sources` | array of `layers` \| `anps` \| `ncg` \| `text` |
| `layers` | array of integers |
| `bbox`, `polygon` | as §5.1 |
| `date_from`, `date_to` | string |
| `include_undated`, `summary_only` | boolean |
| `state`, `lga`, `feature_term`, `record_type`, `extended_data` | string |
| `sort` | `id` \| `title` \| `start` \| `end` |
| `direction` | `asc` \| `desc` |
| `nulls` | `first` \| `last` |
| `limit`, `sample` | integer |
| `seed`, `cursor` | string |
| `per_page` | integer |

**Output:** `{query, summary, page, resource_uri, provenance}`.

- `summary` describes the whole match set, on every page. `summary.by_layer` names each contributing
  layer with its `warning` verbatim; for licence and rights the agent calls
  `tlcmap_get_layer_metadata` on those ids.
- With `summary_only: true`, v2 returns no page and no features: `page` and `resource_uri` are
  `null`, and `provenance.record_uids` is `[]`.

### 5.3 `tlcmap_get_layer`

One page of a layer's records, with the layer's metadata and the summary of the whole layer.

**Upstream:** `GET /api/v2/layers/{layer_id}/features` (A1 API spec §10.4).

| Input | Type |
| --- | --- |
| `layer_id` | integer, 1 to 2147483647 — required |
| `sort` | `order` \| `id` \| `title` \| `start` \| `end` |
| `direction` | `asc` \| `desc` |
| `nulls` | `first` \| `last` |
| `limit` | integer |
| `cursor` | string |
| `per_page` | integer |

**Output:** `{layer, query, summary, page, resource_uri, provenance}`.

A layer of any size is read by following `page.next_cursor`: one call, one v2 request, one page.
The layer's records within a region are a `tlcmap_search_places` query with `layers` and `bbox`, so
this tool takes no geometry filter.

**This redefines A0's tool in place**, rather than shipping a new name as
[§6.5](../DESIGN.md#65-tool-surface-stability) requires of a published tool. A0's tool runs only on
staging and nothing has been built on it — no skill, fixture or prompt names it — and v2 is not
stable ([§5.1](../DESIGN.md#51-two-versions)). From this milestone on, §6.5 applies.

### 5.4 `tlcmap_get_layer_metadata`

A layer's rights, licence, citation and warning, and the summary of its records, without features.
Cheap by design, so an agent can check rights before fetching data
([§3.3](../DESIGN.md#33-attribution-travels-permission-is-never-guessed)).

**Upstream:** `GET /api/v2/layers/{layer_id}` (A1 API spec §10.3).

| Input | Type |
| --- | --- |
| `layer_id` | integer, 1 to 2147483647 — required |

**Output:** `{layer, summary, provenance}`.

### 5.5 Descriptions

Each tool's description states, besides what it does:

- that licence, rights and warning text is returned verbatim and must be shown to the user, not
  paraphrased;
- for the three tools with a `summary`, that it describes the whole match set or layer, not the
  page, and that its undated and no-coordinates counts are what lets exclusions be reported
  ([§3.4](../DESIGN.md#34-report-what-was-excluded));
- for the two paged feature tools, that records are behind `resource_uri` and further pages are
  reached with `page.next_cursor`.

---

## 6. Resources

Each page of features has a handle named by the SHA-256 of its bytes:

```
tlcmap://layer/{id}/features/{sha256}     a page from tlcmap_get_layer
tlcmap://search/{sha256}                  a page from tlcmap_search_places
```

Both are resource templates; `resources/list` returns no entries. A handle is obtained from a tool
result.

- **A read serves the exact bytes v2 sent for that page** — the whole response, with its `layer`
  (warning included), `query`, `summary`, `page` and `features` — as UTF-8 text with
  `mimeType: application/json`. The read's `_meta.provenance` is the page's provenance (§7), with
  `cache: "hit"` and the same `sha256` the tool reported.
- **A read never fetches.** When the server no longer holds the page — TTL elapsed, evicted, or the
  process restarted — the read fails with `RESOURCE_EXPIRED`, naming the tool to call again
  ([§6.4](../DESIGN.md#64-resources)). This is A0's rule, kept.
- **The hash names the content, not the request.** A URI can only ever name one set of bytes, so a
  newer fetch never changes what an old URI returns; two calls that receive identical bytes get the
  same URI; and a provenance record cites a specific state of the data.
- **`{id}` must be a canonical layer id** (A0 §5) and **`{sha256}` 64 lowercase hex characters**.
  Anything else is `INVALID_ARGUMENT`. A well-formed URI the server does not hold — including a
  layer id that does not match the held page — is `RESOURCE_EXPIRED`.

A0's `tlcmap://layer/{id}/features` is removed. With one page per call it could not tell pages or
sort orders apart.

**Pages are not assembled.** A whole layer or search is the sequence of its pages, each with its own
handle and provenance. Assembling them in the server would make one call cost many upstream
requests, serve bytes TLCMap never sent, and join pages that may disagree if the data changes in
between.

---

## 7. Provenance

Every tool result and every resource read carries the envelope
([§3.2](../DESIGN.md#32-provenance-by-construction), [§6.2](../DESIGN.md#62-design-rules) rule 4):

```json
{
  "request": "GET http://tlcmap-ghap.laragon.local/api/v2/layers/206/features?per_page=30",
  "fetched_at": "2026-11-06T04:12:09.123Z",
  "cache": "miss",
  "sha256": "…hex digest of the exact response bytes…",
  "bytes": 61234,
  "record_uids": ["ta4f5", "…"],
  "server": "tlcmap-mcp 0.2.0"
}
```

(Values illustrative.)

| Field | Rule |
| --- | --- |
| `request`, `fetched_at`, `cache`, `sha256`, `bytes`, `server` | As A0 §6. `request` is the v2 URL of §4.3, reproducible with `curl`. |
| `record_uids` | On `tlcmap_search_places` and `tlcmap_get_layer`: every feature's `properties.id` on the page, in order, never truncated. `[]` for `summary_only`. |
| `layer_ids` | On `tlcmap_list_layers`: every returned entry's `id`, in order. On `tlcmap_get_layer_metadata`: `[layer_id]`. |

Each envelope has exactly one of `record_uids` and `layer_ids`.

---

## 8. Cache

A0's cache, with its limits unchanged (`CACHE_MAX_BYTES`, `CACHE_TTL_S`, least recently used first,
bodies as `Buffer`s outside the heap, errors never cached).

- **Keyed by the canonical v2 URL** (§4.3), for every endpoint — catalogue and metadata responses
  too, so a repeated call does not reach TLCMap again within the TTL.
- **An entry holds** the body bytes, the parsed response without its `features`, the ids for
  provenance, the provenance itself and, for a features page, its resource URI. A hit never
  re-parses the body.
- **A second index maps each resource URI to its entry.** Evicting or expiring the entry drops the
  URI with it.
- Single-flight, the concurrency cap and the queue (A0 §9) apply to every endpoint alike.

---

## 9. Errors

### 9.1 The wire shape

A0's, unchanged: a tool failure is a tool result with `isError: true` and
`structuredContent: {error: {code, message, details}}`, and a resource-read failure is an MCP
protocol error carrying the same object in its `data` (A0 §7).

### 9.2 Mapping v2's errors

A0's hand-written map from status to code is replaced. The client reads v2's documented error body
(A1 API spec §11) — the status and `error.code`, which are the contract — never the message text to
decide anything.

| v2 responds | Tool error | `message` and `details` |
| --- | --- | --- |
| `400 INVALID_PARAMETER` | `INVALID_PARAMETER` | v2's message, unchanged. v2's details (`parameter`, `rule`, `accepts`), plus `request` and `status`. |
| `400 INVALID_GEOMETRY` | `INVALID_GEOMETRY` | As above (`parameter`, `reason`). |
| `400 FILTER_REJECTED` | `FILTER_REJECTED` | As above (`expression`, `position`, `expected`). |
| `403 LAYER_PRIVATE` | `LAYER_PRIVATE` | As above (`layer_id`). Also a private id in a search's `layers`. |
| `404 LAYER_NOT_FOUND` | `LAYER_NOT_FOUND` | As above (`layer_id`). Also a missing id in a search's `layers`. |
| `429 RATE_LIMITED`, any `5xx`, a timeout, a network failure | `UPSTREAM_UNAVAILABLE` | `request`, `status` if any, elapsed time, and `retry_after` from the `Retry-After` header if sent. |
| Any `3xx`; `404 NOT_FOUND` or `405 METHOD_NOT_ALLOWED`; any other status; a status whose body is not v2's error shape or whose `code` does not belong with that status; a `2xx` that fails §4.4 | `UPSTREAM_CONTRACT_VIOLATION` | `request`, `status`; `location` for a `3xx`; v2's body as `api_body` when it is JSON. |

`404 NOT_FOUND` and `405` mean the server built a URL v2 does not route, which is the server's
fault, not the caller's. Redirects are never followed (②).

v2's detail keys (A1 API spec §11) never include `request` or `status`, so adding them loses
nothing.

**Why v2's messages pass through.** A tool's inputs carry v2's names, so v2's message — "The
parameter 'per_page' …" — already names what the agent sent, and every rule the server does not
check itself (§4.1) reaches the agent with what to change.

### 9.3 The complete set

| Code | Source |
| --- | --- |
| `INVALID_ARGUMENT` | A resource URI that is not well formed (§6). Tool input that fails its schema is rejected by the SDK before the handler runs, as in A0. |
| `INVALID_PARAMETER`, `INVALID_GEOMETRY`, `FILTER_REJECTED` | New — v2's `400`s (§9.2). |
| `LAYER_PRIVATE`, `LAYER_NOT_FOUND` | v2's `403` and `404`; live from this milestone (A0 §14). |
| `UPSTREAM_CONTRACT_VIOLATION`, `UPSTREAM_UNAVAILABLE` | §9.2. |
| `RESPONSE_TOO_LARGE`, `SERVER_BUSY`, `RESOURCE_EXPIRED` | As A0 §7. |

`RESULT_TOO_LARGE` is not raised by any A1 tool: every v2 request is bounded by `per_page` or
`sample`, and v2 raises no `413` (A1 API spec §11).

**`/healthz`.** `upstream.last_error` records only errors that say something about upstream health —
`UPSTREAM_UNAVAILABLE`, `UPSTREAM_CONTRACT_VIOLATION`, `RESPONSE_TOO_LARGE`. A caller's `400`, `403`
or `404` is about the request and leaves it unchanged; `last_success_at` counts any response v2
answered as designed, error statuses included.

---

## 10. Configuration and deployment

**No new variables.** `TLCMAP_BASE_URL` now names an application that serves v2; the server appends
`/api/v2`. `env.example`, `DEPLOY.md` and the README say so.

**Three environments, each reading its own TLCMap:**

| Stage | MCP server | `TLCMAP_BASE_URL` | When |
| --- | --- | --- | --- |
| Local | `http://localhost:3100` | `http://tlcmap-ghap.laragon.local` | This milestone; fixtures are captured here |
| Staging | `https://test-mcp.tlcmap.org` | `https://test-ghap.tlcmap.org` | Once v2 is deployed to the staging application |
| Production | none in A1 (`mcp.tlcmap.org` is Phase C) | `https://tlcmap.org` | Once v2 is released; A1 runs only the compatibility check |

**Staging now reads the staging application, not production.** This reverses A0 §11, where staging
called production. It is a change to `/etc/tlcmap-mcp/env`, not to code, and it happens with the
deployment of 0.2.0. Until then staging serves 0.1.0 against production's v1, unchanged.

**Prerequisites for each deployment**, recorded in the acceptance record (§12):

- v2 deployed to that environment's application.
- `API_V2_RATE_LIMIT` there set for the MCP server's load. Paging makes requests add up: layer 461's
  17,917 records are four calls at `per_page=5000`. Until ⑱ this is configuration.
- `TrustProxies` configured for that application's proxy; otherwise v2 sees every client, the MCP
  server included, as one address and shares one rate-limit key among them (A1 API acceptance §6).
- On staging, one host serves both test-ghap and the MCP server, so the server's requests reach v2
  from that host's own address, and the limit applies to it.

**Locally**, the same `npm start` with `PUBLIC_URL=http://localhost:3100` and `TLCMAP_BASE_URL` as
above; Claude Code connects with
`claude mcp add --transport http tlcmap-local http://localhost:3100/mcp`. The application's local
`.env` sets `API_V2_RATE_LIMIT` high enough for testing (for example `600,1`), and
`php artisan config:clear` applies it if the configuration is cached.

**Delivery** is on branch `a1-mcp` in `tlcmap-mcp`, by pull request to `main`.

---

## 11. Testing

Vitest, built test-first, as A0. CI runs typecheck, lint, unit and integration tests on every push;
none of them touch the network.

### 11.1 Unit tests

- `v2/url` — every endpoint's parameter order; absent inputs omitted; list, polygon and boolean
  encoding; `,` literal and everything else percent-encoded; an `extended_data` value such as
  `"Years" > 50`; numbers in plain decimal (`0.0000001`, `1e21`-sized and negative values, `-0`);
  trailing slashes on the base URL.
- `v2/schemas` — every fixture passes; a missing required key fails; a wrong type fails; an extra
  key passes and is kept.
- `upstream/client` — every row of §9.2, from status and body to code, message and details;
  redirects not followed; the size cap; the `User-Agent`.
- `pages/service` — the cache key; the resource-URI index and its eviction; each check of §4.4;
  nothing cached on failure; `cache` outcomes `miss`, `hit`, `shared`.
- `provenance` — `record_uids` complete and in order; `layer_ids`; `sha256` and `bytes` over the
  exact body.
- `resources/pages` — URI parsing: canonical id, 64-hex hash, each rejection.

### 11.2 Fixtures

Captured from **local v2** by `npm run capture-fixtures` with `TLCMAP_BASE_URL` set, bytes stored as
received. `test/fixtures/CAPTURED.md` records for each the URL, host, status, capture time, size and
`sha256`, and states that the fixtures describe `temp_tlcmap` on that date, not production. The v1
fixtures are deleted.

| Fixture | Property exercised |
| --- | --- |
| Layer 206 detail; its features at `per_page=30`, both pages | `warning` `"None"`, `license` `null`; following a cursor |
| Layer 258 detail and features | a restrictive licence and an advisory warning |
| Layer 472 detail | an HTML warning, passed through byte for byte |
| Layer 33 detail | `403 LAYER_PRIVATE` |
| Layer 99999999 detail | `404 LAYER_NOT_FOUND` |
| Catalogue, Hunter Valley box | discovery |
| Place search, Hunter Valley box: a page at `per_page=20`, and `summary_only` | search, summary, `by_layer` warnings |
| `400` bodies: `per_page=6000`, `extended_data=Years > many`, a self-intersecting polygon | the three `400` codes |

Synthetic fixtures cover what local v2 does not show on demand: `429` with `Retry-After`, a `5xx`, an
error body that is not JSON, an error body with a code that does not belong with its status, a
layer response for the wrong id, a feature without `properties.id`, a feature of another layer, a
missing required key.

### 11.3 Integration tests

The real Express app in-process against `test/fake-tlcmap.ts`, which serves the fixtures by path
and query, driven by `@modelcontextprotocol/client` over Streamable HTTP:

- `tools/list` holds exactly the four tools; `resources/templates/list` exactly the two templates.
- Each tool's result for each fixture: v2's objects deep-equal to the fixture's, `layer.warning` and
  `layer.license` identical to the fixture's, provenance as §7, the text block equal to the
  serialised `structuredContent`.
- A resource read returns bytes whose `sha256` equals the tool's.
- Following `next_cursor` from page 1 of layer 206 to page 2 visits all 52 ids once.
- A second identical call is a cache hit with the original `fetched_at`.
- Every error path of §9 end to end, and `RESOURCE_EXPIRED` and `INVALID_ARGUMENT` on reads.
- A0's transport, health and rate-limit tests continue to pass.

### 11.4 The compatibility check

`npm run test:prod` becomes **`npm run test:compat`**: run by hand, never in CI, against
`TLCMAP_BASE_URL` (default `https://tlcmap.org`). It asserts the v2 behaviours the server depends
on, using only layers that behave the same in every environment (§2):

- layer 206's detail and features responses have the shapes of §4.4, `warning` is `"None"`, and
  `layer.id` is 206;
- a features page's records have string `properties.id` and the requested `layer_id`;
- layer 99999999 is `404` with v2's error body and code `LAYER_NOT_FOUND`;
- `per_page=0` on layer 206's features is `400` with v2's error body and code `INVALID_PARAMETER`.

It runs against local v2 in this milestone, against staging and production at their releases, and
from then on it is how the server learns that v2 changed
([§4.3](../DESIGN.md#43-packaging-and-distribution)).

---

## 12. Acceptance

Recorded with the date in `tlcmap-mcp/docs/acceptance/A1.md`, beside A0's, in three parts. Each part
names the layers that play each role in its own dataset.

| Role | Local |
| --- | --- |
| `warning` `"None"`, no licence | 206 |
| a restrictive licence | 258 |
| an HTML warning | 472 |
| a private layer | 33 |
| a layer that needs many pages | 461 (17,917 records) |
| a missing layer | 99999999 |
| the region | the Hunter Valley box `150.8,-33.1,151.4,-32.6` |

### 12.1 Local — this milestone

Claude Code connected to the local server (§10), which reads local v2:

1. CI is green on the pull request, and `npm run test:compat` passes against local v2.
2. Claude Code connects and lists exactly the four tools.
3. `tlcmap_list_layers` with the region's box gives the same `total` and layer ids as `curl` of the
   same v2 request.
4. `tlcmap_get_layer_metadata` returns the rights fields byte for byte as v2 does: the `"None"`
   layer's warning and `null` licence; the restrictive layer's licence and warning; the HTML layer's
   warning.
5. `tlcmap_get_layer` on 206 at `per_page=30`, then on its `next_cursor`, gives 52 distinct record
   ids, equal to `summary.total`; on 461 at `per_page=5000`, four calls give 17,917.
6. `tlcmap_search_places` with the region's box and `summary_only` gives v2's summary; a page's
   resource read has the tool's `sha256`.
7. Errors, each carrying v2's details: `tlcmap_get_layer_metadata` gives `LAYER_PRIVATE` for the
   private layer and `LAYER_NOT_FOUND` for the missing one; `tlcmap_search_places` gives
   `FILTER_REJECTED` for `extended_data: "Years > many"`; `tlcmap_get_layer` on 206 gives
   `INVALID_PARAMETER` for `per_page: 6000`.
8. After restarting the server, reading an earlier resource URI returns `RESOURCE_EXPIRED`.

Plus the merge of `a1-mcp` into `main`. This completes the milestone.

### 12.2 Staging — at v2's deployment to test-ghap

Pending, with its prerequisites (§10). Deploy 0.2.0 to `test-mcp.tlcmap.org` with `TLCMAP_BASE_URL`
`https://test-ghap.tlcmap.org`; run `test:compat` against test-ghap; choose staging's layers for each
role and record them; repeat checks 2 to 8; and check A0 §13's deploy and rollback for the new
release.

### 12.3 Production — at v2's release

Pending, with its prerequisites (§10). Run `test:compat` against `https://tlcmap.org` and record the
result. Production's layers for each role — 206, 1125 for the restrictive licence, 2749 for the HTML
warning, a private layer still to be identified — are confirmed and recorded then, ready for Phase
C's `mcp.tlcmap.org`.

---

## 13. Changes to other documents

Made in the same commit as this spec, in this repository:

- **DESIGN.md §6.2 rule 2** — the example result becomes v2's summary and a page handle.
- **DESIGN.md §6.3** — the four specified tools' inputs, outputs and errors revised to §5: v2's
  names (`q` for `query`, `name` for `name_query`, `layers` for `layer_ids[]`, `per_page` for
  `tlcmap_list_layers`'s `limit`); v2's objects as outputs; `places[]` dropped; `cursor`,
  `per_page`, `direction` and `limit` added to `tlcmap_get_layer`; `RESULT_TOO_LARGE` removed from
  A1's tools.
- **DESIGN.md §6.4** — `tlcmap://search/{query_hash}` and `tlcmap://layer/{id}/features` become the
  page handles of §6.
- **DESIGN.md §10 A1** — a pointer to this spec.
- **A1 API spec §16** — "assembled from pages" becomes one handle per page, with a pointer here.
- **README** — the status, and this spec and its plan in the documents table.

No heading changes, so no anchor in either document moves.
