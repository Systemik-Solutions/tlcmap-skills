# A0 — MCP server skeleton: implementation plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development
> (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use
> checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build and deploy to `test-mcp.tlcmap.org` a stateless MCP server with one read tool
(`tlcmap_get_layer`), one resource template, provenance, typed errors, scopes and courtesy controls,
calling production tlcmap.org.

**Architecture:** One Node 24 process. Express receives `POST /mcp` and hands it to the SDK v2
fetch-shaped handler (`createMcpHandler`, JSON response mode) through `toNodeHandler`; a fresh
`McpServer` is built per request from a factory that closes over long-lived services. Every request
to tlcmap.org goes through one upstream client, behind a global limiter, single-flight and a
byte-capped cache; tools and resources see only the layer service.

**Tech Stack:** TypeScript `~6.0.3` (ES modules, strict), Node 24, `@modelcontextprotocol/server`
`^2.4.0`, `@modelcontextprotocol/node` `^2.1.2`, `@modelcontextprotocol/express` `^2.0.2`, Express
`^5.3.0`, zod `^4.6.7`, `express-rate-limit` `^8.7.3`; Vitest `^5.1.0` with
`@modelcontextprotocol/client` `^2.4.0`; ESLint `^10` with `typescript-eslint` `^8.72`; `tsx` for
scripts.

**Spec:** [A0-SERVER-SKELETON.md](../specs/A0-SERVER-SKELETON.md). Read it first; this plan argues from
it and cites it as "spec §n".

**Where the work happens:** the `tlcmap-mcp` repository (`https://github.com/Systemik-Solutions/tlcmap-mcp.git`,
local clone `C:\AIProject\tlcmap-mcp`, currently a one-line README). All paths below are relative
to that repository. Task 12 runs on the staging server.

## Global Constraints

- Node `>=24 <25` in `engines`; `.nvmrc` contains `24`. CI runs Node 24.
- TypeScript pinned `~6.0.3`: `typescript-eslint` 8.72 declares `typescript >=4.8.4 <6.1.0`.
- `tsconfig.json` has `"types": ["node"]` — TypeScript ≥ 6 no longer auto-includes `@types/*`, and
  the SDK's declarations reference `Buffer`.
- ES modules (`"type": "module"`), `module`/`moduleResolution` `NodeNext`, relative imports end in
  `.js`.
- LF line endings (`.gitattributes`: `* text=auto eol=lf`), UTF-8 without BOM, one trailing newline.
- Australian English in documentation, log messages and error messages (`licence` as a noun in
  prose; the API field stays `license`).
- Never the name "GHAP" in new text; it is "TLCMap".
- No unit or integration test touches the network; only `npm run test:prod` does.
- `license`, `rights`, `warning` and every other `metadata` value pass through unchanged — never
  trimmed, decoded, defaulted or interpreted (spec §4).
- Upstream redirects are never followed (`redirect: 'manual'`).
- No model calls, no write tools, no content-type sniffing, no workaround for a §5 API item (spec
  §1 "Out of scope").
- Server identity string: `tlcmap-mcp {version}`, with `version` read from `package.json`.

## Review Focus

The inputs the spec implies but does not spell out, most likely to bite first. Each has a test in
the task named.

1. **Non-ASCII feed content** — Indigenous placenames with diacritics, typographic quotes, HTML
   entities. The resource's UTF-8 text must re-encode to exactly the bytes whose `sha256` the tool
   reported. *Task 8, `resource text round-trips multi-byte UTF-8 to the reported sha256`.*
2. **A feed bigger than the cache.** If `CACHE_MAX_BYTES` < `UPSTREAM_MAX_BYTES`, a body could be
   accepted but never cached, so its resource handle would be expired on arrival. Config must
   refuse that combination. *Task 2, `rejects CACHE_MAX_BYTES smaller than UPSTREAM_MAX_BYTES`.*
3. **A shared fetch that fails.** Every single-flight joiner gets the same typed error, nothing is
   cached, and the next call fetches again. *Task 7, `a failed shared fetch rejects every joiner
   and is not cached`.*
4. **Odd layer IDs in resource URIs** — `007`, `0`, `-1`, `1.5`, `abc`, `99999999999`. Only
   canonical decimal integers 1–2147483647 are valid; a URI is never normalised into another
   layer's handle. *Task 8, `resource rejects non-canonical layer ids`.*
5. **Malformed features inside an otherwise valid feed** — `properties: null`, a geometry without
   `coordinates`, empty coordinate arrays, `NaN`-like strings in coordinates. Each is
   `UPSTREAM_CONTRACT_VIOLATION` or a correct count, never a crash or a silent `0`. *Task 3,
   `malformed features`.*

---

## File structure

```
package.json  package-lock.json  tsconfig.json  tsconfig.build.json
eslint.config.js  vitest.config.ts  vitest.prod.config.ts
.gitignore  .gitattributes  .nvmrc  README.md
.github/workflows/ci.yml
src/
  index.ts                     wiring, listen, graceful shutdown                    Task 9
  version.ts                   name + version from package.json                    Task 1
  config.ts                    env → Config; CONFIG_VARIABLES                       Task 2
  log.ts                       JSON-line logger                                     Task 2
  errors.ts                    TlcmapError, ErrorCode, wire shapes                  Task 3
  summarise/layer.ts           feed bytes → LayerSummary (pure)                     Task 3
  upstream/cache.ts            ByteLruCache                                         Task 4
  upstream/limiter.ts          Limiter, SingleFlight                                Task 5
  upstream/client.ts           the only fetch to tlcmap.org                         Task 6
  provenance.ts                Provenance, buildProvenance                          Task 7
  layers/service.ts            LayerService: cache + limiter + client + summary     Task 7
  mcp/scopes.ts                Registration, registerEnabled                        Task 8
  tools/get-layer.ts           tlcmap_get_layer                                     Task 8
  resources/layer-features.ts  tlcmap://layer/{id}/features                         Task 8
  mcp/server.ts                createServerFactory                                  Task 8
  http/app.ts                  Express app: /mcp, /healthz, rate limit, logging     Task 9
test/
  fixtures/                    captured + synthetic feeds, CAPTURED.md              Tasks 3, 6
  fake-tlcmap.ts               local HTTP server standing in for tlcmap.org         Task 6
  helpers/client.ts            connect an MCP client to a running app               Task 9
  unit/…  integration/…        per task
  prod/compat.test.ts          production compatibility check                       Task 10
  deploy/env-example.test.ts   env.example ↔ config                                 Task 11
scripts/capture-fixtures.ts                                                         Task 6
deploy/  tlcmap-mcp.service  test-mcp.tlcmap.org.conf  env.example  deploy.sh  DEPLOY.md   Task 11
docs/acceptance/A0.md                                                               Task 12
```

---

### Task 1: Scaffold the repository

**Files:**
- Create: `package.json`, `tsconfig.json`, `tsconfig.build.json`, `eslint.config.js`,
  `vitest.config.ts`, `.gitignore`, `.gitattributes`, `.nvmrc`, `.github/workflows/ci.yml`,
  `src/version.ts`
- Modify: `README.md`
- Test: `test/unit/version.test.ts`

**Interfaces:**
- Produces: `export const SERVER_NAME: string`, `export const SERVER_VERSION: string`,
  `export const SERVER_ID: string` (`"tlcmap-mcp 0.1.0"`) from `src/version.ts`.

- [ ] **Step 1: Write `package.json`** — `"name": "tlcmap-mcp"`, `"version": "0.1.0"`,
  `"private": true`, `"type": "module"`, `"engines": {"node": ">=24 <25"}`, the dependency ranges in
  the Tech Stack line (runtime: server, node, express adapter, express, zod, express-rate-limit;
  dev: typescript, @types/node `^24`, @types/express `^5`, vitest, @modelcontextprotocol/client,
  eslint, @eslint/js, typescript-eslint, tsx). Scripts:

  | Script | Command |
  | --- | --- |
  | `build` | `tsc -p tsconfig.build.json` |
  | `typecheck` | `tsc --noEmit` |
  | `lint` | `eslint .` |
  | `test` | `vitest run` |
  | `test:prod` | `vitest run --config vitest.prod.config.ts` |
  | `capture-fixtures` | `tsx scripts/capture-fixtures.ts` |
  | `start` | `node dist/index.js` |

- [ ] **Step 2: Write configs.** `tsconfig.json`: `target` `ES2024`, `module`/`moduleResolution`
  `NodeNext`, `strict`, `noUncheckedIndexedAccess`, `types: ["node"]`, `resolveJsonModule`,
  includes `src`, `test`, `scripts`. `tsconfig.build.json` extends it with `rootDir: src`,
  `outDir: dist`, includes `src` only. `vitest.config.ts`: `include: ['test/unit/**', 'test/integration/**', 'test/deploy/**']`.
  `eslint.config.js`: `@eslint/js` recommended + `typescript-eslint` recommended, ignoring `dist`.
  `.gitignore`: `node_modules`, `dist`. `.gitattributes`: `* text=auto eol=lf`. `.nvmrc`: `24`.

- [ ] **Step 3: Write the failing test** `test/unit/version.test.ts`:

```ts
import pkg from '../../package.json' with { type: 'json' };
import { SERVER_ID, SERVER_NAME, SERVER_VERSION } from '../../src/version.js';

test('server identity comes from package.json', () => {
  expect(SERVER_NAME).toBe('tlcmap-mcp');
  expect(SERVER_VERSION).toBe(pkg.version);
  expect(SERVER_ID).toBe(`tlcmap-mcp ${pkg.version}`);
});
```

- [ ] **Step 4: Run** `npm install && npx vitest run test/unit/version.test.ts` — Expected: FAIL,
  cannot resolve `src/version.js`.
- [ ] **Step 5: Implement `src/version.ts`** with a JSON import of `../package.json`.
- [ ] **Step 6: Run** `npm run typecheck && npm run lint && npm test` — Expected: all PASS.
- [ ] **Step 7: Write `.github/workflows/ci.yml`** — on push and pull request; `actions/setup-node`
  with `node-version-file: .nvmrc`, `cache: npm`; `npm ci`, `npm run typecheck`, `npm run lint`,
  `npm test`, `npm run build`.
- [ ] **Step 8: Rewrite `README.md`** — one paragraph on what the server is, a link to the spec in
  `tlcmap-skills` (`documents/specs/A0-SERVER-SKELETON.md`), the scripts table, and "Deployment:
  see `deploy/DEPLOY.md`".
- [ ] **Step 9: Commit** — `git add -A && git commit -m "Scaffold the TypeScript project"`, push,
  and confirm the CI run is green.

---

### Task 2: Configuration and logger

**Files:**
- Create: `src/config.ts`, `src/log.ts`
- Test: `test/unit/config.test.ts`, `test/unit/log.test.ts`

**Interfaces:**
- Produces, from `src/config.ts`:

```ts
export type Scope = 'read' | 'write';
export type LogLevel = 'debug' | 'info' | 'warn' | 'error';
export interface Config {
  host: string; port: number; publicUrl: URL; tlcmapBaseUrl: URL;
  scopes: Scope[]; logLevel: LogLevel; rateLimitPerMin: number;
  upstream: { concurrency: number; queueMax: number; queueTimeoutMs: number;
              timeoutMs: number; maxBytes: number };
  cache: { maxBytes: number; ttlMs: number };
}
export const CONFIG_VARIABLES: readonly string[];          // every variable name in spec §9
export class ConfigError extends Error { readonly variable: string }
export function loadConfig(env: Record<string, string | undefined>): Config;
```

- Produces, from `src/log.ts`:

```ts
export interface Logger { log(level: LogLevel, event: string, fields?: Record<string, unknown>): void }
export function createLogger(level: LogLevel, write?: (line: string) => void, now?: () => Date): Logger;
```

- [ ] **Step 1: Write the failing tests** `test/unit/config.test.ts`. A `base` env holds only
  `PUBLIC_URL=https://test-mcp.tlcmap.org`.

```ts
test('defaults match spec §9', () => {
  const c = loadConfig(base);
  expect(c).toMatchObject({ host: '127.0.0.1', port: 3100, scopes: ['read'], logLevel: 'info',
    rateLimitPerMin: 60,
    upstream: { concurrency: 4, queueMax: 20, queueTimeoutMs: 20000, timeoutMs: 60000, maxBytes: 67108864 },
    cache: { maxBytes: 1073741824, ttlMs: 3600000 } });
  expect(c.tlcmapBaseUrl.href).toBe('https://tlcmap.org/');
});
test('PUBLIC_URL is required', () => expectConfigError({}, 'PUBLIC_URL'));
test.each([['PORT', '0'], ['PORT', '70000'], ['PORT', '3.5'], ['UPSTREAM_CONCURRENCY', '0'],
  ['CACHE_TTL_S', '-1'], ['TLCMAP_BASE_URL', 'not a url'], ['LOG_LEVEL', 'loud'],
  ['TLCMAP_MCP_SCOPES', 'read,admin']])('rejects %s=%s', (k, v) => expectConfigError({ ...base, [k]: v }, k));
test('write scope refuses to start', () =>
  expect(() => loadConfig({ ...base, TLCMAP_MCP_SCOPES: 'read,write' }))
    .toThrow(/write tools require Phase E/));
test('rejects CACHE_MAX_BYTES smaller than UPSTREAM_MAX_BYTES', () =>
  expectConfigError({ ...base, CACHE_MAX_BYTES: '1000', UPSTREAM_MAX_BYTES: '2000' }, 'CACHE_MAX_BYTES'));
test('CONFIG_VARIABLES lists every variable', () =>
  expect([...CONFIG_VARIABLES].sort()).toEqual(['CACHE_MAX_BYTES', 'CACHE_TTL_S', 'HOST', 'LOG_LEVEL',
    'PORT', 'PUBLIC_URL', 'RATE_LIMIT_PER_MIN', 'TLCMAP_BASE_URL', 'TLCMAP_MCP_SCOPES',
    'UPSTREAM_CONCURRENCY', 'UPSTREAM_MAX_BYTES', 'UPSTREAM_QUEUE_MAX',
    'UPSTREAM_QUEUE_TIMEOUT_MS', 'UPSTREAM_TIMEOUT_MS']));
```

  `expectConfigError(env, variable)` asserts a `ConfigError` whose `.variable` is `variable`.
  `test/unit/log.test.ts`: a line is one JSON object `{ts, level, event, ...fields}` with `ts` from
  `now()` as ISO 8601; lines below the configured level are not written; a field value that cannot
  be serialised (a `BigInt`, a cycle) is written as a string rather than throwing.

- [ ] **Step 2: Run** `npx vitest run test/unit/config.test.ts test/unit/log.test.ts` — Expected:
  FAIL, modules not found.
- [ ] **Step 3: Implement `loadConfig`** with a zod schema per variable (integers via
  `z.coerce.number().int()` with the spec's ranges; `PORT` 1–65535; every count and size ≥ 1;
  `CACHE_TTL_S` ≥ 1). `TLCMAP_MCP_SCOPES` is a comma list. `write` throws `ConfigError('TLCMAP_MCP_SCOPES',
  'write tools require Phase E token pass-through; enable read only')`. The cross-field check
  names `CACHE_MAX_BYTES`. `ConfigError.message` begins with the variable name.
- [ ] **Step 4: Implement `createLogger`** — `write` defaults to `process.stdout.write` with a
  trailing `\n`.
- [ ] **Step 5: Run** the two test files — Expected: PASS.
- [ ] **Step 6: Commit** — `git commit -m "Add configuration and JSON logger"`.

---

### Task 3: Typed errors and the feed summary

**Files:**
- Create: `src/errors.ts`, `src/summarise/layer.ts`
- Test: `test/unit/errors.test.ts`, `test/unit/summarise-layer.test.ts`,
  `test/fixtures/synthetic/*.json`

**Interfaces:**
- Produces, from `src/errors.ts`:

```ts
export type ErrorCode = 'INVALID_ARGUMENT' | 'UPSTREAM_CONTRACT_VIOLATION' | 'LAYER_PRIVATE'
  | 'LAYER_NOT_FOUND' | 'UPSTREAM_UNAVAILABLE' | 'RESPONSE_TOO_LARGE' | 'SERVER_BUSY'
  | 'RESOURCE_EXPIRED';
export class TlcmapError extends Error {
  constructor(readonly code: ErrorCode, message: string, readonly details: Record<string, unknown> = {});
}
export function toToolErrorResult(e: TlcmapError): CallToolResult;   // spec §7 shape
export function toProtocolError(e: TlcmapError): ProtocolError;      // data = {code, message, details}
```

- Produces, from `src/summarise/layer.ts`:

```ts
export type Extent = [minLon: number, minLat: number, maxLon: number, maxLat: number];
export interface DateBound { udate: number; as_written: string | null }
export interface LayerSummary {
  metadata: Record<string, unknown>;
  record_count: number;
  extent: Extent | null;
  date_range: { start: DateBound | null; end: DateBound | null } | null;
  undated_count: number;
  ungeolocated_count: number;
  record_uids: string[];
}
export function summariseLayerFeed(body: Uint8Array): LayerSummary;   // throws TlcmapError
```

- [ ] **Step 1: Write the failing tests** `test/unit/errors.test.ts`:
  - `toToolErrorResult` returns `{isError: true, structuredContent: {error: {code, message, details}},
    content: [{type: 'text', text: `${code}: ${message}`}]}`.
  - `toProtocolError(e)` is an SDK `ProtocolError` whose `data` deep-equals
    `{code, message, details}` and whose `message` begins with the code.

- [ ] **Step 2: Write synthetic fixtures** in `test/fixtures/synthetic/` (small hand-written
  FeatureCollections): `mixed-geometry.json` (Point, LineString, Polygon, MultiPoint,
  GeometryCollection), `no-geometry.json` (two of three features `geometry: null`, one with the key
  absent), `dates.json` (start ties, one-sided records, `udatestart` without `datestart`),
  `empty.json` (`features: []`), and one file per malformed case below.

- [ ] **Step 3: Write the failing tests** `test/unit/summarise-layer.test.ts`:

```ts
test('extent covers every position of every geometry type', () =>
  expect(summarise('mixed-geometry').extent).toEqual([/* min/max computed by hand from the fixture */]));
test('extent is null and ungeolocated counts when no geometry', () => {
  const s = summarise('no-geometry');
  expect(s.extent).toBeNull(); expect(s.ungeolocated_count).toBe(3);
});
test('date_range takes min udatestart and max udateend, first in feed order on ties', ...);
test('as_written is the same record\'s datestart/dateend verbatim, or null if absent', ...);
test('one-sided range: end null when no udateend anywhere', ...);
test('date_range null and undated_count = record_count when no udate fields', ...);
test('empty layer: record_count 0, extent null, date_range null, record_uids []', ...);
test('metadata deep-equals the feed metadata, absent keys stay absent', () => {
  const s = summarise('dates');
  expect(s.metadata).toEqual(raw('dates').metadata);
  expect('license' in s.metadata).toBe(false);
});
test('record_uids are properties.id in feed order', ...);
describe('malformed features', () => {
  test.each(['not-json', 'no-features', 'features-not-array', 'properties-null',
    'id-missing', 'id-empty', 'udatestart-string', 'coordinates-missing',
    'coordinates-empty', 'coordinate-not-number', 'metadata-not-object'])(
    '%s is UPSTREAM_CONTRACT_VIOLATION', (name) => {
      expect(() => summarise(name)).toThrow(expect.objectContaining({ code: 'UPSTREAM_CONTRACT_VIOLATION' }));
    });
  test('no-features passes on the API metadata verbatim in details.api_metadata', ...);
});
```

  A hand-computed `extent` for `mixed-geometry.json` is written into the test as literals.
  `coordinates-empty` means a Point with `coordinates: []`; an empty `MultiPoint` is valid and
  contributes nothing.

- [ ] **Step 4: Run** both files — Expected: FAIL, modules not found.
- [ ] **Step 5: Implement `errors.ts`** using the SDK's `ProtocolError` (code `-32602`,
  `INVALID_PARAMS`, for `INVALID_ARGUMENT`; `-32603`, `INTERNAL_ERROR`, for everything else).
