# A1 — MCP server on API v2: implementation plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development
> (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use
> checkbox (`- [ ]`) syntax for tracking.

**Goal:** Move `tlcmap-mcp`'s upstream from v1 to API v2 and give it four read tools —
`tlcmap_list_layers`, `tlcmap_search_places`, `tlcmap_get_layer` (rewritten) and
`tlcmap_get_layer_metadata` — that pass v2's objects through, one page per call, each page behind a
handle named by its content hash; accepted on the local stack.

**Architecture:** A0's stack stays: one upstream client behind a limiter, single-flight and a
byte-capped cache; a fresh `McpServer` per request from a factory. A pure URL builder turns typed
tool inputs into canonical v2 URLs; zod schemas check v2's responses and let extra keys through; one
generic page service fetches, checks, caches and builds provenance for every endpoint. Tools build a
URL, call the page service and return v2's objects plus `resource_uri` and `provenance`.

**Tech Stack:** A0's, unchanged — TypeScript `~6.0.3` (ES modules, strict), Node 24,
`@modelcontextprotocol/server` `^2.4.0` with the Express and Node adapters, Express 5, zod `^4.6.7`,
Vitest `^5.1.0` with `@modelcontextprotocol/client`, `tsx` for scripts. No new dependency.

**Spec:** [A1-MCP.md](../specs/A1-MCP.md). Read it first; this plan argues from it and cites it as
"spec §n". It builds on [A0-SERVER-SKELETON.md](../specs/A0-SERVER-SKELETON.md) ("A0 §n") and
targets the contract in [A1-API-V2.md](../specs/A1-API-V2.md) ("API §n").

**Where the work happens:** the `tlcmap-mcp` repository (local clone `C:\AIProject\tlcmap-mcp`), on
a branch `a1-mcp` cut from `main` at `da4c7fa`. All paths below are relative to that repository
unless stated. Task 2 and Task 11 need the local TLCMap application running under Laragon at
`http://tlcmap-ghap.laragon.local` (`tlcmap-ghap` at `9209514` on `tlcmap-api-enhancement`, database
`temp_tlcmap`).

## Global Constraints

- All of A0's global constraints still hold (A0 plan): Node `>=24 <25`, TypeScript `~6.0.3`, ES
  modules with `.js` relative imports, LF line endings, UTF-8 without BOM, one trailing newline,
  Australian English in prose and messages, never "GHAP".
- No unit or integration test touches the network; only `npm run test:compat` does (spec §11).
- The server calls only `/api/v2/…`. No v1 URL, fixture or code survives Task 7 (spec §1).
- Tool inputs carry v2's parameter names and meanings; schemas check type and form only, never v2's
  ranges or combination rules (spec §4.1). `layer_id` is the one range: 1 to 2147483647.
- Absent inputs are not sent; the server adds no defaults (spec §4.1).
- `structuredContent` is v2's objects as parsed plus `resource_uri` and `provenance`; nothing
  computed, renamed or dropped; `license`, `rights`, `citation`, `warning` verbatim (spec §4.2).
- The text content block is `JSON.stringify(structuredContent)` (spec §4.2).
- Features never inline; one v2 request per tool call; pages never assembled (spec §6).
- Response schemas use `z.looseObject`, so extra v2 keys pass through (spec §4.4).
- v2's status and `error.code` decide the error code; message text is never inspected (spec §9.2).
- Redirects never followed (`redirect: 'manual'`), as A0.
- Server identity `tlcmap-mcp 0.2.0` from `package.json` (spec §1).

## Review Focus

Inputs and conditions the spec implies but does not spell out, most likely to bite first. Each has
a test in the task named.

1. **Non-ASCII and quote characters in a search** — a `name` like `Ngunnawal – ‘Canberra’` or an
   `extended_data` of `"Years" > 50`. The URL must percent-encode them exactly once, so the cache key,
   the provenance `request` and what v2 decodes all agree. *Task 1, `encodes non-ASCII, quotes and
   spaces once`.*
2. **A cursor reused with changed filters.** An agent changes `bbox` but keeps the previous
   `next_cursor`. v2's `400 INVALID_PARAMETER` must reach the agent naming `cursor`, not be swallowed
   or retried. *Task 8, `a cursor from another query is INVALID_PARAMETER naming cursor`.*
3. **`false` is a value.** `include_undated: false` or `summary_only: false` must be sent as
   `false`, never dropped as falsy, since dropping `include_undated: false` silently keeps undated
   records. *Task 1, `false is sent, not omitted`.*
4. **A handle outliving its cache entry, or shared by two entries.** An evicted page's URI must read
   as `RESOURCE_EXPIRED`, and when two URLs returned identical bytes, evicting the older must not
   break the newer's handle. *Task 5, `evicting an entry drops its resource URI` and `a URI shared by
   two entries survives the older one's eviction`.*
5. **An empty layer.** A public layer with no records has a features page with `features: []`; the
   tool must return its layer and summary, an empty `record_uids` and a readable handle, not an error.
   *Task 7, `an empty layer returns its page and an empty record_uids`.*

---

## File structure

```
src/
  v2/url.ts                   NEW   formatNumber, v2Url, the four path builders, PARAM_ORDER
  v2/schemas.ts               NEW   zod schemas for v2 objects and responses; parseV2; identity checks
  pages/service.ts            NEW   PageService: fetch → check → cache → provenance; URI index; stats
  tools/common.ts             NEW   runTool, provenanceSchema, shared input schemas
  tools/get-layer.ts          REWRITTEN
  tools/get-layer-metadata.ts NEW
  tools/list-layers.ts        NEW
  tools/search-places.ts      NEW
  resources/pages.ts          NEW   the two page templates, URI parsing
  errors.ts                   MODIFIED  three new codes
  provenance.ts               MODIFIED  record_uids | layer_ids
  upstream/client.ts          MODIFIED  v2 error mapping
  upstream/cache.ts           MODIFIED  onEvict
  mcp/server.ts, wire.ts, http/app.ts   MODIFIED  wiring
  layers/service.ts, summarise/layer.ts, resources/layer-features.ts   DELETED (Task 7)
scripts/capture-fixtures.ts   REWRITTEN for v2
test/fixtures/v2/             NEW   captured v2 bodies, manifest.json, CAPTURED.md
test/fixtures/*.json, test/fixtures/synthetic/, test/fixtures/CAPTURED.md   DELETED (Task 7)
test/helpers/v2-fixtures.ts   NEW   fixture loaders and a fetch stub serving them
test/fake-tlcmap.ts           MODIFIED  path+query routes from the manifest
test/compat/compat.test.ts    NEW   replaces test/prod/compat.test.ts
vitest.compat.config.ts       NEW   replaces vitest.prod.config.ts
```

---

### Task 1: The v2 URL builder

**Files:**
- Create: `src/v2/url.ts`
- Test: `test/unit/v2-url.test.ts`

**Interfaces:**
- Produces:

```ts
export type ParamValue = string | number | boolean | readonly number[] | readonly string[]
  | readonly (readonly [number, number])[];
export type Params = Record<string, ParamValue | undefined>;
export const PARAM_ORDER: {
  places: readonly string[];        // name, match, sources, layers, bbox, polygon, date_from, date_to,
                                    // include_undated, state, lga, feature_term, record_type,
                                    // extended_data, summary_only, sort, direction, nulls, limit,
                                    // sample, seed, cursor, per_page
  layers: readonly string[];        // q, bbox, polygon, date_from, date_to, include_undated,
                                    // record_type, creator, has_license, has_warning, min_records,
                                    // sort, direction, cursor, per_page
  layer_features: readonly string[]; // sort, direction, nulls, limit, cursor, per_page
};
export function formatNumber(n: number): string;          // plain decimal; RangeError if not finite
export function placesUrl(base: URL, params: Params): URL;          // /api/v2/places
export function layersUrl(base: URL, params: Params): URL;          // /api/v2/layers
export function layerUrl(base: URL, layerId: number): URL;          // /api/v2/layers/{id}
export function layerFeaturesUrl(base: URL, layerId: number, params: Params): URL; // …/{id}/features
```

- [ ] **Step 0: Branch** — in `C:\AIProject\tlcmap-mcp`: `git switch main && git pull && git switch -c a1-mcp`.
- [ ] **Step 1: Write the failing tests** in `test/unit/v2-url.test.ts`:

```ts
const base = new URL('https://tlcmap.org/');
test('formatNumber writes plain decimals', () => {
  expect(formatNumber(150.8)).toBe('150.8');
  expect(formatNumber(-33.1)).toBe('-33.1');
  expect(formatNumber(0.0000001)).toBe('0.0000001');
  expect(formatNumber(-2.5e-7)).toBe('-0.00000025');
  expect(formatNumber(1.5e-10)).toBe('0.00000000015');
  expect(formatNumber(1e21)).toBe('1000000000000000000000');
  expect(formatNumber(-0)).toBe('0');
  expect(() => formatNumber(NaN)).toThrow(RangeError);
});
test('parameters follow v2 echo order whatever the input order', () =>
  expect(placesUrl(base, { per_page: 20, bbox: [150.8, -33.1, 151.4, -32.6] }).href)
    .toBe('https://tlcmap.org/api/v2/places?bbox=150.8,-33.1,151.4,-32.6&per_page=20'));
test('absent inputs are omitted; no query string when none given', () =>
  expect(layersUrl(base, { q: undefined }).href).toBe('https://tlcmap.org/api/v2/layers'));
test('false is sent, not omitted', () =>
  expect(placesUrl(base, { date_from: '1820', include_undated: false, summary_only: false }).href)
    .toBe('https://tlcmap.org/api/v2/places?date_from=1820&include_undated=false&summary_only=false'));
test('lists join with literal commas; polygon vertices are "lon lat"', () => {
  expect(placesUrl(base, { sources: ['anps', 'ncg'], layers: [206, 258] }).search)
    .toBe('?sources=anps,ncg&layers=206,258');
  expect(placesUrl(base, { polygon: [[0, 0], [1, 1], [0, 1], [1, 0]] }).search)
    .toBe('?polygon=0%200,1%201,0%201,1%200');
});
test('encodes non-ASCII, quotes and spaces once', () => {
  expect(placesUrl(base, { name: 'Ngunnawal – ‘Canberra’' }).search)
    .toBe('?name=Ngunnawal%20%E2%80%93%20%E2%80%98Canberra%E2%80%99');
  expect(placesUrl(base, { extended_data: '"Years" > 50 AND Name textmatch \'a&b\'' }).search)
    .toBe('?extended_data=%22Years%22%20%3E%2050%20AND%20Name%20textmatch%20\'a%26b\'');
});
test('a cursor passes through untouched', ...);   // 'eyJ2IjoxfQ' → '?cursor=eyJ2IjoxfQ'
test('trailing slashes on the base are removed', () =>
  expect(layerUrl(new URL('http://tlcmap-ghap.laragon.local///'), 206).href)
    .toBe('http://tlcmap-ghap.laragon.local/api/v2/layers/206'));
test('layer features path', ...);   // layerFeaturesUrl(base, 206, {per_page: 30}).href ends '/api/v2/layers/206/features?per_page=30'
test('a parameter not in the endpoint order is a programming error', () =>
  expect(() => layersUrl(base, { nulls: 'last' })).toThrow());
```

- [ ] **Step 2: Run** `npx vitest run test/unit/v2-url.test.ts` — Expected: FAIL (module not found).
- [ ] **Step 3: Implement** `src/v2/url.ts`. Encoding is `encodeURIComponent(value).replaceAll('%2C', ',')`
  (spec §4.3); numbers inside lists and vertices go through `formatNumber` too. `formatNumber`:
  take `String(n)`; if it has no `e`, return it; otherwise split mantissa and exponent, remove the
  mantissa's point, and shift it by the exponent, padding with zeros — the digits are exactly
  `String`'s, which are the shortest that round-trip. Build the query string by hand (not
  `URLSearchParams`, which writes spaces as `+` and encodes commas).
