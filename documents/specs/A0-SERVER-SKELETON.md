# A0 — MCP server skeleton

**Status:** spec for build · **Date:** 2026-11-03 · **Repository:** `tlcmap-mcp`

Milestone A0 of Phase A in [DESIGN.md §10](../DESIGN.md#phase-a--the-vertical-slice):

> The TypeScript server end to end against production: HTTP transport, one tool calling the live
> API, a resource handle, the provenance envelope, a typed error, and read and write registered as
> separate scopes. Deployed to staging the way it will run in production.

This document expands that paragraph into something buildable. DESIGN.md remains the authority on
principles; where this spec and DESIGN.md disagree, DESIGN.md is wrong or this spec is, and one of
them is fixed.

---

## Contents

| | |
| --- | --- |
| [1. Purpose and scope](#1-purpose-and-scope) | What A0 proves, what it leaves out |
| [2. Facts this spec rests on](#2-facts-this-spec-rests-on) | Measured against production |
| [3. Components](#3-components) | The units and their boundaries |
| [4. The tool](#4-the-tool-tlcmap_get_layer) | `tlcmap_get_layer`, A0 subset |
| [5. The resource](#5-the-resource-tlcmaplayeridfeatures) | `tlcmap://layer/{id}/features` |
| [6. Provenance](#6-provenance) | The envelope on every result |
| [7. Errors](#7-errors) | The typed error set |
| [8. Scopes](#8-scopes) | Read and write, separated |
| [9. Courtesy and configuration](#9-courtesy-and-configuration) | Protecting tlcmap.org |
| [10. Observability](#10-observability) | Logs and health |
| [11. Deployment](#11-deployment) | Staging, as production will run |
| [12. Testing](#12-testing) | Unit, integration, production check |
| [13. Acceptance](#13-acceptance) | When A0 is done |
| [14. API dependencies](#14-api-dependencies) | What changes when §5 items land |

---

## 1. Purpose and scope

A0 fixes the server's **shape** before the API work in §5 commits to a tool-shaped contract. It
carries one tool, but every structural concern a later tool depends on is real: transport, the
single upstream client, caching, resource handles, provenance, typed errors, scopes, courtesy
controls, logging, health, and a production-shaped deployment.

**In scope**

- A stateless MCP server over Streamable HTTP, on the official TypeScript SDK v2.
- One read tool, `tlcmap_get_layer`, accepting `layer_id` only.
- One resource template, `tlcmap://layer/{id}/features`.
- The provenance envelope, the typed error set, the scope registry.
- Courtesy controls: global upstream concurrency, a bounded queue, single-flight, a byte-capped
  cache, a per-IP rate limit, an identifying `User-Agent`.
- JSON logging to journald and a `/healthz` endpoint.
- Deployment artefacts and a deployment to staging at `test-mcp.tlcmap.org`.
- Unit and integration tests, and a production compatibility check.

**Out of scope**

- Every other tool in [DESIGN.md §6.3](../DESIGN.md#63-the-tool-surface), and the remaining
  `tlcmap_get_layer` parameters (`sort`, `nulls`, `line`, `bbox`), which wait on §5.2 ⑦.
- Any write tool, any authentication, any token handling (Phase E).
- Any change to the TLCMap application. A0 runs against production tlcmap.org as it is.
- Any workaround for behaviour §5 will fix ([§3.8](../DESIGN.md#38-fix-it-upstream)).
- Production deployment at `mcp.tlcmap.org` (Phase C).

---

## 2. Facts this spec rests on

Verified 2026-11-03.

**SDK.** The official MCP TypeScript SDK v2 is the stable line, published as separate packages:
`@modelcontextprotocol/server` 2.4.0 and `@modelcontextprotocol/client` 2.4.0 (both 2026-10-23),
and the Express adapter `@modelcontextprotocol/express` 2.0.2 (2026-10-02). All Apache-2.0, Node 20
or later, zod 4. The v1 package `@modelcontextprotocol/sdk` (1.32.2) is the maintenance line: it
does not implement the 2026-07-28 MCP specification, and its support is guaranteed only for six
months after v2's release on 2026-07-27.

**Node.** Ubuntu 22.04 ships Node 12.22.9, which the SDK cannot run on. Current LTS lines on
nodejs.org: 26.13.0 ("Lithium", active), 24.22.2 ("Krypton", maintenance), 22.23.4 ("Jod",
maintenance). A0 runs on **Node 24**, from the NodeSource apt repository.

**The layer feed**, `GET https://tlcmap.org/layers/{id}/json`:

- Returns `{type: "FeatureCollection", metadata, features, display, dataset_id}` with
  `Content-Type: application/json` and `Cache-Control: no-cache, private`, and no `ETag` or
  `Last-Modified`.
- `metadata` omits keys that have no value — layer 206 has no `license` key at all — rather than
  returning them as null.
- Each feature's `properties.id` is the TLCMap UID (`ta4f5`). Dated features carry `datestart`
  and `dateend` as the contributor wrote them (`"1826-09-01"`) and `udatestart` / `udateend` as
  epoch milliseconds computed by TLCMap (`-4523249092000`).
- Extended data is merged into `properties` alongside the built-in fields (§5.2 ⑤).

| Layer | Size | Records | Dated | Note |
| --- | --- | --- | --- | --- |
| 206 Music communities | 0.1 MB | 52 | 0 | `warning` is the literal string `"None"`; no `license` key |
| 1125 50 words project | 0.2 MB | 94 | 94 | `license`: `"Closed (subject to the access condition details)"` |
| 2749 Public schools … Aboriginal and/or Torres Strait Islander students | 4.5 MB | 1,602 | 1,602 | `warning` is HTML, beginning `<p>Aboriginal and/or Torres Strait Islander Peoples are advised…` |
| 461 Weather stations | 34.9 MB | 17,917 | 17,917 | 13.8 s to download; about 61 MB of heap once parsed in Node 24 |

**A missing layer** (`/layers/99999999/json`) returns `200` with a FeatureCollection whose
`features` is an empty array and whose `metadata` holds only a `warnnig` message — no `layer_id`,
no `name`. By shape alone it is an empty layer. §5.1 ② has not landed.

---

## 3. Components

One Node process. Each unit has one job and is testable alone.

```
src/
  index.ts                     entry point: load config, build app, listen
  config.ts                    environment → typed, validated config; exit on bad values
  log.ts                       JSON lines to stdout
  errors.ts                    the typed error set (§7) and its wire shape
  provenance.ts                builds the envelope (§6)
  upstream/client.ts           the only code that talks to tlcmap.org
  upstream/cache.ts            byte-capped LRU with TTL
  upstream/limiter.ts          global concurrency cap, bounded queue, single-flight
  summarise/layer.ts           layer feed bytes → summary (§4); pure, no I/O
  mcp/scopes.ts                scope registry (§8)
  mcp/server.ts                builds the MCP server; registers tools and resources by scope
  tools/get-layer.ts           tlcmap_get_layer
  resources/layer-features.ts  tlcmap://layer/{id}/features
  http/app.ts                  Express: POST /mcp, GET /healthz, rate limit
test/
  fixtures/                    captured production responses (§12)
  fake-tlcmap.ts               local HTTP server serving fixtures
deploy/
  tlcmap-mcp.service           systemd unit
  test-mcp.tlcmap.org.conf     Apache vhost
  env.example                  every configuration variable, documented
  deploy.sh                    build and switch a release
  DEPLOY.md                    one-time setup and runbook
scripts/
  capture-fixtures.ts          refreshes test/fixtures from production, dated
```

Two boundaries carry most of the design:

1. **`upstream/client.ts` is the only code that talks to tlcmap.org.** The `User-Agent`, the
   timeout, the size cap, the refusal to follow redirects and the status-to-error mapping live
   there, so they hold for every tool by construction, and so does
   [§6.2 rule 3](../DESIGN.md#62-design-rules): a tool cannot reach anything the API does not
   answer.
2. **Tools never see HTTP.** A tool asks the upstream layer for a URL and receives either a cache
   entry or a typed error.

**Stack.** TypeScript 6.0, ES modules, strict mode — not 7, because `typescript-eslint` 8.72
requires TypeScript below 6.1. `@modelcontextprotocol/server`, `@modelcontextprotocol/node`,
`@modelcontextprotocol/express`, Express 5, zod 4, `express-rate-limit`. Tests on Vitest, driven by
`@modelcontextprotocol/client`. Node's built-in `fetch` for upstream requests.

**Transport.** Streamable HTTP, **stateless** (no `Mcp-Session-Id`), with **JSON responses
enabled** so every reply is a single JSON body and no response is a server-sent event stream. The
SDK's `createMcpHandler` builds a fresh server instance per request from a factory and serves both
the 2026-07-28 protocol and stateless 2025-era clients; `toNodeHandler` mounts it on Express.
`POST /mcp` handles MCP; `GET /mcp` and `DELETE /mcp` return `405`.

**Host validation.** The SDK's Express app validates the `Host` header and by default admits only
localhost names. Behind Apache with `ProxyPreserveHost On` the header is `test-mcp.tlcmap.org`, so
the allowed hosts are the `PUBLIC_URL` hostname plus `localhost` and `127.0.0.1`; any other `Host`
is `400`.

---

## 4. The tool: `tlcmap_get_layer`

The A0 subset of the tool in [DESIGN.md §6.3](../DESIGN.md#tlcmap_get_layer).

**Input** — declared as a zod schema, from which the published JSON Schema is generated:

| Field | Type | Rule |
| --- | --- | --- |
| `layer_id` | integer | 1 to 2147483647, the range of the database's layer ID |

No other parameter is accepted. Unknown fields are rejected, not ignored.

**Upstream.** Exactly one request: `GET {TLCMAP_BASE_URL}/layers/{layer_id}/json`, through the
cache (§9).

**Output** — `structuredContent`, also declared as a zod output schema. Layer 206 as fetched on
2026-11-03:

```json
{
  "layer_id": 206,
  "metadata": { "…": "the feed's metadata object, verbatim" },
  "record_count": 52,
  "extent": [115.7544702, -42.88144955, 151.6112744, -31.95200028],
  "date_range": null,
  "undated_count": 52,
  "ungeolocated_count": 0,
  "resource_uri": "tlcmap://layer/206/features",
  "provenance": { "…": "§6" }
}
```

The tool result also carries a short text content block restating the counts and the resource URI,
for clients that do not read `structuredContent`.

Every field is computed by code from the one response:

| Field | Rule |
| --- | --- |
| `metadata` | The feed's `metadata` object, **verbatim**. No key is added, renamed, removed or reordered, and no value is trimmed, decoded or interpreted. An absent `license` stays absent; `warning: "None"` stays `"None"`; HTML in `warning` stays HTML ([§3.3](../DESIGN.md#33-attribution-travels-permission-is-never-guessed), [§6.6](../DESIGN.md#66-what-the-server-must-not-do)). |
| `record_count` | The number of features. |
| `extent` | `[minLon, minLat, maxLon, maxLat]` over every position of every geometry (Point, LineString, Polygon, their Multi forms, GeometryCollection). Computed, never taken from contributor-declared fields. `null` when no feature has a geometry. |
| `date_range` | `{start: {udate, as_written}, end: {udate, as_written}}`. `start` is the feature with the smallest numeric `udatestart`; `end` the feature with the largest numeric `udateend`; ties go to the first in feed order. `as_written` is that same feature's `datestart` or `dateend` string, verbatim — never reformatted, because mixed precision is correct for historical data ([§5.5](../DESIGN.md#55-what-stays-in-the-client)). Either side is `null` if no feature supplies it; the whole field is `null` if neither side has a value. |
| `undated_count` | Features with neither `udatestart` nor `udateend`. |
| `ungeolocated_count` | Features whose `geometry` is null or absent. Additive to §6.3; it is what lets a brief state "records without coordinates" ([§9.4](../DESIGN.md#94-success-criteria) criterion 5). |
| `resource_uri` | `tlcmap://layer/{layer_id}/features`. |

**The response must be the expected shape, and about the layer requested.** The tool reads
`features` as an array, each feature's `properties.id` as a non-empty string, and `udatestart` /
`udateend`, where present and not null, as finite numbers; and `metadata.layer_id` must equal the
requested `layer_id`. A response that does not satisfy this is `UPSTREAM_CONTRACT_VIOLATION` (§7).
The tool never returns a summary of a response it could not read, nor of a response describing some
other layer or none — which is what keeps today's missing-layer response from passing as an empty
layer.

Summarising one response is code computing over data TLCMap returned. It is not the stitching of
several calls to approximate a missing capability that rule 3 forbids.

**Known limitation — depends on §5.2 ⑤.** Because extended data is merged into `properties`, a
contributor column named `id`, `datestart`, `udatestart` or similar could shadow the built-in
field, and the server cannot tell which it is reading. A0 records this and does not guess.

---

## 5. The resource: `tlcmap://layer/{id}/features`

Registered as a resource template. A handle is obtained from a tool result; `resources/list`
returns no entries.

- **A read serves the exact cached bytes** of the feed the tool fetched, as UTF-8 text with
  `mimeType: application/json`. The read's `_meta` carries that cache entry's provenance (§6),
  including the same `sha256` the tool reported, so a client can confirm that the payload is the
  data the summary describes.
- **A read never fetches.** If the cache holds no fresh entry for that layer — TTL elapsed, evicted
  under the byte cap, or the process restarted — the read fails with `RESOURCE_EXPIRED`, and the
  client calls the tool again ([§6.4](../DESIGN.md#64-resources)). A resource is never silently
  rebuilt from a newer fetch.
- `{id}` must be a canonical decimal layer ID — no sign, no leading zero, no fraction, within
  `layer_id`'s range — so `007` is never read as layer 7. Anything else is `INVALID_ARGUMENT`.

---

## 6. Provenance

Every tool result and every resource read carries the envelope
([§3.2](../DESIGN.md#32-provenance-by-construction), §6.2 rule 4):

```json
{
  "request": "GET https://tlcmap.org/layers/206/json",
  "fetched_at": "2026-11-03T04:12:09.123Z",
  "cache": "miss",
  "sha256": "…hex digest of the exact response bytes…",
  "bytes": 109369,
  "record_uids": ["ta4f5", "…"],
  "server": "tlcmap-mcp 0.1.0"
}
```

| Field | Rule |
| --- | --- |
| `request` | The method and full URL the server actually sent. Reproducible with `curl`. |
| `fetched_at` | When the upstream response was received, UTC, millisecond ISO 8601. On a cache hit, the time of the **original** fetch — provenance never claims data is fresher than it is. |
| `cache` | `miss`, `hit`, or `shared` (joined an identical request already in flight). |
| `sha256`, `bytes` | Over the response body exactly as received, before any parsing. |
| `record_uids` | Every feature's `properties.id`, in feed order. **Never truncated.** For layer 461 this is 17,917 identifiers, about 160 KB. |
| `server` | Package name and version, from `package.json`. |

---

## 7. Errors

Tool failures are returned as tool results with `isError: true`, so the model sees them and can act
on them (§6.2 rule 5):

```json
{
  "isError": true,
  "structuredContent": {
    "error": { "code": "UPSTREAM_CONTRACT_VIOLATION", "message": "…", "details": { "…": "…" } }
  },
  "content": [{ "type": "text", "text": "UPSTREAM_CONTRACT_VIOLATION: …" }]
}
```

Resource-read failures are MCP protocol errors carrying the same `{code, message, details}` in their
`data`.

| Code | Raised when | `details` |
| --- | --- | --- |
| `INVALID_ARGUMENT` | A resource URI whose `{id}` is not a canonical layer ID (§5). Tool input that fails its schema never reaches the handler: the SDK rejects it as an `isError` result whose text begins `Input validation error:` and names the field, without `structuredContent`. | The failing field and rule. |
| `UPSTREAM_CONTRACT_VIOLATION` | A `2xx` response that is not JSON, does not have the shape in §4, or whose `metadata.layer_id` is not the requested layer; or any `3xx`. Redirects are never followed. | `status`, the `request`, what was expected, and the API's `metadata` object verbatim as `api_metadata` — for the current missing-layer response, that carries the `warnnig` text. |
| `LAYER_PRIVATE` | Upstream `403`. | `status`, `request`, the API's JSON body. |
| `LAYER_NOT_FOUND` | Upstream `404`. | As above. |
| `UPSTREAM_UNAVAILABLE` | Timeout, network failure, `429` or `5xx`. | `status` if any, `request`, elapsed time, and upstream `Retry-After` if sent. |
| `RESPONSE_TOO_LARGE` | The body exceeds `UPSTREAM_MAX_BYTES`. The server's own limit, distinct from the API's future `413`. | The limit, the bytes read before stopping. |
| `SERVER_BUSY` | The upstream queue is full, or a request waited past its queue timeout (§9). | Queue length, a suggested retry delay. |
| `RESOURCE_EXPIRED` | A resource read with no fresh cache entry. | The URI, and the tool to call. |

`UPSTREAM_CONTRACT_VIOLATION` is the error a missing or private layer produces **today**. It does
not claim to know why the response does not describe the requested layer, and so never guesses
between "not found" and "private"; it reports that the API returned something other than the layer
asked for and passes on what the API said. That is ordinary validation — the server checks that the
response is about the thing it requested — and not the content sniffing
[§7.5](../DESIGN.md#75-what-is-deliberately-not-built) rules out: no message text or key name, such
as `warnnig`, is ever inspected. When §5.1 ②
lands, the same request becomes `LAYER_NOT_FOUND` with no server change.

Any other upstream `4xx` is `UPSTREAM_CONTRACT_VIOLATION`, since `tlcmap_get_layer` sends nothing
the API should reject.

The per-IP rate limit (§9) is an HTTP `429` with `Retry-After`, applied before MCP is reached. It is
not a tool error.

---

## 8. Scopes

Every tool and resource declares a scope, `read` or `write`. `TLCMAP_MCP_SCOPES` lists the scopes a
deployment enables, and only registrations in an enabled scope are made — a disabled tool does not
appear in `tools/list`, rather than appearing and refusing.

In A0 every registration is `read`, and **enabling `write` stops the server at startup** with a
message saying that write tools require Phase E's token pass-through. The separation is real and
enforced from the first commit (§6.2 rule 6) without pretending a capability exists.

---

## 9. Courtesy and configuration

One machine serves both the TLCMap application and its API, with no rate limiting
([§3.6](../DESIGN.md#36-courtesy-is-a-feature)). The controls protect tlcmap.org first.

| Control | Variable | Default | Rule |
| --- | --- | --- | --- |
| Upstream concurrency | `UPSTREAM_CONCURRENCY` | 4 | Global, across all callers. Cache hits and single-flight joins do not take a slot. |
| Upstream queue | `UPSTREAM_QUEUE_MAX` | 20 | Requests waiting for a slot. Beyond it, `SERVER_BUSY`. |
| Queue wait | `UPSTREAM_QUEUE_TIMEOUT_MS` | 20000 | A request still waiting after this is `SERVER_BUSY`. Queue wait plus the upstream timeout stays under Apache's 90 s `ProxyTimeout`. |
| Single-flight | — | always on | Identical concurrent requests share one fetch and one result. |
| Upstream timeout | `UPSTREAM_TIMEOUT_MS` | 60000 | Whole request, including the body. |
| Response size cap | `UPSTREAM_MAX_BYTES` | 67108864 (64 MB) | Enforced while streaming the body, not after. |
| Cache size | `CACHE_MAX_BYTES` | 1073741824 (1 GB) | Sum of stored body bytes; least recently used evicted first. |
| Cache TTL | `CACHE_TTL_S` | 3600 | |
| Per-IP rate limit | `RATE_LIMIT_PER_MIN` | 60 | On `POST /mcp`. Client IP from `X-Forwarded-For`, trusted only from loopback. |

**The cache** is keyed by the canonical upstream URL. An entry holds the body bytes, its provenance
and the computed summary, so a cache hit never re-parses. Only `2xx` responses that pass the shape
check are cached; errors are not. Bodies are stored as `Buffer`s, outside the V8 heap.

**`User-Agent`:** `tlcmap-mcp/{version} (+{PUBLIC_URL})`, so TLCMap can tell this server's traffic
apart ([§6.1](../DESIGN.md#61-architecture)).

**Memory budget.** About 80 MB baseline, plus up to four in-flight responses at 64 MB raw and about
110 MB parsed each, plus 1 GB of cache: about 1.8 GB at worst, well under 150 MB typically.
systemd's `MemoryMax=2G` is the hard ceiling, and Node runs with `--max-old-space-size=1024`, which
covers the transient parses since cached bodies live outside the heap.

**Other configuration**

| Variable | Default | |
| --- | --- | --- |
| `HOST` | `127.0.0.1` | Never a public interface; Apache fronts it. |
| `PORT` | `3100` | |
| `PUBLIC_URL` | — (required) | `https://test-mcp.tlcmap.org` on staging. |
| `TLCMAP_BASE_URL` | `https://tlcmap.org` | Tests point it at the fake. |
| `TLCMAP_MCP_SCOPES` | `read` | §8. |
| `LOG_LEVEL` | `info` | |

`config.ts` validates every variable at startup — type, range, URL form — and the process exits
naming the variable at fault.

---

## 10. Observability

**Logs** are JSON lines on stdout, collected by journald (`journalctl -u tlcmap-mcp`). One line per
event:

| Event | Fields |
| --- | --- |
| `http` | method, path, status, client IP, duration |
| `tool` | tool, arguments, outcome (`ok` or the error code), cache outcome, duration |
| `resource` | URI, outcome, duration |
| `upstream` | URL, status, bytes, queue wait, duration |
| `startup` / `shutdown` | version, enabled scopes, effective limits |

Counting `upstream` lines gives [§11](../DESIGN.md#11-measurement)'s courtesy measure. The queue
wait on each `upstream` line is the evidence for whether `UPSTREAM_CONCURRENCY` ever constrains
callers.

**`GET /healthz`** returns `200` with (values illustrative):

```json
{
  "status": "ok",
  "version": "0.1.0",
  "uptime_s": 86400,
  "scopes": ["read"],
  "cache": { "entries": 12, "bytes": 52428800 },
  "upstream": { "requests_total": 40, "in_flight": 0, "queued": 0,
                "last_success_at": "…", "last_error": null }
}
```

Upstream health is reported **passively**, from the last real request. `/healthz` never contacts
tlcmap.org, so an external uptime checker polling it never becomes a poller of production
([§12](../DESIGN.md#12-risks-and-ethics)). `/healthz` is exempt from the rate limit.

**Restart.** systemd restarts the process on failure. A restart empties the cache, which §5 already
accounts for.

---

## 11. Deployment

Staging is an Ubuntu 22.04 host running Apache, which also serves the staging TLCMap application
(`test-ghap.tlcmap.org`). The MCP server is served at **`test-mcp.tlcmap.org`** and calls
**production** tlcmap.org.

**Runtime.** Node 24 from the NodeSource apt repository, installed to `/usr/bin/node` and patched
by `apt upgrade`. A system user `tlcmap-mcp`, with no login shell, runs the service.

**Layout**

```
/opt/tlcmap-mcp/releases/<git-sha>/    a built release: dist/ and production node_modules
/opt/tlcmap-mcp/current                symlink to the release systemd runs
/etc/tlcmap-mcp/env                    configuration; root:tlcmap-mcp, mode 0640
```

**`deploy/deploy.sh <git-ref>`**, run on the server:

1. Clones the ref into a new release directory.
2. Runs `npm ci`, `npm run build`, `npm prune --omit=dev`.
3. Switches `current`, restarts the service.
4. Polls `/healthz` until it answers or 30 s pass, and prints the version it serves. On failure,
   switches back and restarts.

It keeps the three most recent releases. Rollback is the same switch, by hand.

**`deploy/tlcmap-mcp.service`**

- `ExecStart=/usr/bin/node --max-old-space-size=1024 /opt/tlcmap-mcp/current/dist/index.js`
- `EnvironmentFile=/etc/tlcmap-mcp/env`, `User=tlcmap-mcp`
- `Restart=on-failure`, `MemoryMax=2G`
- `NoNewPrivileges=yes`, `ProtectSystem=strict`, `ProtectHome=yes`, `PrivateTmp=yes` — the server
  writes nothing to disk and needs no writable path.

**`deploy/test-mcp.tlcmap.org.conf`**

- `mod_proxy` and `mod_proxy_http` forwarding to `http://127.0.0.1:3100/`, with
  `ProxyPreserveHost On` and `ProxyTimeout 90`, so Apache never cuts off a request still within the
  60 s upstream timeout.
- TLS from `certbot --apache`.
- No SSE configuration is needed, because the transport sends JSON responses only (§3).

**`deploy/DEPLOY.md`** documents the one-time setup — the DNS A record, NodeSource, the user and
directories, the unit, the vhost and modules, certbot, confirming port 3100 is free — then the
deploy and rollback runbook, reading logs, and pointing an external uptime check at `/healthz`.

**Production** at `mcp.tlcmap.org` (Phase C) uses the same artefacts with a different hostname and
environment file.

---

## 12. Testing

Vitest. Built test-first. CI on GitHub Actions runs typecheck, lint, unit and integration tests on
every push; none of them touch the network.

**Unit tests**, per unit in §3:

- `config` — each variable's validation; `write` in `TLCMAP_MCP_SCOPES` stops startup.
- `summarise/layer` — `extent` over every geometry type; `date_range` with `as_written` verbatim and
  first-in-feed-order ties; one-sided ranges; `undated_count`; `ungeolocated_count`; an empty
  layer; each shape violation.
- `provenance` — `sha256` and `bytes` over the exact body; original `fetched_at` on a hit;
  `record_uids` complete and in order.
- `upstream/client` — every row of §7's table, from status and body to error code; redirects not
  followed; the size cap enforced mid-stream; the `User-Agent` sent.
- `upstream/cache` — byte cap, TTL, LRU order; errors not cached.
- `upstream/limiter` — the concurrency cap; `SERVER_BUSY` past the queue limit and past the queue
  timeout; single-flight
  sharing one fetch.
- `mcp/scopes` — disabled scopes not registered.

**Fixtures**, captured from production by `scripts/capture-fixtures.ts`, each with its capture date:

| Fixture | Why |
| --- | --- |
| Layer 206, full | Undated; `warning: "None"`; no `license` key |
| Layer 1125, full | Dated; the `"Closed…"` licence |
| Layer 2749, trimmed | `metadata` unchanged; `features` cut to the first 20, to keep 4.5 MB out of the repository. The HTML warning must pass through unchanged. |
| Missing layer | The current `200` with `warnnig`, empty `features` and no `layer_id` |

Synthetic fixtures cover what production does not show on demand: features without geometry, mixed
geometry types, a feature without `properties.id`, a non-numeric `udatestart`, malformed JSON. The
size-cap test generates its body.

**Integration tests** run the real Express app in-process against `test/fake-tlcmap.ts`, a local
HTTP server serving the fixtures, driven by `@modelcontextprotocol/client` over Streamable HTTP:

- `tools/list` holds `tlcmap_get_layer` and nothing else.
- `tlcmap_get_layer` returns the expected summary for each fixture, with `metadata` deep-equal to
  the fixture's.
- A resource read returns bytes whose `sha256` equals the tool's.
- A second call is a cache hit with the original `fetched_at`.
- Each error path end to end: `200` for a layer other than the one requested, a `3xx`, a `5xx`, a timeout, the size cap,
  `SERVER_BUSY`, `RESOURCE_EXPIRED`, `INVALID_ARGUMENT`.
- `429` past the rate limit; a spoofed `X-Forwarded-For` from a non-loopback peer is ignored.
- `/healthz` shape; `GET /mcp` is `405`.

**Production compatibility check** — `npm run test:prod`, run by hand and before each deploy, never
in CI. It asserts against live tlcmap.org the behaviours A0 depends on:

- The layer feed's top-level shape and `metadata` keys for layer 206.
- `properties.id` present and `udatestart` / `udateend` numeric on layer 1125.
- The missing-layer request producing `UPSTREAM_CONTRACT_VIOLATION`.

That last assertion is **written to fail when §5.1 ② lands**. The fix is to change its expectation
to `LAYER_NOT_FOUND` — the red test is how A0 learns the API changed
([§4.3](../DESIGN.md#43-packaging-and-distribution)).

---

## 13. Acceptance

A0 is done when all of the following hold on `test-mcp.tlcmap.org`, checked by hand and recorded
with the date:

1. CI is green and `npm run test:prod` passes.
2. Claude Code connects with
   `claude mcp add --transport http tlcmap https://test-mcp.tlcmap.org/mcp`, and lists exactly one
   tool.
3. `tlcmap_get_layer` on 206 and on 2749 returns correct summaries with provenance; 2749's warning
   is byte-for-byte the API's, and 206's `"None"` is preserved.
4. Reading the returned resource URI yields the feed, with a `sha256` matching the tool's.
5. `layer_id: "abc"` returns the SDK's input-validation error naming `layer_id`; layer 99999999
   returns `UPSTREAM_CONTRACT_VIOLATION` carrying the API's message.
6. `/healthz` is `200`, and `journalctl -u tlcmap-mcp` shows a `tool` line and an `upstream` line for
   each call.
7. After `systemctl restart tlcmap-mcp`, reading the earlier resource URI returns
   `RESOURCE_EXPIRED`.
8. `deploy.sh` deploys a second release and a rollback to the first restores its version on
   `/healthz`.

---

## 14. API dependencies

What changes in A0's behaviour as [§5](../DESIGN.md#5-layer-1--the-api) items land. None needs
feature detection: each is a fixed expectation updated when the API changes, caught by the
production check.

| Item | Effect on A0 |
| --- | --- |
| §5.1 ② honest errors | `LAYER_NOT_FOUND` and `LAYER_PRIVATE` go live; the missing-layer case stops being `UPSTREAM_CONTRACT_VIOLATION`. The production check fails, and its expectation is updated. |
| §5.2 ⑤ namespaced extended data | Removes the shadowing limitation in §4. The feed's shape changes, so the shape check and fixtures are updated, and resource bytes — served verbatim — change shape for clients. |
| §5.2 ⑦ `nulls` and `include_undated` | Lets `tlcmap_get_layer` gain `sort` and `nulls` as additive parameters ([§6.5](../DESIGN.md#65-tool-surface-stability)). |
| §5.3 ⑭ `ETag` / `Last-Modified` | Allows conditional revalidation in the cache instead of TTL-only expiry. |
| §5.3 ⑰ `warnnig` → `warning` | No code change; `details` passes on whatever the API says. |