- [ ] **Step 6: Implement `summariseLayerFeed`** — decode with a fatal UTF-8 `TextDecoder`,
  `JSON.parse`, then validate by hand in one pass over features (no zod: the feed is large and
  only the fields in spec §4 are read). `null`/absent `udatestart` = no start. A record is undated
  when it has neither. `details` on every violation: `{expected, feature_index?}`; for a missing
  `features` array, also `api_metadata` (the parsed `metadata` object, verbatim, if it is an
  object) — no key is singled out by name.
- [ ] **Step 7: Run** both files — Expected: PASS.
- [ ] **Step 8: Commit** — `git commit -m "Add typed errors and the layer feed summary"`.

---

### Task 4: Byte-capped cache

**Files:**
- Create: `src/upstream/cache.ts`
- Test: `test/unit/cache.test.ts`

**Interfaces:**

```ts
export class ByteLruCache<V> {
  constructor(opts: { maxBytes: number; ttlMs: number; sizeOf: (v: V) => number; now?: () => number });
  get(key: string): V | undefined;          // undefined if absent or expired; refreshes recency
  set(key: string, value: V): void;         // evicts LRU entries until the total fits
  stats(): { entries: number; bytes: number };
}
```

- [ ] **Step 1: Write the failing tests** — with an injected clock:
  - `get` after `ttlMs` returns `undefined` and the entry no longer counts in `stats()`.
  - Inserting past `maxBytes` evicts least recently *used* first (a `get` refreshes recency).
  - Replacing a key adjusts `bytes` rather than double-counting.
  - A value larger than `maxBytes` is not stored and evicts nothing.