- [ ] **Step 4: Run** — Expected: PASS. Then `npm run typecheck && npm run lint` — Expected: clean.
- [ ] **Step 5: Commit** — `git add src/v2/url.ts test/unit/v2-url.test.ts && git commit -m "Add the v2 URL builder"`.

---

### Task 2: Capture v2 fixtures from the local application

**Files:**
- Rewrite: `scripts/capture-fixtures.ts`
- Create: `test/fixtures/v2/*.json`, `test/fixtures/v2/manifest.json`, `test/fixtures/v2/CAPTURED.md`,
  `test/helpers/v2-fixtures.ts`
- Modify: `test/fake-tlcmap.ts`
- Test: `test/unit/fake-tlcmap.test.ts`

**Interfaces:**
- Consumes: Task 1's builders.
- Produces:

```ts
// test/helpers/v2-fixtures.ts
export interface ManifestEntry { file: string; path: string; status: number }  // path = pathname + search
export const MANIFEST: readonly ManifestEntry[];
export function v2Bytes(file: string): Buffer;
export function v2Json<T = Record<string, unknown>>(file: string): T;
/** A fetch that serves the manifest by pathname + search; overrides win; unknown → 404 empty body. */
export function fixtureFetch(overrides?: Record<string, { status: number; body: string | Buffer;
  headers?: Record<string, string> }>): typeof globalThis.fetch;
// test/fake-tlcmap.ts — routes are looked up by pathname + search first, then pathname
```