- [ ] **Step 2: Run** `npx vitest run test/unit/cache.test.ts` — Expected: FAIL.
- [ ] **Step 3: Implement** with a `Map` (insertion order as recency: delete and re-set on `get`).
- [ ] **Step 4: Run** — Expected: PASS.
- [ ] **Step 5: Commit** — `git commit -m "Add the byte-capped LRU cache"`.

---

### Task 5: Limiter and single-flight

**Files:**
- Create: `src/upstream/limiter.ts`
- Test: `test/unit/limiter.test.ts`

**Interfaces:**

```ts
export class Limiter {
  constructor(opts: { concurrency: number; queueMax: number; queueTimeoutMs: number; now?: () => number });
  run<T>(fn: () => Promise<T>): Promise<{ value: T; queueWaitMs: number }>;   // throws TlcmapError SERVER_BUSY
  stats(): { inFlight: number; queued: number };
}
export class SingleFlight<T> {
  do(key: string, fn: () => Promise<T>): Promise<{ value: T; shared: boolean }>;
}
```

- [ ] **Step 1: Write the failing tests** (Vitest fake timers):
  - With `concurrency: 2`, a third `run` waits until one of the first two settles; `stats()` shows
    `{inFlight: 2, queued: 1}` meanwhile.
  - With `queueMax: 1`, a fourth concurrent `run` rejects immediately with `SERVER_BUSY`,
    `details: {queued: 1, retry_after_s: 20}` (`retry_after_s` = `ceil(queueTimeoutMs / 1000)`).
  - A queued `run` still waiting after `queueTimeoutMs` rejects with `SERVER_BUSY`, and its `fn`
    is never called.
  - `queueWaitMs` reports the time spent queued.
  - A rejected `fn` frees its slot.
  - `SingleFlight`: two concurrent `do('k', fn)` call `fn` once; the first gets `shared: false`,
    the second `shared: true`; both receive the same value or the same rejection; after it
    settles, a new `do('k')` calls `fn` again.
- [ ] **Step 2: Run** `npx vitest run test/unit/limiter.test.ts` — Expected: FAIL.
- [ ] **Step 3: Implement** — a FIFO of waiters, each with its own timeout handle cleared when it
  is granted a slot.
- [ ] **Step 4: Run** — Expected: PASS.
- [ ] **Step 5: Commit** — `git commit -m "Add the upstream limiter and single-flight"`.

---

### Task 6: Upstream client, fake TLCMap and fixtures

**Files:**
- Create: `src/upstream/client.ts`, `test/fake-tlcmap.ts`, `scripts/capture-fixtures.ts`,
  `test/fixtures/layer-206.json`, `test/fixtures/layer-1125.json`,
  `test/fixtures/layer-2749-trimmed.json`, `test/fixtures/layer-missing.json`,
  `test/fixtures/CAPTURED.md`
- Test: `test/unit/upstream-client.test.ts`

**Interfaces:**

```ts
// src/upstream/client.ts
export interface UpstreamResponse { url: string; status: number; body: Buffer; fetchedAt: Date; durationMs: number }
export interface UpstreamClient { get(url: URL): Promise<UpstreamResponse> }   // 2xx only; else throws TlcmapError
export function createUpstreamClient(opts: { timeoutMs: number; maxBytes: number; userAgent: string;
  fetch?: typeof globalThis.fetch }): UpstreamClient;

// test/fake-tlcmap.ts
export interface FakeTlcmap { baseUrl: URL; requests: { path: string; headers: Record<string, string> }[];
  close(): Promise<void> }
export function startFakeTlcmap(routes?: Record<string, FakeRoute>): Promise<FakeTlcmap>;
export type FakeRoute = { status?: number; headers?: Record<string, string>; body?: string | Buffer;
  delayMs?: number; streamBytes?: number };
```

  `startFakeTlcmap()` with no argument serves `/layers/206/json`, `/layers/1125/json`,
  `/layers/2749/json` (trimmed fixture) and `/layers/99999999/json` (missing-layer fixture) with
  `Content-Type: application/json`; any other path is `404` with an empty body. Extra routes are
  merged in.

- [ ] **Step 1: Write `scripts/capture-fixtures.ts`** — fetches layers 206, 1125, 2749 and
  99999999 from `https://tlcmap.org/layers/{id}/json` with the server's `User-Agent`, one at a
  time; writes 206, 1125 and the missing layer verbatim; writes 2749 as
  `JSON.stringify({...feed, features: feed.features.slice(0, 20)})`; writes `CAPTURED.md` with,
  per file, the URL, capture time (UTC), `sha256`, and for 2749 the original byte and feature
  counts and the trimming rule.