- [ ] **Step 1: Write the failing test** `test/unit/fake-tlcmap.test.ts`: a route keyed
  `/api/v2/layers/206/features?per_page=30` is served for exactly that query; a route keyed by
  pathname only is served for any query; the default table contains every manifest entry with its
  status and `Content-Type: application/json; charset=utf-8`.
- [ ] **Step 2: Rewrite `scripts/capture-fixtures.ts`.** It requires `TLCMAP_BASE_URL` (exits naming
  it when absent — never defaults to production), sends sequential GETs with no retries,
  `redirect: 'manual'`, `User-Agent: tlcmap-mcp/{version} (+capture-fixtures)`, and writes each body
  byte for byte. URLs are built with Task 1's builders, so they equal what the server will send. The
  captures, in this order:

| File | Request | Status |
| --- | --- | --- |
| `layer-206.json` | `layerUrl(206)` | 200 |
| `layer-206-features.json` | `layerFeaturesUrl(206, {})` | 200 |
| `layer-206-features-p1.json` | `layerFeaturesUrl(206, {per_page: 30})` | 200 |
| `layer-206-features-p2.json` | `layerFeaturesUrl(206, {cursor: <p1 page.next_cursor>, per_page: 30})` | 200 |
| `layer-258.json`, `layer-258-features.json` | detail; features `{}` | 200 |
| `layer-472.json` | detail | 200 |
| `layer-257-features.json` | features `{}` — a public layer with no records | 200 |
| `layer-33.json` | detail | 403 |
| `layer-99999999.json` | detail | 404 |
| `layers-hunter.json` | `layersUrl({bbox: [150.8, -33.1, 151.4, -32.6]})` | 200 |
| `places-hunter.json` | `placesUrl({bbox: …same, per_page: 20})` | 200 |
| `places-hunter-summary.json` | `placesUrl({bbox: …same, summary_only: true})` | 200 |
| `places-layers-33.json` | `placesUrl({layers: [33]})` | 403 |
| `error-per-page.json` | `layerFeaturesUrl(206, {per_page: 6000})` | 400 |
| `error-filter.json` | `placesUrl({extended_data: 'Years > many'})` | 400 |
| `error-geometry.json` | `placesUrl({polygon: [[0,0],[1,1],[0,1],[1,0]]})` | 400 |
| `error-cursor.json` | `placesUrl({bbox: [150, -33, 151, -32], cursor: <places-hunter next_cursor>, per_page: 20})` | 400 |
| `route-not-found.json` | `{base}/api/v2/nowhere` | 404 |

  A status other than the expected one stops the script with a message naming the file. It writes
  `manifest.json` (`[{file, path, status}]`) and `CAPTURED.md`: a header stating the host, that the
  data is the local `temp_tlcmap` database on the capture date and **not production data** (spec
  §11.2), and per file its URL, status, capture time (UTC), size and `sha256`. "Do not edit by hand."
- [ ] **Step 3: Capture.** With the local application up, from Git Bash:
  `TLCMAP_BASE_URL=http://tlcmap-ghap.laragon.local npm run capture-fixtures`.
  Expected: 19 lines, each file with its expected status; no stop. (19 requests fit v2's default 60 a
  minute.) Check `layer-206-features-p2.json` has `page.returned` 22 and `next_cursor` null.
- [ ] **Step 4: Implement** `test/helpers/v2-fixtures.ts` and the fake's lookup: `table[pathname +
  search] ?? table[pathname]`, with the manifest added to the default routes. The v1 routes stay
  until Task 7.
- [ ] **Step 5: Run** `npm test` — Expected: PASS (A0's tests are untouched).
- [ ] **Step 6: Commit** — `git add scripts/capture-fixtures.ts test/fixtures/v2 test/helpers/v2-fixtures.ts
  test/fake-tlcmap.ts test/unit/fake-tlcmap.test.ts && git commit -m "Capture v2 fixtures from the local application"`.

---

### Task 3: v2 response schemas and checks

**Files:**
- Create: `src/v2/schemas.ts`
- Test: `test/unit/v2-schemas.test.ts`

**Interfaces:**
- Produces (all objects `z.looseObject`; "text" is `z.string().nullable()`; "int" is `z.number().int()`):

```ts
export const pageSchema;          // {returned: int, per_page: int, next_cursor: string|null, next: string|null}
export const undatedSchema;       // {total, no_date, unparseable_date: int}
export const dateRangeSchema;     // {start: int, end: int} | null
export const summarySchema;       // {total: int, extent: [4 numbers]|null, no_coordinates_count: int,
                                  //  date_range, undated, unparseable_dates: int,
                                  //  by_source: {layers, anps, ncg, text: int}}  — by_layer optional
export const placesSummarySchema; // summarySchema + by_layer: [{layer_id: int, name: text, count: int, warning: text}]
export const layerSchema;         // API §10.3: id int; name, description, record_type, creator, publisher,
                                  //  contact, citation, doi, source_url, linkback, license, rights,
                                  //  language, warning: text; subject_keywords: string[];
                                  //  declared: {latitude_from, latitude_to, longitude_from, longitude_to:
                                  //  number|null, temporal_from, temporal_to: text}; created, created_at,
                                  //  updated_at: text; url: string
export const catalogueEntrySchema; // API §10.2 entry: id int; name, description, creator, record_type: text;
                                  //  record_count int; matching_count int|null; bbox [4]|null; date_range;
                                  //  undated; has_license, has_warning: boolean; url: string
export const featureSchema;       // {type: 'Feature', geometry: {type: 'Point', coordinates: [number, number]}|null,
                                  //  properties: API §9 — id: string.min(1); name, placename, description,
                                  //  record_type, state, lga, parish, feature_term, datestart, dateend, source,
                                  //  external_url, image_url, glycerine_url, created_at, updated_at: text;
                                  //  datasource: 'layers'|'anps'|'ncg'|'text'|null; layer_id: int|null;
                                  //  udatestart, udateend: int|null; url: string;
                                  //  extended_data: record<string,string>; extended_data_unreadable: boolean}
export const layerDetailResponse;   // {layer, summary}
export const layerFeaturesResponse; // {type: 'FeatureCollection', layer, query: looseObject, summary, page, features: feature[]}
export const catalogueResponse;     // {query, total: int, page, layers: catalogueEntry[]}
export const placesResponse;        // {type: 'FeatureCollection', query, summary: placesSummary, page, features}
export const placesSummaryOnlyResponse; // {query, summary: placesSummary}
export const errorBodySchema;       // {error: {code: string, message: string, details: looseObject}}
export function parseV2<S extends z.ZodType>(body: Buffer, schema: S): z.infer<S>;
  // fatal UTF-8 decode → JSON.parse → schema; any failure throws TlcmapError UPSTREAM_CONTRACT_VIOLATION
  // with details.expected = the first zod issue as "<path>: <message>" (or "UTF-8 JSON")
export function checkLayerIdentity(layer: { id: number }, layerId: number): void;   // CV unless equal
export function checkFeaturesOfLayer(features: { properties: { layer_id: number | null } }[], layerId: number): void;
export function withoutFeatures<T extends { features?: unknown }>(doc: T): Omit<T, 'features'>;
```

- [ ] **Step 1: Write the failing tests** `test/unit/v2-schemas.test.ts`:
  - Every 200 fixture in `MANIFEST` parses with its response schema; every non-200 fixture with
    `errorBodySchema`.
  - **Extra keys are kept:** `layerDetailResponse` on layer 206 with `layer.new_field: 1` added
    → parsed `layer.new_field === 1`.
  - Missing required key fails: `layer.warning` deleted → CV, `details.expected` starts `layer.warning`.
  - Wrong type fails: `summary.total: "52"` → CV.
  - A feature with `properties.id: ""` fails; with `geometry: null` passes.
  - `layer.warning` of 206 parses to `"None"`; of 472 to the exact fixture string
    `<p>Thus <strong>this</strong> is a <em>warning</em></p>`.
  - Bytes that are not UTF-8 (`Buffer.from([0xff])`) and text that is not JSON → CV, no throw of
    another kind.
  - `checkLayerIdentity({id: 206}, 206)` passes; `(…, 207)` throws CV with
    `expected: 'layer.id to equal 207'`. `checkFeaturesOfLayer` throws when one feature's
    `layer_id` is another layer or `null`, with `expected: 'every feature to have layer_id 206'`.
  - `withoutFeatures(doc)` drops `features` only and does not mutate `doc`.
- [ ] **Step 2: Run** `npx vitest run test/unit/v2-schemas.test.ts` — Expected: FAIL.
- [ ] **Step 3: Implement** `src/v2/schemas.ts`.
- [ ] **Step 4: Run** — Expected: PASS; typecheck and lint clean.
- [ ] **Step 5: Commit** — `git commit -m "Add v2 response schemas and checks"`.

---

### Task 4: v2 errors in the upstream client

**Files:**
- Modify: `src/errors.ts`, `src/upstream/client.ts` (`errorForStatus`)
- Test: `test/unit/upstream-client.test.ts`, `test/unit/errors.test.ts`

**Interfaces:**
- Consumes: `errorBodySchema` (Task 3).
- Produces: `ErrorCode` gains `'INVALID_PARAMETER' | 'INVALID_GEOMETRY' | 'FILTER_REJECTED'`.
  `UpstreamClient.get` keeps its signature; non-2xx statuses map per spec §9.2.

- [ ] **Step 1: Write the failing tests** — replace A0's 403/404 cases in
  `test/unit/upstream-client.test.ts` with a table driven by `fixtureFetch` and synthetic bodies:

| Response | Code | Asserted |
| --- | --- | --- |
| `error-per-page.json` (400) | `INVALID_PARAMETER` | `message` === the fixture's `error.message`; `details` equals `{...fixture details, request: 'GET <url>', status: 400}` |
| `error-geometry.json` (400) | `INVALID_GEOMETRY` | `details.reason` from the fixture |
| `error-filter.json` (400) | `FILTER_REJECTED` | `details.expression === 'Years > many'`, `details.position === 8` |
| `layer-33.json` (403) | `LAYER_PRIVATE` | `details.layer_id === 33` |
| `layer-99999999.json` (404) | `LAYER_NOT_FOUND` | `details.layer_id === 99999999` |
| `route-not-found.json` (404 `NOT_FOUND`) | `UPSTREAM_CONTRACT_VIOLATION` | `details.api_body` deep-equals the fixture |
| synthetic 405 `{error:{code:'METHOD_NOT_ALLOWED',…}}` | `UPSTREAM_CONTRACT_VIOLATION` | |
| synthetic 400 with code `LAYER_PRIVATE` | `UPSTREAM_CONTRACT_VIOLATION` | a code that does not belong with its status |
| synthetic 404, body `<html>` | `UPSTREAM_CONTRACT_VIOLATION` | no `api_body` |
| synthetic 418 JSON | `UPSTREAM_CONTRACT_VIOLATION` | `api_body` present |
| synthetic 429 `RATE_LIMITED`, `Retry-After: 30` | `UPSTREAM_UNAVAILABLE` | `details.retry_after === '30'` |
| synthetic 503 | `UPSTREAM_UNAVAILABLE` | as A0 |
| synthetic 302 | `UPSTREAM_CONTRACT_VIOLATION` | `details.location`, as A0 |

  Keep A0's timeout, network, size-cap and `User-Agent` tests. In `errors.test.ts`, the three new
  codes produce `isError` tool results and `-32603` protocol errors like the others.
- [ ] **Step 2: Run** `npx vitest run test/unit/upstream-client.test.ts test/unit/errors.test.ts` — Expected: FAIL.
- [ ] **Step 3: Implement.** The 3xx, 429 and ≥500 branches stay as A0. For any other status, parse
  the body with `errorBodySchema`; if it parses and its code is in the allowed set for the status
  (`400` → the three new codes; `403` → `LAYER_PRIVATE`; `404` → `LAYER_NOT_FOUND`), throw that code
  with v2's `message` and `{...details, request, status}`. Otherwise `UPSTREAM_CONTRACT_VIOLATION`,
  message `TLCMap answered HTTP {status}{ code, if any}, which the server did not expect.`, with
  `api_body` when the body is JSON. Update the comment above `errorForStatus`, which says the
  403/404 map is for the v1 layer endpoint.
- [ ] **Step 4: Run** — Expected: PASS; then `npm test` — Expected: PASS (no A0 integration test
  depends on 403/404).
- [ ] **Step 5: Commit** — `git commit -m "Map v2 error responses to typed errors"`.

---

### Task 5: Provenance and the page service

**Files:**
- Modify: `src/upstream/cache.ts`, `src/provenance.ts`
- Create: `src/pages/service.ts`
- Test: `test/unit/cache.test.ts`, `test/unit/provenance.test.ts`, `test/unit/page-service.test.ts`

**Interfaces:**
- Consumes: `UpstreamClient`, `Limiter`, `SingleFlight`, `ByteLruCache`, `Logger`.
- Produces:

```ts
// src/upstream/cache.ts — ByteLruCacheOptions gains:
onEvict?: (key: string, value: V) => void;   // called when an entry leaves for any reason:
                                             // LRU eviction, TTL expiry, or replacement by set()

// src/provenance.ts
export type Ids = { record_uids: string[] } | { layer_ids: number[] };
export type FetchProvenance = { request: string; fetched_at: string; sha256: string; bytes: number;
  server: string } & Ids;