- [ ] **Step 2: Run** `npm run capture-fixtures`, then commit the fixtures:
  `git add scripts test/fixtures && git commit -m "Capture production fixtures"`. Check
  `CAPTURED.md` shows 52 features for 206, 94 for 1125, and an empty `features` array with no `metadata.layer_id` for the missing layer.
- [ ] **Step 3: Write `test/fake-tlcmap.ts`** — a `node:http` server on `127.0.0.1:0` recording
  every request's path and headers. `streamBytes` streams that many bytes of `[` padding in 64 KB
  chunks.
- [ ] **Step 4: Write the failing tests** `test/unit/upstream-client.test.ts`, against the fake:

  | Fake route | Expected |
  | --- | --- |
  | `200` fixture 206 | resolves; `body` equals the fixture bytes; `status` 200 |
  | any request | `User-Agent` header equals the configured string |
  | `302` with `Location: /maxpaging` | `UPSTREAM_CONTRACT_VIOLATION`, `details.status` 302, `details.location` `/maxpaging`; the fake saw one request only |
  | `403` JSON body | `LAYER_PRIVATE`, `details.api_body` the parsed JSON |
  | `404` empty body | `LAYER_NOT_FOUND`, no `api_body` |
  | `400` | `UPSTREAM_CONTRACT_VIOLATION` |
  | `429` with `Retry-After: 30` | `UPSTREAM_UNAVAILABLE`, `details.retry_after` `"30"` |
  | `503` | `UPSTREAM_UNAVAILABLE` |
  | `delayMs` > `timeoutMs` | `UPSTREAM_UNAVAILABLE`, `details.elapsed_ms` ≥ `timeoutMs` |
  | connection refused (closed fake) | `UPSTREAM_UNAVAILABLE` |
  | `streamBytes` = `maxBytes + 1` | `RESPONSE_TOO_LARGE`, `details: {limit, bytes_read}`, rejected before the stream ends |

  Every error's `details.request` is `GET {url}`.
- [ ] **Step 5: Run** `npx vitest run test/unit/upstream-client.test.ts` — Expected: FAIL.
- [ ] **Step 6: Implement `createUpstreamClient`** — `fetch(url, {redirect: 'manual', headers:
  {'User-Agent': …, Accept: 'application/json'}, signal: AbortSignal.timeout(timeoutMs)})`; read
  `response.body` with a reader, counting bytes, cancelling past `maxBytes`. Status mapping in one
  function with a comment that the 403/404 codes are specific to the layer endpoint, the only one
  A0 calls.
- [ ] **Step 7: Run** — Expected: PASS.
- [ ] **Step 8: Commit** — `git commit -m "Add the upstream client and fake TLCMap"`.

---

### Task 7: Provenance and the layer service

**Files:**
- Create: `src/provenance.ts`, `src/layers/service.ts`
- Test: `test/unit/provenance.test.ts`, `test/unit/layer-service.test.ts`

**Interfaces:**

```ts
// src/provenance.ts
export type CacheOutcome = 'miss' | 'hit' | 'shared';
export interface Provenance { request: string; fetched_at: string; cache: CacheOutcome;
  sha256: string; bytes: number; record_uids: string[]; server: string }
export type FetchProvenance = Omit<Provenance, 'cache'>;
export function buildProvenance(r: UpstreamResponse, recordUids: string[], server: string): FetchProvenance;

// src/layers/service.ts
export interface LayerEntry { layerId: number; body: Buffer; summary: LayerSummary; provenance: FetchProvenance }
export interface UpstreamStats { requests_total: number; in_flight: number; queued: number;
  last_success_at: string | null; last_error: { code: ErrorCode; at: string } | null }
export interface LayerService {
  getLayer(layerId: number): Promise<{ entry: LayerEntry; cache: CacheOutcome }>;   // throws TlcmapError
  peekLayer(layerId: number): LayerEntry | undefined;                               // never fetches
  stats(): { upstream: UpstreamStats; cache: { entries: number; bytes: number } };
}
export function layerUrl(baseUrl: URL, layerId: number): URL;   // {base}/layers/{id}/json
export function checkLayerIdentity(summary: LayerSummary, layerId: number): void;
  // throws TlcmapError UPSTREAM_CONTRACT_VIOLATION unless summary.metadata.layer_id === layerId;
  // details: {expected: `metadata.layer_id to equal ${layerId}`, api_metadata: summary.metadata}
export function createLayerService(deps: { client: UpstreamClient; cache: ByteLruCache<LayerEntry>;
  limiter: Limiter; baseUrl: URL; serverId: string; logger: Logger }): LayerService;
```

- [ ] **Step 1: Write the failing tests** `test/unit/provenance.test.ts`: `sha256` is the hex
  SHA-256 of `body` (assert against `node:crypto` on the 206 fixture); `bytes` is
  `body.length`; `fetched_at` is `fetchedAt.toISOString()`; `request` is `GET {url}`; `server`
  passes through.
- [ ] **Step 2: Write the failing tests** `test/unit/layer-service.test.ts`, with a stub
  `UpstreamClient` counting calls and returning fixture 206:
  - First `getLayer(206)` → `cache: 'miss'`, one upstream call; second → `cache: 'hit'`, no new
    call, and `entry.provenance.fetched_at` equal to the first.
  - Two concurrent `getLayer(206)` → one upstream call; outcomes `miss` and `shared`.
  - **A failed shared fetch rejects every joiner and is not cached:** a stub that rejects with
    `UPSTREAM_UNAVAILABLE` → both concurrent callers reject with that code; `peekLayer(206)` is
    `undefined`; a following `getLayer` calls the client again.
  - A body that fails `summariseLayerFeed` (a synthetic fixture with no `features`) rejects with
    `UPSTREAM_CONTRACT_VIOLATION` and is not cached.
  - **The missing-layer fixture** (`features: []`, `metadata` holding only `warnnig`) requested as
    layer 99999999 rejects with `UPSTREAM_CONTRACT_VIOLATION`, `details.api_metadata` deep-equal to
    the fixture's `metadata`, and is not cached. `checkLayerIdentity` unit tests: matching
    `layer_id` passes; absent, string `"206"`, or a different number throws. A real layer with zero
    features and the right `layer_id` summarises normally (`record_count` 0).
  - `peekLayer` returns the cached entry and never calls the client.
  - `stats().upstream`: `requests_total` counts client calls only (not hits); `last_success_at`
    and `last_error` update.
  - The logger receives one `upstream` event per client call with `{url, status, bytes,
    queue_wait_ms, duration_ms}`, and for a failure `{url, outcome: code, duration_ms}`.