export type Provenance = FetchProvenance & { cache: CacheOutcome };
export function buildProvenance(r: UpstreamResponse, ids: Ids, server: string): FetchProvenance;

// src/pages/service.ts
export interface PageEntry { url: string; body: Buffer; head: Record<string, unknown>;
  provenance: FetchProvenance; resourceUri: string | null }
export interface PageRequest {
  url: URL;
  /** Validate and check the body; return it without features, and its ids. Throws TlcmapError. */
  parse(body: Buffer): { head: Record<string, unknown>; ids: Ids };
  /** Present only for feature pages. */
  resourceUri?(sha256: string): string;
}
export interface UpstreamStats { requests_total: number; in_flight: number; queued: number;
  last_success_at: string | null; last_error: { code: ErrorCode; at: string } | null }   // moved from layers/service
export interface PageService {
  fetch(req: PageRequest): Promise<{ entry: PageEntry; cache: CacheOutcome }>;
  peekResource(uri: string): PageEntry | undefined;   // never fetches
  stats(): { upstream: UpstreamStats; cache: { entries: number; bytes: number } };
}
export function createPageService(deps: { client: UpstreamClient; limiter: Limiter;
  cache: { maxBytes: number; ttlMs: number; now?: () => number }; serverId: string; logger: Logger }): PageService;
```

  `layers/service.ts` keeps its own copy of `UpstreamStats` until Task 7 deletes it; the
  provenance change ripples into it, and into `test/unit/mcp-surface.test.ts`'s `entryFrom`, as
  `buildProvenance(res, { record_uids: summary.record_uids }, serverId)`.

- [ ] **Step 1: Write the failing tests.**
  - `cache.test.ts`: `onEvict` fires once with the key and value on LRU eviction, on TTL expiry
    (found by `get` or by `purgeExpired`), and when `set` replaces a key; never for a value too large
    to store that was never stored.
  - `provenance.test.ts`: A0's assertions, plus `record_uids` and `layer_ids` each pass through, and
    the envelope has exactly one of them.
  - `page-service.test.ts`, with the real `createUpstreamClient` over `fixtureFetch` and a
    `PageRequest` for `layer-206-features-p1.json` whose `parse` uses Task 3:
    - Miss then hit: one upstream request; `cache` `miss` then `hit`; same `fetched_at`.
    - Concurrent identical fetches: one request; outcomes `miss` and `shared`.
    - `entry.resourceUri === req.resourceUri(entry.provenance.sha256)`, and `peekResource(uri)`
      returns the entry without a request.
    - A `parse` that throws CV: rejected; nothing cached; `peekResource` undefined; the CV's details
      gain `request` and `status` and carry no body.
    - **Evicting an entry drops its resource URI:** with `cache.maxBytes` just above one body, fetching
      a second URL evicts the first; `peekResource(firstUri)` is undefined.
    - **A URI shared by two entries survives the older one's eviction:** two URLs whose stub bodies
      are byte-identical give the same URI; after the older entry is evicted, `peekResource(uri)`
      returns the newer entry.
    - TTL expiry (via `now`) drops the URI too.
    - `stats().upstream`: `requests_total` counts client calls only; a `LAYER_NOT_FOUND` from
      `layer-99999999.json` sets `last_success_at` and leaves `last_error` null; an
      `UPSTREAM_UNAVAILABLE` (synthetic 503) sets `last_error` (spec §9.3).
    - One `upstream` log event per client call, with A0's fields.
- [ ] **Step 2: Run** the three files — Expected: FAIL.
- [ ] **Step 3: Implement.** The cache key is `req.url.href`; `sizeOf` is `entry.body.length`. The
  order is cache `get`, else `SingleFlight.do(key, () => limiter.run(() => client.get(url)))`, then
  `parse`, then `buildProvenance`, then the URI from `sha256`, then `cache.set`. The URI index is a
  `Map<uri, cacheKey>`, written on `set` and cleared in `onEvict` only when it still points at the
  evicted key. `last_success_at` is set on a 2xx and on the five v2 error codes (`INVALID_PARAMETER`,
  `INVALID_GEOMETRY`, `FILTER_REJECTED`, `LAYER_PRIVATE`, `LAYER_NOT_FOUND`); `last_error` is set only
  for `UPSTREAM_UNAVAILABLE`, `UPSTREAM_CONTRACT_VIOLATION` and `RESPONSE_TOO_LARGE`.
- [ ] **Step 4: Run** — Expected: PASS; then `npm test` — Expected: PASS.
- [ ] **Step 5: Commit** — `git commit -m "Add the page service and per-page provenance"`.

---

### Task 6: Tool scaffolding, `tlcmap_get_layer_metadata` and the page resources

**Files:**
- Create: `src/tools/common.ts`, `src/tools/get-layer-metadata.ts`, `src/resources/pages.ts`
- Test: `test/unit/tools-layer.test.ts`, `test/unit/resources-pages.test.ts`

**Interfaces:**
- Consumes: `PageService` (Task 5), `layerUrl` (Task 1), `layerDetailResponse`, `checkLayerIdentity`
  (Task 3).
- Produces:

```ts
// src/tools/common.ts
export const provenanceSchema;   // zod for Provenance, with record_uids | layer_ids as a union
export const layerIdSchema;      // z.number().int().min(1).max(2147483647)
export function runTool(name: string, args: Record<string, unknown>, logger: Logger,
  body: () => Promise<{ structured: Record<string, unknown>; cache: CacheOutcome }>): Promise<CallToolResult>;
  // success → {structuredContent: structured, content: [{type: 'text', text: JSON.stringify(structured)}]};
  // TlcmapError → toToolErrorResult; anything else rethrown; one `tool` log event {tool, arguments,
  // outcome: 'ok' | code, cache?, duration_ms}
export interface ToolDeps { pages: PageService; baseUrl: URL; logger: Logger }

// src/tools/get-layer-metadata.ts
export const GET_LAYER_METADATA_TOOL = 'tlcmap_get_layer_metadata';
export function getLayerMetadataTool(deps: ToolDeps): Registration;

// src/resources/pages.ts
export const LAYER_PAGE_TEMPLATE = 'tlcmap://layer/{id}/features/{sha256}';
export const SEARCH_PAGE_TEMPLATE = 'tlcmap://search/{sha256}';
export function layerPageUri(layerId: number, sha256: string): string;
export function searchPageUri(sha256: string): string;
export function parseLayerId(raw: string): number;   // A0's rule; INVALID_ARGUMENT otherwise
export function parseSha256(raw: string): string;    // /^[0-9a-f]{64}$/; INVALID_ARGUMENT otherwise
export function pageResources(deps: { pages: PageService; logger: Logger }): Registration[];  // both templates
```

  The tool names `tlcmap_get_layer` and `tlcmap_search_places` are used in `RESOURCE_EXPIRED`
  details as string literals here, so `resources/pages.ts` imports no tool module.

- [ ] **Step 1: Write the failing tests.** Both files build an `McpServer`, `registerEnabled` the
  registrations under test with `['read']`, and connect a `Client` over
  `InMemoryTransport.createLinkedPair()`. The page service is real, over `fixtureFetch`, with base
  `http://tlcmap-ghap.laragon.local`.
  - `tools-layer.test.ts`:
    - `tlcmap_get_layer_metadata` on 206: `structuredContent.layer` deep-equals the fixture's
      `layer`, `summary` its `summary`; `layer.warning === 'None'`, `layer.license === null`;
      `provenance` is `{request: 'GET http://tlcmap-ghap.laragon.local/api/v2/layers/206',
      cache: 'miss', layer_ids: [206], sha256: <sha of fixture>, bytes, fetched_at, server: SERVER_ID}`;
      no `record_uids` key; `content` is `[{type: 'text', text: JSON.stringify(structuredContent)}]`.
    - On 258: `layer.license === 'Do not use without permission.'` and `layer.warning` equals the
      fixture's string. On 472: `layer.warning` byte-identical to the fixture's.
    - 33 → `isError`, code `LAYER_PRIVATE`, `details.layer_id` 33; 99999999 → `LAYER_NOT_FOUND`.
    - A stub body for 206 whose `layer.id` is 207 → `UPSTREAM_CONTRACT_VIOLATION`.
    - `layer_id: 'abc'`, `0` and `2147483648` → the SDK's `Input validation error:`, no upstream
      request; an extra argument `{layer_id: 206, sort: 'id'}` is rejected too.
    - One `tool` log line per call with `outcome`.
  - `resources-pages.test.ts`: a page put into the service with
    `pages.fetch({url, parse, resourceUri: sha => layerPageUri(206, sha)})` for
    `layer-206-features-p1.json`:
    - Reading its URI returns `text` equal to the fixture's bytes as UTF-8, `mimeType`
      `application/json`, and `_meta.provenance` with `cache: 'hit'` and the same `sha256`.
    - `resources/templates/list` holds exactly the two templates; `resources/list` is empty.
    - A well-formed URI not held → protocol error `RESOURCE_EXPIRED` with
      `details.tool === 'tlcmap_get_layer'`; the same for a search URI, naming `tlcmap_search_places`.
    - The held hash under another layer id (`tlcmap://layer/207/features/<sha>`) → `RESOURCE_EXPIRED`.
    - `test.each` over `007`, `0`, `-1`, `abc`, `2147483648` as `{id}`, and an uppercase hash, a
      63-character hash and `xyz` as `{sha256}` → `INVALID_ARGUMENT`, no `peekResource` lookup.
- [ ] **Step 2: Run** both files — Expected: FAIL.
- [ ] **Step 3: Implement.** `getLayerMetadataTool` registers with
  `inputSchema: z.strictObject({layer_id: layerIdSchema})`, an `outputSchema` of
  `{layer: layerSchema, summary: summarySchema, provenance: provenanceSchema}`, and this description:

  > Read one TLCMap layer's metadata — name, creator, citation, licence, rights and warning — and the
  > summary of its records (count, extent, date range, undated and no-coordinates counts), without
  > fetching any records. Use it to check a layer's rights before using its data. Licence, rights and
  > warning text are returned verbatim and must be shown to the user, not paraphrased. The summary's
  > undated and no-coordinates counts are what lets a result say which records it could not place.

  Its `parse` is `parseV2(body, layerDetailResponse)`, then `checkLayerIdentity`, `ids` `{layer_ids:
  [layer_id]}`, no `resourceUri`. A resource read matches the URI to an entry with
  `pages.peekResource(uri.href)` — the held entry's own URI, so a different `{id}` cannot match.
- [ ] **Step 4: Run** — Expected: PASS; `npm test`, typecheck and lint clean.
- [ ] **Step 5: Commit** — `git commit -m "Add tlcmap_get_layer_metadata and the page resources"`.

---

### Task 7: `tlcmap_get_layer` on v2, the wiring, and v1 removed

**Files:**
- Rewrite: `src/tools/get-layer.ts`
- Modify: `src/mcp/server.ts`, `src/wire.ts`, `src/http/app.ts` (`UpstreamStats` import), `package.json`
  (`version` `0.2.0`), `package-lock.json`, `test/fake-tlcmap.ts` (v1 routes removed),
  `test/helpers/client.ts`, `test/integration/server.test.ts`
- Delete: `src/layers/service.ts`, `src/summarise/layer.ts`, `src/resources/layer-features.ts`,
  `test/unit/layer-service.test.ts`, `test/unit/summarise-layer.test.ts`, `test/unit/mcp-surface.test.ts`,
  `test/prod/compat.test.ts` (it imports the v1 service; Task 9 replaces it), `test/fixtures/*.json`,
  `test/fixtures/synthetic/`, `test/fixtures/CAPTURED.md`
- Test: `test/unit/tools-layer.test.ts` (extended), `test/integration/server.test.ts`

**Interfaces:**
- Consumes: Tasks 1, 3, 5, 6.
- Produces:

```ts
// src/tools/get-layer.ts
export const GET_LAYER_TOOL = 'tlcmap_get_layer';
export function getLayerTool(deps: ToolDeps): Registration;
// src/mcp/server.ts
export function createServerFactory(deps: ToolDeps & { scopes: Scope[] }): McpServerFactory;
// src/wire.ts — buildServer(config, logger) returns { app, pages: PageService, close() }
```

- [ ] **Step 1: Write the failing tests.**
  - `tools-layer.test.ts` gains `tlcmap_get_layer`:
    - On 206 with no options: `structuredContent` has exactly the keys `layer, query, summary, page,
      resource_uri, provenance`; the first four deep-equal the fixture's; `summary.total` 52;
      `resource_uri === layerPageUri(206, provenance.sha256)`; `provenance.record_uids` is the 52
      feature ids in order; request `GET …/api/v2/layers/206/features`.
    - `{layer_id: 206, per_page: 30}` gives page 1 (30 ids, `page.next_cursor` a string);
      `{layer_id: 206, per_page: 30, cursor: <that>}` gives page 2 (22 ids, `next_cursor` null); the
      union is 52 distinct ids, equal to `summary.total` on both pages.
    - **An empty layer returns its page and an empty record_uids:** 257 → not `isError`,
      `summary.total` 0, `provenance.record_uids` `[]`, and reading `resource_uri` succeeds.
    - `per_page: 6000` → `INVALID_PARAMETER`, `details.parameter === 'per_page'`, v2's message.
    - A stub page for 206 containing a feature with `layer_id: 258` → `UPSTREAM_CONTRACT_VIOLATION`.
    - Reading the returned `resource_uri` gives bytes whose sha256 equals `provenance.sha256`.
    - Input schema: `sort` enum `order|id|title|start|end`, `direction` `asc|desc`, `nulls`
      `first|last`, `limit` and `per_page` integers (no range), `cursor` string; unknown keys rejected.
  - `test/integration/server.test.ts`, rewritten against the v2 fake:
    - `tools/list` names exactly `tlcmap_get_layer` and `tlcmap_get_layer_metadata` (Task 8 makes it
      four); both protocol eras, as A0.
    - Cursor following, the resource `sha256` match, and a second call as a cache hit with the
      original `fetched_at` and no new fake request, end to end.
    - A0's upstream-failure tests (redirect, 503, timeout, size cap, `SERVER_BUSY`) with their
      routes moved to `/api/v2/layers/206/features`. The size cap is `RESPONSE_TOO_LARGE`, never a
      partial page.
    - 403, 404 and `INVALID_PARAMETER` end to end; `RESOURCE_EXPIRED` after a fresh `startTestServer`.
    - A0's HTTP-surface, logs, bad-body, proxy and entry-point tests, with tool calls updated.
    - `/healthz` `version` is `0.2.0`.