- [ ] **Step 3: Run** both files — Expected: FAIL.
- [ ] **Step 4: Implement** — the cache key is `layerUrl(...).href`. The order is: cache `get`,
  else `SingleFlight.do(key, () => limiter.run(() => client.get(url)))`, then summarise, then
  `checkLayerIdentity`, then `cache.set`. `sizeOf` for the cache is `entry.body.length`. Only a fully summarised entry is
  cached.
- [ ] **Step 5: Run** — Expected: PASS.
- [ ] **Step 6: Commit** — `git commit -m "Add provenance and the layer service"`.

---

### Task 8: MCP surface — scopes, tool, resource, server factory

**Files:**
- Create: `src/mcp/scopes.ts`, `src/tools/get-layer.ts`, `src/resources/layer-features.ts`,
  `src/mcp/server.ts`
- Test: `test/unit/mcp-surface.test.ts`

**Interfaces:**

```ts
// src/mcp/scopes.ts
export interface Registration { scope: Scope; register(server: McpServer): void }
export function registerEnabled(server: McpServer, registrations: Registration[], enabled: Scope[]): void;

// src/tools/get-layer.ts
export const GET_LAYER_TOOL = 'tlcmap_get_layer';
export function getLayerTool(deps: { layers: LayerService; logger: Logger }): Registration;

// src/resources/layer-features.ts
export const LAYER_FEATURES_TEMPLATE = 'tlcmap://layer/{id}/features';
export function layerFeaturesUri(layerId: number): string;
export function parseLayerId(raw: string): number;   // throws TlcmapError INVALID_ARGUMENT
export function layerFeaturesResource(deps: { layers: LayerService; logger: Logger }): Registration;

// src/mcp/server.ts
export function createServerFactory(deps: { layers: LayerService; logger: Logger; scopes: Scope[] }): McpServerFactory;
```

  Tests connect `@modelcontextprotocol/client`'s `Client` to a server from the factory over
  `InMemoryTransport.createLinkedPair()`, with a stub `LayerService` serving fixture entries.

- [ ] **Step 1: Write the failing tests** `test/unit/mcp-surface.test.ts`:

```ts
test('tools/list has exactly tlcmap_get_layer', async () =>
  expect((await client.listTools()).tools.map(t => t.name)).toEqual(['tlcmap_get_layer']));
test('input schema is layer_id integer ≥ 1 ≤ 2147483647, no other properties', ...);
test('registerEnabled skips registrations outside the enabled scopes', ...);  // a fake 'write' Registration is not listed
test('get_layer returns the summary, resource_uri and provenance', async () => {
  const r = await client.callTool({ name: 'tlcmap_get_layer', arguments: { layer_id: 206 } });
  expect(r.isError).toBeFalsy();
  expect(r.structuredContent).toMatchObject({ layer_id: 206, record_count: 52, undated_count: 52,
    ungeolocated_count: 0, date_range: null, resource_uri: 'tlcmap://layer/206/features',
    provenance: { request: expect.stringMatching(/^GET .*\/layers\/206\/json$/), cache: 'miss',
      record_uids: expect.arrayContaining(['ta4f5']), server: SERVER_ID } });
  expect(r.structuredContent.metadata).toEqual(fixture206.metadata);
  expect(r.structuredContent.metadata.warning).toBe('None');
  expect('record_uids' in r.structuredContent).toBe(false);   // only inside provenance
  expect(r.content).toEqual([{ type: 'text', text:
    'Layer 206: 52 records, 52 undated, 0 without geometry. Features: tlcmap://layer/206/features' }]);
});
test('2749 warning passes through unchanged', ...);   // equals fixture string, HTML included
test('a TlcmapError from the service becomes an isError result with structuredContent.error', ...);
test('invalid layer_id is rejected by the SDK before the handler runs', async () => {
  const r = await client.callTool({ name: 'tlcmap_get_layer', arguments: { layer_id: 'abc' } });
  expect(r.isError).toBe(true);
  expect(r.content[0].text).toMatch(/^Input validation error:/);
  expect(stubLayers.calls).toBe(0);
});
test('resource read serves the cached bytes with provenance in _meta', ...);   // text === body.toString('utf8'); _meta.provenance.sha256 === entry's
test('resource text round-trips multi-byte UTF-8 to the reported sha256', ...);
  // stub entry body contains "Ngunnawal – ‘Canberra’ — Mparntwe &amp;"; sha256(Buffer.from(text,'utf8')) === _meta.provenance.sha256
test('resource read with no cached entry is RESOURCE_EXPIRED, naming the tool', ...);
test.each(['007', '0', '-1', '1.5', 'abc', '99999999999'])('resource rejects non-canonical layer ids: %s', ...);
  // protocol error, data.code INVALID_ARGUMENT, stub peekLayer never called
test('resources/list is empty and the template is listed', ...);
```

  The tool-level cache outcome comes from `getLayer`'s `cache` and is merged into `provenance`.
  The resource's `_meta.provenance` has `cache: 'hit'`.

- [ ] **Step 2: Run** `npx vitest run test/unit/mcp-surface.test.ts` — Expected: FAIL.
- [ ] **Step 3: Implement.** `getLayerTool` registers with
  `inputSchema: z.strictObject({layer_id: z.number().int().min(1).max(2147483647)})` and an
  `outputSchema` zod object mirroring spec §4's output (with `metadata: z.record(z.string(),
  z.unknown())`), with this description:

  > Fetch one TLCMap layer by its numeric ID. Returns a summary computed from the layer's records —
  > record count, extent, date range, undated and ungeolocated counts — the layer's metadata exactly
  > as TLCMap holds it, a resource URI for the full GeoJSON features, and provenance. Licence,
  > rights and warning text are returned verbatim and must be shown to the user, not paraphrased.

  The tool catches only `TlcmapError` (→ `toToolErrorResult`); anything else propagates. It logs
  one `tool` event `{tool, arguments, outcome, cache, duration_ms}`. `parseLayerId` accepts
  `/^[1-9][0-9]{0,9}$/` and values ≤ 2147483647 only. The resource registers
  `new ResourceTemplate(LAYER_FEATURES_TEMPLATE, {list: undefined})` with
  `mimeType: 'application/json'`; a miss throws `toProtocolError(new TlcmapError('RESOURCE_EXPIRED',
  …, {uri, tool: 'tlcmap_get_layer'}))`; it logs one `resource` event. `createServerFactory`
  returns `() => { const s = new McpServer({name: SERVER_NAME, version: SERVER_VERSION});
  registerEnabled(s, [getLayerTool(deps), layerFeaturesResource(deps)], deps.scopes); return s; }`.
- [ ] **Step 4: Run** — Expected: PASS.
- [ ] **Step 5: Commit** — `git commit -m "Add the tool, the resource and the scope registry"`.

---

### Task 9: HTTP app, health, rate limit and entry point

**Files:**
- Create: `src/http/app.ts`, `src/index.ts`, `test/helpers/client.ts`
- Test: `test/integration/server.test.ts`

**Interfaces:**

```ts
// src/http/app.ts
export interface HealthBody { status: 'ok'; version: string; uptime_s: number; scopes: Scope[];
  cache: { entries: number; bytes: number }; upstream: UpstreamStats }
export function createApp(deps: { config: Config; factory: McpServerFactory; logger: Logger;
  health: () => HealthBody }): { app: Express; close(): Promise<void> };

// src/index.ts — no exports; builds everything from loadConfig(process.env), listens on
// config.host:config.port, logs `startup` {version, scopes, limits}; on SIGTERM/SIGINT closes the
// HTTP server and the MCP handler, logs `shutdown`, exits 0.

// test/helpers/client.ts
export function startTestServer(env?: Record<string, string>): Promise<{ url: URL; app: Express;
  fake: FakeTlcmap; client: Client; logs: Record<string, unknown>[]; close(): Promise<void> }>;
```

- [ ] **Step 1: Write the failing tests** `test/integration/server.test.ts`. `startTestServer`
  starts the fake TLCMap, builds the real stack with `TLCMAP_BASE_URL` set to the fake and
  `PUBLIC_URL=https://test-mcp.tlcmap.org`, captures log lines, listens on port 0, and connects a
  `Client` over `StreamableHTTPClientTransport`.
  - `tlcmap_get_layer` on 206, 1125 and 2749 → summaries matching each fixture (1125:
    `record_count` 94, `undated_count` 0, `date_range` non-null; 2749: warning string equal to the
    fixture's).
  - Resource read of the returned URI → `sha256(text)` equals the tool's `provenance.sha256`.
  - Second call → `provenance.cache` `'hit'`, same `fetched_at`; the fake saw one request.
  - Layer 99999999 → `isError`, `structuredContent.error.code` `UPSTREAM_CONTRACT_VIOLATION`,
    `details.api_metadata` equal to the missing-layer fixture's `metadata`.
  - Fake routes for `302`, `503`, slow (`UPSTREAM_TIMEOUT_MS=200`), oversized
    (`UPSTREAM_MAX_BYTES=1024`, `CACHE_MAX_BYTES=1024`) → `UPSTREAM_CONTRACT_VIOLATION`,
    `UPSTREAM_UNAVAILABLE`, `UPSTREAM_UNAVAILABLE`, `RESPONSE_TOO_LARGE`.
  - `SERVER_BUSY`: `UPSTREAM_CONCURRENCY=1`, `UPSTREAM_QUEUE_MAX=1`, a slow route; three
    concurrent calls for distinct layers → the third is `SERVER_BUSY`.
  - `RATE_LIMIT_PER_MIN=3` → the fourth `POST /mcp` in a minute is `429` with `Retry-After`;
    `/healthz` is still `200`.
  - Proxy trust: `app.get('trust proxy fn')('127.0.0.1', 0)` is `true`;
    `app.get('trust proxy fn')('203.0.113.5', 0)` is `false`.
  - `Host: evil.example` on `POST /mcp` → `403`; `Host: test-mcp.tlcmap.org` → accepted.
  - `GET /mcp` and `DELETE /mcp` → `405`.
  - `/healthz` → `200` with the `HealthBody` keys; `upstream.requests_total` equals the fake's
    request count; it makes no request to the fake.
  - Logs: each tool call produced one `tool` line and, on a miss, one `upstream` line; each HTTP
    request one `http` line with `{method, path, status, ip, duration_ms}`.
- [ ] **Step 2: Run** `npx vitest run test/integration/server.test.ts` — Expected: FAIL.
- [ ] **Step 3: Implement `createApp`** —
  `createMcpExpressApp({host: config.host, allowedHosts: [config.publicUrl.hostname, 'localhost', '127.0.0.1']})`;
  `app.set('trust proxy', 'loopback')`; an `http` logging middleware on `finish`; `GET /healthz`
  before the limiter; `rateLimit({windowMs: 60_000, limit: config.rateLimitPerMin,
  standardHeaders: 'draft-8', legacyHeaders: false})` on `POST /mcp`; then
  `toNodeHandler(createMcpHandler(factory, {responseMode: 'json', onerror}))` called as
  `node(req, res, req.body)`; `GET`/`DELETE /mcp` → `405`. `close()` closes the MCP handler.
- [ ] **Step 4: Implement `src/index.ts`** — wire `loadConfig` → `createLogger` →
  `createUpstreamClient({userAgent: `tlcmap-mcp/${SERVER_VERSION} (+${config.publicUrl.origin})`})` →
  `ByteLruCache` → `Limiter` → `createLayerService` → `createServerFactory` → `createApp`. A
  `ConfigError` is written to stderr as one line and exits with code 1.
- [ ] **Step 5: Run** `npm test` — Expected: PASS.
- [ ] **Step 6: Smoke-run locally** — `npm run build && PUBLIC_URL=http://localhost:3100 npm start`,
  then `claude mcp add --transport http tlcmap-local http://localhost:3100/mcp` and call
  `tlcmap_get_layer` with `layer_id` 206 from Claude Code. Expected: 52 records and a provenance
  envelope; one `upstream` log line to `https://tlcmap.org/layers/206/json`.
- [ ] **Step 7: Commit** — `git commit -m "Add the HTTP app, health and entry point"`.

---

### Task 10: Production compatibility check

**Files:**
- Create: `vitest.prod.config.ts`, `test/prod/compat.test.ts`

**Interfaces:**
- Consumes: `createUpstreamClient`, `summariseLayerFeed`, `layerUrl`, `TlcmapError`.

- [ ] **Step 1: Write `vitest.prod.config.ts`** — `include: ['test/prod/**']`,
  `testTimeout: 90_000`, `fileParallelism: false`.
- [ ] **Step 2: Write `test/prod/compat.test.ts`** against `https://tlcmap.org`, one request per
  test, sequentially:
  - Layer 206: top-level keys include `type`, `metadata`, `features`; `metadata` has `layer_id`,
    `name`, `warning`; `summariseLayerFeed` succeeds; `record_count` > 0.
  - Layer 1125: every feature has a non-empty string `properties.id`; at least one has numeric
    `udatestart` and `udateend`.
  - Layer 99999999: `client.get` resolves `200`, `summariseLayerFeed` succeeds with `record_count` 0,
    and `checkLayerIdentity(summary, 99999999)` throws
    `UPSTREAM_CONTRACT_VIOLATION`. The test's comment: *"Fails when DESIGN.md §5.1 ② lands. Then
    expect client.get to throw LAYER_NOT_FOUND, and update spec §14."*
- [ ] **Step 3: Run** `npm run test:prod` — Expected: PASS against today's production.
- [ ] **Step 4: Confirm** `npm test` does not run it (the default config's `include` excludes
  `test/prod`).
- [ ] **Step 5: Commit** — `git commit -m "Add the production compatibility check"`.

---

### Task 11: Deployment artefacts

**Files:**
- Create: `deploy/tlcmap-mcp.service`, `deploy/test-mcp.tlcmap.org.conf`, `deploy/env.example`,
  `deploy/deploy.sh`, `deploy/DEPLOY.md`
- Test: `test/deploy/env-example.test.ts`

**Interfaces:**
- Consumes: `loadConfig`, `CONFIG_VARIABLES`.

- [ ] **Step 1: Write the failing test** `test/deploy/env-example.test.ts`: parse
  `deploy/env.example` (`KEY=VALUE` lines; `#` comments) → the set of keys equals
  `CONFIG_VARIABLES`, and `loadConfig(parsed)` succeeds with `publicUrl.href` equal to
  `https://test-mcp.tlcmap.org/`.
- [ ] **Step 2: Run** — Expected: FAIL, file missing.
- [ ] **Step 3: Write `deploy/env.example`** — every variable with its spec §9 default and a
  one-line comment; `PUBLIC_URL=https://test-mcp.tlcmap.org`.
- [ ] **Step 4: Run** — Expected: PASS.
- [ ] **Step 5: Write `deploy/tlcmap-mcp.service`** exactly per spec §11: `User=tlcmap-mcp`,
  `EnvironmentFile=/etc/tlcmap-mcp/env`, `WorkingDirectory=/opt/tlcmap-mcp/current`,
  `ExecStart=/usr/bin/node --max-old-space-size=1024 /opt/tlcmap-mcp/current/dist/index.js`,
  `Restart=on-failure`, `RestartSec=2`, `MemoryMax=2G`, `NoNewPrivileges=yes`,
  `ProtectSystem=strict`, `ProtectHome=yes`, `PrivateTmp=yes`, `WantedBy=multi-user.target`.
- [ ] **Step 6: Write `deploy/test-mcp.tlcmap.org.conf`** — a `*:80` vhost that redirects to
  HTTPS, and a `*:443` vhost with `ServerName test-mcp.tlcmap.org`, `ProxyPreserveHost On`,
  `ProxyTimeout 90`, `ProxyPass / http://127.0.0.1:3100/` and `ProxyPassReverse`, and
  `RequestHeader set X-Forwarded-Proto https`. The certificate lines are left for `certbot --apache`
  to add.
- [ ] **Step 7: Write `deploy/deploy.sh <git-ref>`** — `set -euo pipefail`; must run as root;
  clones `https://github.com/Systemik-Solutions/tlcmap-mcp.git` at the ref into
  `/opt/tlcmap-mcp/releases/<sha>`; `npm ci && npm run build && npm prune --omit=dev` as
  `tlcmap-mcp`; records the previous `current` target; `ln -sfn` the new one;
  `systemctl restart tlcmap-mcp`; polls `http://127.0.0.1:${PORT:-3100}/healthz` once a second for
  30 s, expecting `version` equal to the release's `package.json` version. On failure it re-points
  `current` to the previous target, restarts, and exits 1. On success it deletes all but the three
  newest release directories, never the one `current` points to. `--rollback` re-points to the
  previous release and restarts.
- [ ] **Step 8: Check the script** — `bash -n deploy/deploy.sh`, and `shellcheck deploy/deploy.sh`
  if available. Expected: no output.
- [ ] **Step 9: Write `deploy/DEPLOY.md`** — one-time setup in order:
  1. DNS A record for `test-mcp.tlcmap.org`.
  2. NodeSource Node 24: `curl -fsSL https://deb.nodesource.com/setup_24.x | sudo -E bash -` then
     `sudo apt install -y nodejs`; confirm `node --version` is `v24.*` and the path is
     `/usr/bin/node`.
  3. `sudo useradd --system --home /opt/tlcmap-mcp --shell /usr/sbin/nologin tlcmap-mcp`, plus the
     directories and the env file (`root:tlcmap-mcp`, `0640`).
  4. Clone the repository over HTTPS as root (it is public; no deploy key).
  5. Confirm port 3100 is free: `ss -ltnp | grep 3100` prints nothing.
  6. Install the unit and `systemctl enable tlcmap-mcp`.
  7. `a2enmod proxy proxy_http headers ssl`, install the vhost, `apachectl configtest`, reload,
     `certbot --apache -d test-mcp.tlcmap.org`.
  8. First deploy.

  Then the runbook: deploy, rollback, `journalctl -u tlcmap-mcp -f`, reading `/healthz`, raising
  limits through the env file, and pointing an external uptime check at
  `https://test-mcp.tlcmap.org/healthz` (never at `/mcp`).
- [ ] **Step 10: Run** `npm test` — Expected: PASS.
- [ ] **Step 11: Commit** — `git commit -m "Add deployment artefacts for staging"`.

---

### Task 12: Deploy to staging and record acceptance

Run on the staging server by someone with root, following `deploy/DEPLOY.md`.

**Files:**
- Create: `docs/acceptance/A0.md` (in `tlcmap-mcp`)

- [ ] **Step 1: Pre-flight** — on a workstation: `npm ci && npm test && npm run test:prod`.
  Expected: PASS. Tag the commit `v0.1.0` and push the tag.
- [ ] **Step 2: One-time setup** — DEPLOY.md steps 1–7. Expected: `apachectl configtest` prints
  `Syntax OK`, and certbot reports a certificate installed for `test-mcp.tlcmap.org`.
- [ ] **Step 3: Deploy** — `sudo deploy/deploy.sh v0.1.0`. Expected: it prints the served
  version `0.1.0`.
- [ ] **Step 4: Run spec §13's checks 2–8 in order.** For check 3, compare 2749's `warning`
  against `curl -s https://tlcmap.org/layers/2749/json?metadata`. For check 8, deploy a second
  release (bump the version to `0.1.1` with a README-only change), confirm `/healthz` reports
  `0.1.1`, run `deploy.sh --rollback`, and confirm it reports `0.1.0`.
- [ ] **Step 5: Record** — write `docs/acceptance/A0.md`: the date, the commit and tag, the
  Node version, then each of the eight checks with its result and the command or output that
  shows it. A failed check is recorded as failed, with what happened.
- [ ] **Step 6: Commit** — `git commit -m "Record A0 acceptance on staging"`; push.