- [ ] **Step 2: Run** `npm test` — Expected: FAIL.
- [ ] **Step 3: Implement.** `getLayerTool` registers `z.strictObject({layer_id, sort, direction,
  nulls, limit, cursor, per_page})` (all but `layer_id` optional), an `outputSchema` of
  `{layer, query, summary, page, resource_uri: z.string(), provenance}`, and this description:

  > Read one page of a TLCMap layer's records, with the layer's metadata and the summary of the whole
  > layer. The records are not inline: read them from resource_uri. Follow page.next_cursor to read
  > further pages; the summary describes the whole layer on every page, not the page. Its undated and
  > no-coordinates counts are what lets a result say which records it could not place. Licence, rights
  > and warning text are returned verbatim and must be shown to the user, not paraphrased. For a
  > layer's records within a region, use tlcmap_search_places with layers and bbox.

  Its `parse`: `parseV2(body, layerFeaturesResponse)`, `checkLayerIdentity(doc.layer, layer_id)`,
  `checkFeaturesOfLayer(doc.features, layer_id)`, head `withoutFeatures(doc)`, ids the features'
  `properties.id`; `resourceUri: sha => layerPageUri(layer_id, sha)`. The factory registers
  `[getLayerTool, getLayerMetadataTool, ...pageResources]`. `wire.ts` builds `createPageService` with
  `config.cache` and passes `baseUrl: config.tlcmapBaseUrl`. Delete the v1 files listed above and
  their imports; `npm install` to refresh the lock file's version.
- [ ] **Step 4: Run** `npm test && npm run typecheck && npm run lint && npm run build` — Expected: all
  pass; `git ls-files src test scripts | grep -E "layers/service|summarise|layer-features|test/prod|fixtures/[^v]"`
  — Expected: no output.
- [ ] **Step 5: Commit** — `git commit -m "Move tlcmap_get_layer to v2 and remove the v1 upstream"`.

---

### Task 8: `tlcmap_list_layers` and `tlcmap_search_places`

**Files:**
- Create: `src/tools/list-layers.ts`, `src/tools/search-places.ts`
- Modify: `src/mcp/server.ts`, `src/tools/common.ts` (shared input schemas)
- Test: `test/unit/tools-search.test.ts`, `test/integration/server.test.ts`

**Interfaces:**
- Consumes: Tasks 1, 3, 5, 6.
- Produces:

```ts
// src/tools/common.ts adds
export const bboxSchema;      // z.tuple([z.number(), z.number(), z.number(), z.number()])
export const polygonSchema;   // z.array(z.tuple([z.number(), z.number()])).min(3)
// src/tools/list-layers.ts
export const LIST_LAYERS_TOOL = 'tlcmap_list_layers';
export function listLayersTool(deps: ToolDeps): Registration;
// src/tools/search-places.ts
export const SEARCH_PLACES_TOOL = 'tlcmap_search_places';
export function searchPlacesTool(deps: ToolDeps): Registration;
```

- [ ] **Step 1: Write the failing tests** `test/unit/tools-search.test.ts` (same harness as Task 6):
  - `tlcmap_list_layers` with the Hunter Valley `bbox`: `query`, `total`, `page`, `layers` deep-equal
    the fixture's; `total` 8; `provenance.layer_ids` `[84, 102, 140, 170, 206, 379, 461, 462]`; no
    `resource_uri` key; request `GET …/api/v2/layers?bbox=150.8,-33.1,151.4,-32.6`.
  - `tlcmap_search_places` with the box and `per_page: 20`: `query`, `summary` (with `by_layer`),
    `page` deep-equal the fixture's; `summary.total` 1318; `record_uids` the 20 ids;
    `resource_uri === searchPageUri(sha256)`, and reading it returns the fixture bytes.
  - With `summary_only: true`: `summary` deep-equals the summary fixture's; `page === null`,
    `resource_uri === null`, `record_uids` `[]`.
  - `extended_data: 'Years > many'` → `FILTER_REJECTED` with v2's `expression`, `position`,
    `expected`. The self-intersecting polygon → `INVALID_GEOMETRY` with `reason`. `layers: [33]` →
    `LAYER_PRIVATE`, `details.layer_id` 33.
  - **A cursor from another query is INVALID_PARAMETER naming cursor:** the `error-cursor.json`
    request → `details.parameter === 'cursor'`.
  - Input schemas: `bbox` of three numbers, a polygon of two vertices, `sources: ['gaz']`,
    `match: 'like'`, `include_undated: 'false'` (a string) → SDK input validation error, no request.
    `bbox` with `minLon > maxLon` is accepted (the antimeridian, API §7.5).
  - Integration: `tools/list` names exactly the four tools; `resources/templates/list` exactly the two
    templates.
- [ ] **Step 2: Run** `npx vitest run test/unit/tools-search.test.ts` — Expected: FAIL.
- [ ] **Step 3: Implement.** Inputs are spec §5.1 and §5.2's tables, all optional. Descriptions:

  > *tlcmap_list_layers:* Search TLCMap's catalogue of public layers. bbox, polygon and the date
  > filters match a layer when at least one of its records matches — never by a declared extent — and
  > matching_count gives how many do. has_license and has_warning report whether the field is present,
  > not what it says; read the text itself with tlcmap_get_layer_metadata. total counts every matching
  > layer; follow page.next_cursor for further pages.

  > *tlcmap_search_places:* Search TLCMap's places across contributed layers and the ANPS, NCG and
  > text gazetteers. The records are not inline: read them from resource_uri; follow page.next_cursor
  > for further pages. The summary describes the whole match set, not the page; its undated and
  > no-coordinates counts are what lets a result say which records it could not place, and by_layer
  > names every contributing layer with its warning, verbatim, which must be shown to the user. Read
  > each layer's licence and rights with tlcmap_get_layer_metadata. summary_only returns the summary
  > without records. include_undated defaults to true, so a date filter keeps undated records unless
  > it is false.

  `searchPlacesTool`'s `parse` picks `placesSummaryOnlyResponse` when `args.summary_only === true`,
  else `placesResponse`; only the latter has a `resourceUri` (`searchPageUri`). The output schema has
  `page` and `resource_uri` nullable. The factory registers the four tools in the order
  list, search, get layer, get metadata, then the resources.
- [ ] **Step 4: Run** `npm test && npm run typecheck && npm run lint` — Expected: PASS.
- [ ] **Step 5: Commit** — `git commit -m "Add tlcmap_list_layers and tlcmap_search_places"`.

---

### Task 9: The compatibility check

**Files:**
- Create: `test/compat/compat.test.ts`, `vitest.compat.config.ts`
- Delete: `vitest.prod.config.ts` (`test/prod/compat.test.ts` went in Task 7)
- Modify: `package.json` (`test:prod` → `test:compat`: `vitest run --config vitest.compat.config.ts`),
  `eslint.config.js` and `tsconfig.json` if they name `test/prod`

**Interfaces:**
- Consumes: `createUpstreamClient`, Task 1's builders, Task 3's schemas and checks.

- [ ] **Step 1: Write** `test/compat/compat.test.ts`, reading `TLCMAP_BASE_URL` (default
  `https://tlcmap.org`), four tests, one request each, sequential (spec §11.4):
  - `layerUrl(206)` parses with `layerDetailResponse`; `layer.id` 206; `layer.warning === 'None'`.
  - `layerFeaturesUrl(206, {per_page: 5})` parses with `layerFeaturesResponse`; `checkLayerIdentity`
    and `checkFeaturesOfLayer` pass; every `properties.id` is a non-empty string.
  - `layerUrl(99999999)` rejects with `LAYER_NOT_FOUND`.
  - `layerFeaturesUrl(206, {per_page: 0})` rejects with `INVALID_PARAMETER`, `details.parameter`
    `per_page`.
- [ ] **Step 2: Run against local v2** —
  `TLCMAP_BASE_URL=http://tlcmap-ghap.laragon.local npm run test:compat` — Expected: 4 passed.
  Do not run it against production or staging: neither serves v2 yet (spec §2).
- [ ] **Step 3: Confirm CI does not run it** — `npm test` lists no `test/compat` file.
- [ ] **Step 4: Commit** — `git commit -m "Make the compatibility check run against any environment"`.

---

### Task 10: Documentation and deployment notes

**Files:**
- Modify: `README.md`, `deploy/env.example`, `deploy/DEPLOY.md`

- [ ] **Step 1: README.** It now describes a server that reads TLCMap's API v2 and offers the four
  tools and two resource templates; links the A1 MCP spec beside A0's; replaces "Staging" with the
  three environments of spec §10 (staging still serves 0.1.0 until v2 reaches test-ghap); adds
  "Running locally" with the `npm start` line and the `claude mcp add --transport http tlcmap-local
  http://localhost:3100/mcp` line, and the note to raise `API_V2_RATE_LIMIT` in the application's local
  `.env` and run `php artisan config:clear`; updates the scripts table (`test:compat`, and
  `capture-fixtures` requiring `TLCMAP_BASE_URL`).
- [ ] **Step 2: `deploy/env.example`.** `TLCMAP_BASE_URL` gains a comment: an application serving
  `/api/v2`; staging uses `https://test-ghap.tlcmap.org`. The value stays `https://tlcmap.org` (the
  config default).
- [ ] **Step 3: `deploy/DEPLOY.md`.** The opening line says staging reads test-ghap from 0.2.0 on. A
  new section "Before deploying 0.2.0" lists spec §10's prerequisites: v2 on test-ghap,
  `API_V2_RATE_LIMIT` set there for the server's load, `TrustProxies` configured, then
  `TLCMAP_BASE_URL=https://test-ghap.tlcmap.org` in `/etc/tlcmap-mcp/env` and `npm run test:compat`
  against test-ghap before `deploy.sh`.
- [ ] **Step 4: Run** `npm test` — Expected: PASS (`test/deploy/env-example.test.ts` still finds every
  variable).
- [ ] **Step 5: Commit** — `git commit -m "Document v2, the local stack and the staging prerequisites"`.

---

### Task 11: Local acceptance, the record, and the merge

**Files:**
- In `tlcmap-skills`: create `documents/acceptance/A1-MCP.md`
- In `tlcmap-skills`: modify `README.md` (status)

- [ ] **Step 1: Prepare the stack.** In `C:\laragon\www\tlcmap-ghap\.env` (git-ignored) set
  `API_V2_RATE_LIMIT=600,1`, then `php artisan config:clear`. In `tlcmap-mcp`: `npm ci && npm run
  build`, then from Git Bash `PUBLIC_URL=http://localhost:3100
  TLCMAP_BASE_URL=http://tlcmap-ghap.laragon.local npm start`. Expected: a `startup` log line with
  version `0.2.0`.
- [ ] **Step 2: Run spec §12.1's checks 1 to 8** in a scratch directory outside both repositories,
  with `claude mcp add --transport http tlcmap-local http://localhost:3100/mcp` (local scope, removed
  afterwards). For each, capture the tool output and the matching `curl` of the v2 request, and the
  server's `tool` and `upstream` log lines. Check 1's CI is the pull request's (Step 4); run
  `test:compat` against local v2 here. Check 8 restarts the server process.
- [ ] **Step 3: Write `tlcmap-skills/documents/acceptance/A1-MCP.md`** in A0's record format: commit, served version, Node,
  the upstream (`http://tlcmap-ghap.laragon.local`, `tlcmap-ghap` commit, database), the roles table
  of spec §12 with the local layers, and three parts — **Local** (each check with its evidence and
  PASS or FAIL), **Staging** and **Production**, each marked pending with spec §10's prerequisites
  and §12.2/§12.3's steps. Any failure is recorded as a failure and stops the merge.
- [ ] **Step 4: Push and open the pull request.**
  `git push -u origin a1-mcp && gh pr create --base main --title "A1: move to API v2 and add the discovery and rights tools"`,
  body linking the spec and the record. Expected: CI green. Record the run in check 1 and commit.
- [ ] **Step 5: Merge** once the user approves the pull request.
- [ ] **Step 6: Update `tlcmap-skills`.** The README status says the MCP half of A1 is accepted on the
  local stack (with the date) and not deployed, linking `documents/acceptance/A1-MCP.md`. Commit in
  `tlcmap-skills`: `git commit -m "Record the local acceptance of the A1 MCP half"`.
