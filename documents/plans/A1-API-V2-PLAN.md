# A1 — API v2: implementation plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development
> (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use
> checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the first four API v2 endpoints — place search, the layer catalogue, layer detail and
paged layer features — in the TLCMap application, delivering DESIGN §5.2 items ① to ⑧, each pinned by
tests, with v1 untouched and the v2 developer docs written.

**Architecture:** New code under `TLCMap\Api\V2` behind its own route file and middleware group at
`/api/v2/`. Requests are parsed into value objects (`Params/`), turned into SQL by one shared
`RecordQuery` plus endpoint queries (`Query/`), and serialised (`Serialise/`). Dates, extents and
summaries are computed at query time by SQL functions added by migration; nothing derived is stored.

**Tech Stack:** Laravel 5.8.38 on PHP 7.4 (code must also run on 8.2), PostgreSQL 14 with PostGIS 3.2
and pg_trgm, PHPUnit 7.5.20; VitePress for the docs.

**Spec:** [A1-API-V2.md](../specs/A1-API-V2.md). Read it first; this plan argues from it and cites it as
"spec §n". Where a task names a behaviour without restating it, the spec section is the requirement.

**Where the work happens:** the TLCMap application at `C:\laragon\www\tlcmap-ghap` (Tasks 1–15, 17),
on a branch `a1-api-v2` cut from `tlcmap-api-enhancement` and merged back into it at the end — never
into `main` or `dev`; and `C:\MyWork\systemik\projects\tlcmap\tlcmap-docs` on branch `api-v2`
(Task 16). The acceptance record (Task 17) goes in this repository. Paths are relative to the
repository the task names.

## Global Constraints

- PHP 7.4 syntax only: no `match`, named arguments, constructor promotion, union types, `mixed`,
  nullsafe `?->`, `str_contains`/`str_starts_with`, or attributes. Typed properties and arrow functions
  are fine.
- Run every test with PHP 7.4, from the application root in Git Bash:
  `PHP74=/c/laragon/bin/php/php-7.4.19-Win32-vc15-x64/php.exe; $PHP74 vendor/bin/phpunit …`.
  The `php` on the path is 8.2 and must not be used for tests.
- Tests run only against `tlcmap_test`, never `temp_tlcmap`. The V2 base test case fails any test
  whose connection is not a database ending in `_test`.
- v1 is not modified: no edits to `routes/web.php`, `routes/api.php`, v1 controllers, `FileFormatter`,
  `GeneralFunctions`, models, existing middleware, or existing migrations. The only shared files
  edited are `app/Exceptions/Handler.php` (one branch), `app/Http/Kernel.php` (one group, two aliases),
  `app/Providers/RouteServiceProvider.php` (one map call) and `phpunit.xml` (suites and env).
- Every table and function is named with its schema: `tlcmap.dataitem`, `tlcmap.v2_udate(…)`. The
  extensions live in `public`; `DB_SCHEMA` stays unset.
- All SQL values are bound, never interpolated.
- Parameter names, error codes, defaults and limits are exactly the spec's: `per_page` 500/5,000 on
  features and 100/1,000 on the catalogue; `sample` ≤ 5,000; `layers` ≤ 100 ids; `polygon` ≤ 1,000
  vertices; strings ≤ 200 characters; `seed` ≤ 64; `extended_data` ≤ 10 clauses; fuzzy threshold 0.3;
  rate limit `60,1`.
- JSON responses: `JSON_UNESCAPED_SLASHES | JSON_UNESCAPED_UNICODE`, not pretty-printed,
  `Content-Type: application/json; charset=utf-8`.
- Australian English in messages, comments and docs (`licence` as a noun in prose; the field stays
  `license`). Never the name "GHAP" in new prose; the production `datasource.name` value is data and
  is seeded as it is.
- Files use LF line endings, UTF-8 without BOM, one trailing newline.
- Commit messages are short imperative sentences, ending with
  `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`.

## Review Focus

Inputs the spec implies but its own tests would not reach. Each has its test in the owning task.

1. **Encoded and non-ASCII values.** `name=Mount+Vincent`, `name=Mount%20Vincent` and a name with
   diacritics or a curly apostrophe (`Ngun’gara`) must decode exactly as PHP decodes `$_GET`; the
   raw-query repeat check must not mis-split them. → Task 6.
2. **Tampered cursors.** A cursor that is valid base64 and valid JSON but carries a string where an
   id belongs, an array as the key, or SQL text, must be a `400 INVALID_PARAMETER` — never a SQL error
   or a `500`. → Task 9.
3. **Awkward sort keys at page boundaries.** Titles differing only in case, identical titles, and null
   catalogue names, split across a page boundary, must page with nothing skipped or repeated. → Task 9
   and Task 15.
4. **Extreme numbers.** `per_page=99999999999999999999`, `limit=-1`, `bbox=1e400,0,1,1`,
   `bbox=NaN,…`, `layers=0`, a 30-digit year in `date_from` — each a `400` naming the parameter, never
   a `500` or an overflow. → Tasks 6, 7.
5. **Awkward extended data.** Two `Data` elements with the same name, entities (`&amp;`), CDATA, and an
   empty `ExtendedData` element: the first value wins in both the filter (SQL) and the output (PHP),
   entities are decoded, and an empty element is `{}` with `extended_data_unreadable: false`.
   → Tasks 4, 12.

---

## File structure

TLCMap application (`tlcmap-ghap`):

```
app/Api/V2/
  Errors/ApiException.php          status, code, message, details; named constructors
  Errors/ErrorRenderer.php         Throwable → spec §11 JSON
  Params/ParameterSet.php          allowed names, unknown/repeated/empty rejection, scalar readers
  Params/Bbox.php  Polygon.php  DateBound.php  Sources.php  LayerIds.php
  Params/ExtendedDataFilter.php    spec §7.8 grammar
  Params/Sort.php  Paging.php  Cursor.php  Sample.php
  Query/RecordFilters.php          the parsed record filters of spec §7, and their echo
  Query/RecordQuery.php            RecordFilters → filtered builder; the date SQL expressions
  Query/LayerGate.php              403/404 for layer ids
  Query/KeysetPager.php            keyset paging over any builder (spec §5.3–5.5)
  Query/Summary.php                spec §8
  Query/PlaceSearch.php            place search: filters + sort/sample + page
  Query/LayerCatalogue.php         spec §10.2
  Serialise/FeatureSerialiser.php  spec §9
  Serialise/LayerSerialiser.php    spec §10.2 entry, §10.3 layer object
  Support/Json.php                 the one JSON response helper
app/Console/Commands/BuildTestDatabase.php    tlcmap:test-db
app/Http/Controllers/Api/V2/PlaceController.php  LayerController.php
app/Http/Middleware/ApiV2Throttle.php  ApiV2Cors.php
config/api_v2.php
database/migrations/2026_11_10_000001_create_v2_date_functions.php
database/migrations/2026_11_10_000002_create_v2_extended_value_function.php
database/migrations/2026_11_10_000003_add_v2_indexes.php
routes/api_v2.php
tests/V2/TestCase.php  tests/V2/Support/Fixtures.php  tests/V2/Support/Civil.php
tests/V2/Unit/…  tests/V2/Sql/…  tests/V2/Feature/…
tests/V1Guard/V1ResponsesTest.php  tests/V1Guard/golden/*.json
```

Modified: `phpunit.xml`, `app/Exceptions/Handler.php`, `app/Http/Kernel.php`,
`app/Providers/RouteServiceProvider.php`, `app/Console/Kernel.php` (only if commands are not
auto-loaded — check first).

`tlcmap-docs`: `docs/developers/v2/{index,places,layers,dates,errors,differences}.md`,
`docs/.vitepress/config.mts`, `docs/developers/index.md`.

This repository: `documents/acceptance/A1-API-V2.md`, `README.md`.

---

### Task 1: Test database and harness

**Files:**
- Create: `app/Console/Commands/BuildTestDatabase.php`, `tests/V2/TestCase.php`,
  `tests/V2/Support/Fixtures.php`, `tests/V2/Support/Civil.php`, `tests/V2/Feature/HarnessTest.php`
- Modify: `phpunit.xml`

**Interfaces:**
- Produces:
  - Artisan `tlcmap:test-db {--database=tlcmap_test}`.
  - `abstract class Tests\V2\TestCase extends Tests\TestCase` using `DatabaseTransactions`; `setUp()`
    fails unless `DB::connection()->getDatabaseName()` ends in `_test`, and resets `Fixtures`.
  - `Tests\V2\Support\Fixtures` (static):
    `layer(array $attrs = []): int` — inserts `tlcmap.dataset`, defaults `public => true`,
    `name => 'Layer N'`, fixed `created_at`/`updated_at` `2026-01-01 00:00:00`;
    `record(array $attrs = []): int` — inserts `tlcmap.dataitem`; defaults `datasource_id` 1 when
    `dataset_id` is given, otherwise 2; `title => 'Place N'`; fixed timestamps; `uid` = prefix + hex id,
    prefix `t` for datasource 1 and 4, `a` for 2, `n` for 3; accepts `lat`/`lon` shorthand;
    `keyword(int $layerId, string $keyword): void`;
    `datasourceId(string $searchParamName): int`;
    ids come from a counter starting at 900001, reset per test, passed explicitly on insert.
  - `Tests\V2\Support\Civil::udate(int $astronomicalYear, int $month = 1, int $day = 1): int` — epoch
    milliseconds, independent of PostgreSQL.

- [ ] **Step 1: Branch**

```bash
cd /c/laragon/www/tlcmap-ghap && git checkout tlcmap-api-enhancement && git pull --ff-only && git checkout -b a1-api-v2
```

- [ ] **Step 2: Add the suites and test env to `phpunit.xml`**

Add `<testsuite name="V2"><directory suffix="Test.php">./tests/V2</directory></testsuite>` and
`<testsuite name="V1Guard"><directory suffix="Test.php">./tests/V1Guard</directory></testsuite>`, and
under `<php>`: `<env name="DB_DATABASE" value="tlcmap_test" force="true"/>` and
`<env name="APP_URL" value="http://localhost" force="true"/>`.

- [ ] **Step 3: Implement `tlcmap:test-db`**

Refuse (exit 1, message naming the database) unless `--database` ends in `_test`. Then:
1. Connect to the `postgres` maintenance database with the configured credentials and `CREATE DATABASE`
   if absent.
2. Point the `pgsql` connection at the target (`config([...database => $name])`, `DB::purge`).
3. `DROP SCHEMA IF EXISTS tlcmap CASCADE; DROP SCHEMA IF EXISTS public CASCADE; CREATE SCHEMA public;
   CREATE SCHEMA tlcmap;` then `CREATE EXTENSION IF NOT EXISTS postgis SCHEMA public` and the same for
   `pg_trgm`.
4. `Artisan::call('migrate', ['--force' => true])`, printing its output.
5. Upsert the datasource rows by `search_param_name` (the `create_text_table` migration already
   inserts the geocoder row):

| id | name | description | search_param_name | link |
| --- | --- | --- | --- | --- |
| 1 | GHAP | Gazetteer of Historical Australian Places | searchpublicdatasets | https://ghap.tlcmap.org/ |
| 2 | ANPS | Australian National Placenames Survey Gazetteer | searchausgaz | https://www.anps.org.au/ |
| 3 | NCG | Composite Gazetteer of Australia | searchncg | https://placenames.fsdf.org.au/ |
| 4 | GEOCODER | *(as the migration inserts it)* | searchgeocoder | *(as inserted)* |

   (Row 1's `name` is the production value — data, not prose.)
6. Insert `recordtype` rows 1–9 with `type`: `Other`, `Placename`, `Media`, `Text`, `Event`,
   `Person`, `Journey`, `Site`, `Organisation` — in that id order (row 4 must be `Text`; v1 relies on
   it), skipping any that exist.

- [ ] **Step 4: Run it and make the migrations run from scratch**

Run: `$PHP74 artisan tlcmap:test-db`
Expected: every migration reports `Migrated:`. If one fails, the spec (§13.1) applies: do not edit the
migration. Make the bootstrap create whatever the migration assumed existed (before step 4 above), and
record each such adjustment in a comment in the command and in this task's commit message.

- [ ] **Step 5: Write `HarnessTest`**

```php
public function test_connection_is_the_test_database()   // getDatabaseName() === 'tlcmap_test'
public function test_fixture_record_gets_geom_from_trigger()
{   $id = Fixtures::record(['lat' => -32.9, 'lon' => 151.1]);
    $row = DB::selectOne('select ST_X(geom) x, ST_Y(geom) y, uid from tlcmap.dataitem where id = ?', [$id]);
    // x == 151.1, y == -32.9, uid === 'a' . dechex($id)   (no dataset → ANPS default)
}
public function test_fixture_ids_restart_each_test()      // first Fixtures::layer() returns 900001
public function test_civil_reference_values()
{   // Civil::udate(1970) === 0; Civil::udate(1856) === -3597523200000;
    // Civil::udate(1856, 3, 4) === -3592080000000; Civil::udate(-399) === -74758377600000;
    // Civil::udate(-51020) === -1672202880000000; Civil::udate(2000, 2, 29) === 951782400000
}
```

`Civil::udate` uses days-from-civil (floor division throughout):

```
y = Y - (M <= 2 ? 1 : 0)
era = floor(y / 400);  yoe = y - era*400
doy = floor((153*(M + (M > 2 ? -3 : 9)) + 2) / 5) + D - 1
doe = yoe*365 + floor(yoe/4) - floor(yoe/100) + doy
days = era*146097 + doe - 719468;   ms = days * 86400000
```

- [ ] **Step 6: Run the harness tests**

Run: `$PHP74 vendor/bin/phpunit --testsuite V2`
Expected: 4 tests pass.

- [ ] **Step 7: Commit**

```bash
git add phpunit.xml app/Console/Commands/BuildTestDatabase.php tests/V2
git commit -m "Build a test database from migrations and add the V2 test harness"
```

---

### Task 2: The v1 guard

Characterisation tests written **before** any v2 migration exists, so they record v1 as it is.

**Files:**
- Create: `tests/V1Guard/V1ResponsesTest.php`, `tests/V1Guard/golden/*.json`

**Interfaces:**
- Consumes: `Tests\V2\TestCase`, `Fixtures` (Task 1).

- [ ] **Step 1: Write `V1ResponsesTest`**

One fixture method builds: two public layers and one private layer; records in them with full
metadata, extended data (including a field named `description`), coordinates, mixed dates (`1856`,
`1856-03-04`, empty); and two ANPS records (no dataset). Five tests, each requesting a v1 URL and
comparing the body with `golden/<name>.json`:

| Test | URL |
| --- | --- |
| `test_places_name_search` | `/places?format=json&name=Fixture%20Place` |
| `test_places_bbox_search` | `/places?format=json&bbox=150,-34,152,-32` |
| `test_places_date_search` | `/places?format=json&datefrom=1850&dateto=1860` |
| `test_layer_feed` | `/layers/{first public layer id}/json` |
| `test_layer_catalogue` | `/layers/json` |

Comparison: decode both, `assertSame` on the arrays after replacing any `metadata.url` value with
`"<url>"` (the request URL is the only expected volatile part; list any other replacement in the test
with its reason). If the golden file is missing and `UPDATE_GOLDEN=1` is set, write it and mark the
test incomplete; otherwise fail.

- [ ] **Step 2: Generate the golden files**

Run: `UPDATE_GOLDEN=1 $PHP74 vendor/bin/phpunit --testsuite V1Guard`
Expected: 5 incomplete, golden files written. Read each one: it must contain the fixture data (e.g.
the layer feed's features, `warnnig` absent because the layer is public). A golden file of an error
page is a broken fixture, not a baseline.

- [ ] **Step 3: Run against the golden files**

Run: `$PHP74 vendor/bin/phpunit --testsuite V1Guard`
Expected: 5 pass. Run it twice more: still 5 pass (repeatable).

- [ ] **Step 4: Commit**

```bash
git add tests/V1Guard
git commit -m "Characterise v1 search, layer feed and catalogue responses"
```

From here on, every task's final run includes `--testsuite V1Guard` and expects it unchanged.

---

### Task 3: The date functions

**Files:**
- Create: `database/migrations/2026_11_10_000001_create_v2_date_functions.php`,
  `tests/V2/Sql/DateFunctionsTest.php`

**Interfaces:**
- Produces: `tlcmap.v2_udate(text) RETURNS bigint` and `tlcmap.v2_udate_end(text) RETURNS bigint`,
  both `LANGUAGE plpgsql IMMUTABLE STRICT PARALLEL SAFE` (spec §4, §6.1–6.2).

- [ ] **Step 1: Write `DateFunctionsTest`**

A data provider of `[input, expectedStart, expectedEnd]`, expected values built with `Civil::udate`
(never by calling the function under test), asserted with
`DB::selectOne('select tlcmap.v2_udate(?) s, tlcmap.v2_udate_end(?) e', [$in, $in])`:

| Input | start | end |
| --- | --- | --- |
| `1970` | `Civil::udate(1970)` | `Civil::udate(1971) - 1` |
| `1856` | `Civil::udate(1856)` | `Civil::udate(1857) - 1` |
| `400` | `Civil::udate(400)` | `Civil::udate(401) - 1` |
| `0001` | `Civil::udate(1)` | `Civil::udate(2) - 1` |
| `-1` | `Civil::udate(0)` | `Civil::udate(1) - 1` |
| `-400` | `Civil::udate(-399)` | `Civil::udate(-398) - 1` |
| `-51021` | `Civil::udate(-51020)` | `Civil::udate(-51019) - 1` |
| `1856-03` | `Civil::udate(1856, 3)` | `Civil::udate(1856, 4) - 1` |
| `1856-12` | `Civil::udate(1856, 12)` | `Civil::udate(1857) - 1` |
| `-400-02` | `Civil::udate(-399, 2)` | `Civil::udate(-399, 3) - 1` |
| `1856-03-04` | `Civil::udate(1856, 3, 4)` | `Civil::udate(1856, 3, 5) - 1` |
| `2000-02-29` | `Civil::udate(2000, 2, 29)` | `Civil::udate(2000, 3, 1) - 1` |
| `4/3/1856` and `04/03/1856` | `Civil::udate(1856, 3, 4)` | `Civil::udate(1856, 3, 5) - 1` |
| `1856-00-00`, `00/00/1856` | `Civil::udate(1856)` | `Civil::udate(1857) - 1` |
| `1856-03-04T14:30` | `Civil::udate(1856,3,4) + 52200000` | same as start |
| `1856-03-04T14:30:15.1239` | `+ 52215123` | same as start |
| `  1856  ` | as `1856` | as `1856` |

and a provider of inputs returning `NULL` for both: `''`, `'   '`, `'0'`, `'-0'`, `'0000'`,
`'1856-13'`, `'1856-04-31'`, `'1900-02-29'`, `'-16112-11-31'`, `'1856-03-04T24:00'`,
`'1856-03-04T12:60'`, `'13/13/1856'`, `'1856/03/04'`, `'about 1856'`, `'c.1856'`,
`'99999999999999999999999999999'` (30 digits), `'300000'` (beyond the safe-integer range). Plus:
`select tlcmap.v2_udate(null)` is null; the function is `IMMUTABLE` per `pg_proc.provolatile = 'i'`.

- [ ] **Step 2: Run to verify it fails**

Run: `$PHP74 vendor/bin/phpunit --testsuite V2 --filter DateFunctionsTest`
Expected: FAIL — function `tlcmap.v2_udate(unknown)` does not exist.

- [ ] **Step 3: Write the migration**

`up()` creates both functions with `DB::unprepared`; `down()` drops them. The body:
- `btrim`, then match in order: `^(-?)(\d+)-00-00$`, `^00/00/(\d+)$`, `^(\d{1,2})/(\d{1,2})/(\d{4})$`,
  `^(-?)(\d+)(?:-(\d{2})(?:-(\d{2})(?:T(\d{2}):(\d{2})(?::(\d{2})(?:\.(\d+))?)?)?)?)?$`; anything else
  → `NULL`.
- Reject a year of more than 7 digits before casting (overflow guard), year 0, and `-0`. BCE year Y →
  astronomical `1 - Y`.
- Validate month 1–12, day against the month length with Gregorian leap years on the astronomical
  year, hour 0–23, minute and second 0–59; fraction → first three digits, right-padded with zeros.
- Days-from-civil exactly as `Civil` (Task 1), with floor division written for negative years
  (`floor(y::numeric / 400)`), in `bigint`.
- `v2_udate_end`: for a value with a time, the same as `v2_udate`; otherwise the first instant of the
  next year / month / day (by the precision given) minus 1. Implement once as an internal
  `tlcmap.v2_udate_parts(text, boolean is_end)` called by both, or as two functions sharing the
  parse — the implementer's choice, but one parse, not two copies.
- Return `NULL` if the absolute result exceeds 9007199254740991.

- [ ] **Step 4: Run to verify it passes**

Run: `$PHP74 artisan tlcmap:test-db && $PHP74 vendor/bin/phpunit --testsuite V2 --filter DateFunctionsTest`
Expected: PASS.

- [ ] **Step 5: Run the guard**

Run: `$PHP74 vendor/bin/phpunit --testsuite V1Guard`
Expected: 5 pass.

- [ ] **Step 6: Commit**

```bash
git add database/migrations/2026_11_10_000001_create_v2_date_functions.php tests/V2/Sql/DateFunctionsTest.php
git commit -m "Add v2 date functions with BCE and deep-time support"
```

---

### Task 4: The extended-value function and the indexes

**Files:**
- Create: `database/migrations/2026_11_10_000002_create_v2_extended_value_function.php`,
  `database/migrations/2026_11_10_000003_add_v2_indexes.php`,
  `tests/V2/Sql/ExtendedValueFunctionTest.php`, `tests/V2/Sql/IndexesTest.php`

**Interfaces:**
- Produces: `tlcmap.v2_extended_value(xml_text text, field text) RETURNS text`,
  `LANGUAGE sql IMMUTABLE PARALLEL SAFE`; indexes `v2_dataitem_geom_gist`,
  `v2_dataitem_title_trgm`, `v2_dataitem_placename_trgm`.

- [ ] **Step 1: Write the tests**

`ExtendedValueFunctionTest`, asserting `select tlcmap.v2_extended_value(?, ?)`:
- `<ExtendedData><Data name="Years"><value><![CDATA[52]]></value></Data></ExtendedData>`, `Years` →
  `'52'`; `years` → `NULL` (names are case-sensitive).
- A field name containing `"` and `'` → its value (no XPath literal is built from the name).
- Two `Data name="A"` elements, values `1` then `2` → `'1'` (first wins — Review Focus 5).
- `<value>Fish &amp; Chips</value>` → `'Fish & Chips'`.
- Absent field, `NULL` text, `''`, `'   '`, `<ExtendedData/>` → `NULL`.
- Malformed XML (`<ExtendedData><Data name="A">`) → `NULL`, no error raised.

`IndexesTest`: each of the three index names exists in `pg_indexes` for `tlcmap.dataitem`, with
`gist` / `gin` and `gin_trgm_ops` in its definition.

- [ ] **Step 2: Run to verify they fail**

Run: `$PHP74 vendor/bin/phpunit --testsuite V2 --filter 'ExtendedValueFunctionTest|IndexesTest'`
Expected: FAIL — function does not exist; indexes absent.

- [ ] **Step 3: Write the migrations**

`v2_extended_value`: `NULL` when the text is null, blank, or
`NOT xml_is_well_formed_document(xml_text)` (no exception block — it would open a subtransaction per
row); otherwise the `string(value)` of the first element of `xpath('//Data', xml_text::xml)` whose
`@name` equals `field`, compared as text — via `unnest(xpath(...)) WITH ORDINALITY`, ordered, limit 1.

Indexes: `CREATE INDEX IF NOT EXISTS` for the three (GIST on `geom`; GIN `gin_trgm_ops` on `title` and
on `placename`). `down()` drops them `IF EXISTS`. Laravel 5.8 wraps PostgreSQL migrations in a
transaction, so do not use `CONCURRENTLY`.

- [ ] **Step 4: Run to verify they pass**

Run: `$PHP74 artisan tlcmap:test-db && $PHP74 vendor/bin/phpunit --testsuite V2 && $PHP74 vendor/bin/phpunit --testsuite V1Guard`
Expected: all pass.

- [ ] **Step 5: Commit**

```bash
git add database/migrations/2026_11_10_00000[23]_*.php tests/V2/Sql
git commit -m "Add the v2 extended-value function and spatial and trigram indexes"
```

---

### Task 5: The v2 skeleton — routing, errors, throttle, CORS

**Files:**
- Create: `app/Api/V2/Errors/ApiException.php`, `app/Api/V2/Errors/ErrorRenderer.php`,
  `app/Api/V2/Support/Json.php`, `app/Http/Middleware/ApiV2Throttle.php`,
  `app/Http/Middleware/ApiV2Cors.php`, `config/api_v2.php`, `routes/api_v2.php`,
  `tests/V2/Feature/SkeletonTest.php`
- Modify: `app/Providers/RouteServiceProvider.php`, `app/Http/Kernel.php`,
  `app/Exceptions/Handler.php`

**Interfaces:**
- Produces:
  - `ApiException extends \RuntimeException`:
    `__construct(int $status, string $code, string $message, array $details = [])`,
    `status(): int`, `errorCode(): string`, `details(): array`; named constructors
    `invalidParameter(string $parameter, string $rule, string $accepts, string $message): self`,
    `invalidGeometry(string $parameter, string $reason): self`,
    `filterRejected(string $expression, int $position, string $expected): self`,
    `layerPrivate(int $layerId): self`, `layerNotFound(int $layerId): self`.
  - `ErrorRenderer::render(Request $request, \Exception $e): JsonResponse`.
  - `Json::response(array $body, int $status = 200, array $headers = []): JsonResponse` — the flags
    and content type of the Global Constraints. Every v2 response goes through it.
  - `config('api_v2')` keys: `rate_limit` (`env('API_V2_RATE_LIMIT', '60,1')`),
    `features_per_page` 500, `features_per_page_max` 5000, `layers_per_page` 100,
    `layers_per_page_max` 1000, `sample_max` 5000, `fuzzy_threshold` 0.3, `polygon_max_vertices` 1000,
    `layers_max` 100, `string_max` 200, `seed_max` 64, `extended_data_max_clauses` 10.
  - Middleware group `api_v2` = `[ApiV2Cors::class, 'throttle.v2', 'bindings']`; alias `throttle.v2`.
  - `routes/api_v2.php` holds the four GET routes of spec §3.1 (controllers arrive in Tasks 13–15;
    until then register only what exists) and `Route::options('{any?}', …)->where('any', '.*')`
    returning `204`.

- [ ] **Step 1: Write `SkeletonTest`**

In `setUp`, register two test routes inside the `api_v2` group:
`api/v2/__test/ok` (returns `Json::response(['ok' => true])`) and `api/v2/__test/boom` (throws
`\RuntimeException('secret detail')`).

```php
test_unknown_v2_route_is_json_404        // GET /api/v2/nope → 404, error.code NOT_FOUND, details.path 'api/v2/nope'
test_post_is_json_405                    // POST /api/v2/__test/ok → 405, METHOD_NOT_ALLOWED, details.method 'POST'
test_options_preflight                   // OPTIONS /api/v2/__test/ok → 204, Access-Control-Allow-Origin '*',
                                         //   Allow-Methods 'GET, OPTIONS', no Access-Control-Allow-Credentials
test_cors_on_errors                      // the 404 above carries Access-Control-Allow-Origin '*'
test_unexpected_error_is_opaque_500      // GET boom → 500, INTERNAL_ERROR, body does not contain 'secret detail'
test_api_exception_rendered              // a route throwing ApiException::layerPrivate(7) → 403,
                                         //   {"error":{"code":"LAYER_PRIVATE","message":…,"details":{"layer_id":7}}}
test_rate_limited                        // config(['api_v2.rate_limit' => '2,1']); third GET → 429, RATE_LIMITED,
                                         //   Retry-After header, details.limit 2
test_v1_requests_do_not_count_against_v2 // limit '2,1'; GET /layers/json ×3 (v1), then GET ok → 200
test_v2_requests_do_not_count_against_v1 // GET ok ×45, then GET /layers/json → not 429
test_non_v2_paths_untouched              // GET /nope (web) → not JSON content type (the HTML 404 as before)
test_response_flags                      // ok body is exactly '{"ok":true}', content type application/json; charset=utf-8
```

- [ ] **Step 2: Run to verify it fails**

Run: `$PHP74 vendor/bin/phpunit --testsuite V2 --filter SkeletonTest`
Expected: FAIL — the `api_v2` middleware group is not defined.

- [ ] **Step 3: Implement**

- `ApiV2Throttle extends ThrottleRequests`: `handle($request, Closure $next, $maxAttempts = null,
  $decayMinutes = null)` reads `config('api_v2.rate_limit')` (`"max,minutes"`) when not given;
  `resolveRequestSignature()` returns `'api-v2|' . parent::resolveRequestSignature($request)`.
- `ApiV2Cors`: sets the three headers of spec §12 on every response, including error responses
  (apply in a `try/finally`-safe way: headers on the response returned by `$next`, and the renderer's
  responses also get them — have `ErrorRenderer` call the same header method).
- `ErrorRenderer` mapping, in order: `ApiException` → its own; `ThrottleRequestsException` → 429
  `RATE_LIMITED`, keeping its `Retry-After` header, `details: {limit, retry_after}`;
  `MethodNotAllowedHttpException` → 405; `NotFoundHttpException` → 404 `NOT_FOUND`
  `details.path = $request->path()`; anything else → 500 `INTERNAL_ERROR`, message
  `"An unexpected error occurred."`, `details: {}` (the Handler still reports it).
- `Handler::render`: the single branch of spec §3.3, first; add `ApiException::class` to `$dontReport`.
- `RouteServiceProvider::map()`: add `mapApiV2Routes()` per spec §3.1, called before `mapWebRoutes()`.

- [ ] **Step 4: Run to verify it passes**

Run: `$PHP74 vendor/bin/phpunit --testsuite V2 && $PHP74 vendor/bin/phpunit --testsuite V1Guard`
Expected: all pass.

- [ ] **Step 5: Commit**

```bash
git add app/Api app/Http/Middleware/ApiV2*.php config/api_v2.php routes/api_v2.php app/Providers/RouteServiceProvider.php app/Http/Kernel.php app/Exceptions/Handler.php tests/V2/Feature/SkeletonTest.php
git commit -m "Add the v2 route group with JSON errors, its own throttle key and CORS"
```

---

### Task 6: Parameter set and scalar parsing

**Files:**
- Create: `app/Api/V2/Params/ParameterSet.php`, `tests/V2/Unit/ParameterSetTest.php`

**Interfaces:**
- Produces `ParameterSet`:
  - `static fromRequest(Request $request, array $allowed): self` — throws `invalidParameter` for an
    unknown name (`rule: unknown`, `accepts`: the allowed names, comma-joined), a name appearing more
    than once in the raw query string or as an array (`rule: repeated`), or a present key whose value
    is empty (`rule: empty`).
  - `has(string $name): bool`; `raw(string $name): ?string`;
    `string(string $name, ?int $maxLength = null): ?string` (default max `config('api_v2.string_max')`);
    `bool(string $name, bool $default): bool` (`true`/`false` only);
    `int(string $name, int $min, int $max, ?int $default): ?int` (digits with optional leading `-`,
    no `+`, no decimals, range checked before cast — a 20-digit value is out of range, not an
    overflow);
    `enum(string $name, array $values, ?string $default): ?string`;
    `all(): array` — every present parameter, raw strings, for the cursor hash.

- [ ] **Step 1: Write `ParameterSetTest`**

Build requests with `Request::create('/api/v2/places?' . $qs)`:
- unknown `date_form` → `INVALID_PARAMETER`, `details.parameter 'date_form'`, `rule 'unknown'`.
- `sources=anps&sources=ncg` and `layers[]=1` → `rule 'repeated'`.
- `bbox=` → `rule 'empty'`.
- `name=Mount+Vincent`, `name=Mount%20Vincent` → `string('name') === 'Mount Vincent'`;
  `name=Ngun%E2%80%99gara` → `'Ngun’gara'`; `name=a%26b` → `'a&b'` and not reported as repeated
  (Review Focus 1).
- `string()` over 200 characters → `rule 'max_length'`.
- `bool`: `true`/`false` parse; `1`, `yes`, `TRUE` → `rule 'boolean'`.
- `int`: `per_page=99999999999999999999` → `rule 'range'`, `accepts '1–5000'` when called with
  `(1, 5000)`; `limit=-1` → range; `limit=1.5`, `limit=+3`, `limit=1e3` → `rule 'integer'`
  (Review Focus 4).
- `enum`: `match=Fuzzy` → `rule 'one_of'`, `accepts 'exact, contains, fuzzy'` (values are
  case-sensitive).

- [ ] **Step 2: Run to verify it fails** — `--filter ParameterSetTest`, FAIL (class missing).

- [ ] **Step 3: Implement `ParameterSet`**

Count names from `$request->server('QUERY_STRING')`: split on `&`, take the part before `=`, decode it
the way PHP does (`urldecode`), strip a trailing `[...]`, and count. PHP turns `.` and space in names
into `_`; apply the same to the counted names so they agree with `$request->query`.

- [ ] **Step 4: Run to verify it passes**, then the guard. Expected: all pass.

- [ ] **Step 5: Commit** — `git commit -m "Parse v2 parameters strictly"`

---

### Task 7: Value parsers — bbox, polygon, dates, sources, layer ids

**Files:**
- Create: `app/Api/V2/Params/{Bbox,Polygon,DateBound,Sources,LayerIds}.php`,
  `tests/V2/Unit/ValueParsersTest.php`

**Interfaces:**
- Consumes: `ApiException`, `tlcmap.v2_udate` / `v2_udate_end` (Task 3).
- Produces:
  - `Bbox::parse(string $value): Bbox` — public `float $minLon, $minLat, $maxLon, $maxLat`;
    `crossesAntimeridian(): bool`; `toArray(): array` (`[minLon, minLat, maxLon, maxLat]`).
  - `Polygon::parse(string $value, int $maxVertices): Polygon` — `vertices(): array` (closed ring,
    `[[lon, lat], …]`), `toArray(): array` (as given, for the echo).
  - `DateBound::parse(string $parameter, string $value): DateBound` — `string $text`, `int $start`,
    `int $end` (from one `select tlcmap.v2_udate(?), tlcmap.v2_udate_end(?)`).
  - `Sources::ALL = ['layers', 'anps', 'ncg', 'text']`; `Sources::PARAM_NAMES` mapping each to its
    `search_param_name`; `Sources::parse(string $value): array` (unique, in `ALL` order);
    `Sources::idsByKey(): array` (`['layers' => 1, …]`, loaded once per request from
    `tlcmap.datasource`); `Sources::keyById(): array` (the inverse).
  - `LayerIds::parse(string $value, int $max): int[]` (unique, in the order given).

- [ ] **Step 1: Write `ValueParsersTest`**

- Bbox: `150.8,-33.1,151.4,-32.6` parses; `170,-10,-170,10` → `crossesAntimeridian()`; three
  numbers, a letter, `1e400,0,1,1`, `NaN,0,1,1`, `INF,0,1,1`, longitude 181, latitude −91,
  `minLat > maxLat` → each `INVALID_PARAMETER` on `bbox` (Review Focus 4).
- Polygon: `151 -33,151.5 -33,151.5 -32.5` → four vertices, last equals first; an already-closed ring
  is not closed twice; two distinct vertices, a vertex with one number, 1,001 vertices,
  longitude 200 → `INVALID_PARAMETER` on `polygon`.
- DateBound (DB): `date_from=1820` → `start === Civil::udate(1820)`, `end === Civil::udate(1821) - 1`;
  `c.1820` and a 30-digit year → `INVALID_PARAMETER` naming the parameter given.
- Sources: `anps,ncg,anps` → `['anps', 'ncg']`; `ncg,layers` → `['layers', 'ncg']`; `gazetteer` and
  `,` → `INVALID_PARAMETER` on `sources`. `idsByKey()` on the test database →
  `['layers' => 1, 'anps' => 2, 'ncg' => 3, 'text' => 4]`.
- LayerIds: `5,3,5` → `[5, 3]`; `0`, `-1`, `abc`, `1.5`, and 101 ids → `INVALID_PARAMETER` on
  `layers`.

- [ ] **Step 2: Run to verify it fails.** **Step 3: Implement.** Numbers in `bbox`/`polygon` match
  `^-?\d+(\.\d+)?$` before conversion, which excludes exponents, `NaN` and `INF` by construction.
- [ ] **Step 4: Run to verify it passes**, then the guard.
- [ ] **Step 5: Commit** — `git commit -m "Parse v2 bbox, polygon, dates, sources and layer ids"`

---

### Task 8: The extended-data grammar

**Files:**
- Create: `app/Api/V2/Params/ExtendedDataFilter.php`, `tests/V2/Unit/ExtendedDataFilterTest.php`

**Interfaces:**
- Consumes: `DateBound` (for `before`/`after` operands — validated with the same function).
- Produces: `ExtendedDataFilter::parse(string $expression, int $maxClauses): ExtendedDataFilter`;
  `clauses(): array` of `['field' => string, 'op' => '='|'>'|'<'|'textmatch'|'before'|'after',
  'value' => string, 'date' => ?DateBound]` (`date` set for `before`/`after`); `toEcho(): string`
  (the expression as given).

- [ ] **Step 1: Write `ExtendedDataFilterTest`**

Parses (assert the clause arrays):
- `Years>50`, `Years > 50`, `"Years" > "50"`, `'Years'>'50'` → one clause `Years`, `>`, `50`.
- `Catalogue no = 3` → field `Catalogue no`.
- `Name textmatch bridge AND Years < 10` → two clauses; `TEXTMATCH` and `TextMatch` normalise to
  `textmatch`.
- `Title = "Bread AND Butter"` → value `Bread AND Butter`.
- `Built after 1850-03` → `op 'after'`, `date->start === Civil::udate(1850, 3)`.
- `Sandy textmatch sand` → field `Sandy` (word operators need whitespace; `and` inside a word is not
  `AND`).
- `Years > -2.5` parses.

Rejections — each `FILTER_REJECTED` with `details.expression` as given and the exact `position`
(0-based character offset where parsing failed) and `expected`:
- `Years` → position 5, expected `an operator`.
- `= 5` → position 0, expected `a field name`.
- `Years >` → position 7, expected `a value`.
- `Years > many` → position 8, expected `a number`.
- `Built before soon` → position 13, expected `a date`.
- `Years > 50 AND` → position 14, expected `a clause`.
- `"Years > 50` → position 0, expected `a closing quote`.
- `Years > 50 and Size < 3` → position 8, expected `a number`: lower-case `and` is not a separator,
  so the value is `50 and Size < 3`, which is not a number.
- 11 clauses → position of the 11th clause, expected `at most 10 clauses`.

- [ ] **Step 2: Run to verify it fails.**
- [ ] **Step 3: Implement** — a hand-written scanner over the string (no `preg_split` on the whole
  expression), tracking the position for every error. Split clauses on `AND` only where it is a
  whole upper-case word bounded by whitespace and outside quotes.
- [ ] **Step 4: Run to verify it passes**, then the guard.
- [ ] **Step 5: Commit** — `git commit -m "Parse extended-data filters with a strict grammar"`

---

### Task 9: Sort, paging, cursor, sample and the keyset pager

**Files:**
- Create: `app/Api/V2/Params/{Sort,Paging,Cursor,Sample}.php`, `app/Api/V2/Query/KeysetPager.php`,
  `tests/V2/Unit/CursorTest.php`, `tests/V2/Feature/KeysetPagerTest.php`

**Interfaces:**
- Consumes: `ParameterSet` (Task 6).
- Produces:
  - `Sort::parse(ParameterSet $p, array $keys, string $default): Sort` — `string $key`,
    `string $direction` (`asc`|`desc`), `string $nulls` (`first`|`last`); `toEcho(): array`.
  - `Paging::parse(ParameterSet $p, int $default, int $max): Paging` — `int $perPage`,
    `?string $cursor`, `?int $limit` (min 1, max `PHP_INT_MAX`-safe via `ParameterSet::int`).
  - `Cursor::queryHash(array $params): string` — first 16 hex characters of `sha256` of the JSON of
    `$params` without `cursor` and `per_page`, keys sorted.
  - `Cursor::encode(string $hash, $lastKey, int $lastId, int $served): string` — base64url (no
    padding) of `{"v":1,"h":…,"k":lastKey,"i":lastId,"n":served}`.
  - `Cursor::decode(string $value, string $expectedHash, string $keyType): Cursor` — `$keyType` is
    `int`, `string` or `nullable_int`/`nullable_string`; `lastKey`, `lastId`, `served`. Any decoding,
    version, type or hash mismatch → `invalidParameter('cursor', …)`.
  - `Sample::parse(ParameterSet $p, int $max): ?Sample` — `int $size`, `string $seed` (generated
    `bin2hex(random_bytes(8))` when absent); rejects combination with `sort`, `direction`, `nulls`,
    `limit`, `cursor` (`rule: conflicts`).
  - `KeysetPager::page(Builder $query, string $keyExpr, string $keyType, Sort $sort, Paging $paging,
    string $hash, string $idExpr): array` → `['rows' => array, 'next_cursor' => ?string,
    'per_page' => int]`. `$query` already has its filters and select; the pager adds the order, the
    keyset predicate, the limit, and selects `$keyExpr as _k` and `$idExpr as _id`.

- [ ] **Step 1: Write `CursorTest`** (unit)

Round trip for an int key, a string key with a quote and Unicode, and a null key. Rejections, each
`INVALID_PARAMETER` on `cursor` (Review Focus 2): not base64; base64 of non-JSON; `v: 2`; wrong hash;
`i: "1; drop table x"`; `k: [1]` for an int key; `k: "abc"` for an int key; `n: -1`.

- [ ] **Step 2: Write `KeysetPagerTest`** (feature, over `tlcmap.dataitem` fixtures)

A fixture of 23 records whose titles include `alpha`, `Alpha`, `ALPHA`, three identical `Beta`, and
whose sort-date keys include nulls (Review Focus 3). For each combination of key ∈ {id, title, a
nullable integer column}, direction ∈ {asc, desc}, nulls ∈ {first, last}, with `per_page = 4`:
- following `next_cursor` to the end visits every fixture id exactly once;
- the concatenated order equals the order of one unpaged `orderBy` query with the same terms.

Also:
- `limit = 10`, `per_page = 4` → pages of 4, 4, 2, then `next_cursor` null.
- Deleting the last-served row between pages: the next page continues with the following row;
  nothing repeats.
- Inserting a row before the cursor position between pages: it is not served; nothing else changes.

- [ ] **Step 3: Run to verify they fail.**

- [ ] **Step 4: Implement**

Order: `({key} IS NULL) {nulls last ? 'ASC' : 'DESC'}, {key} {dir}, {id} {dir}`. Fetch
`min(perPage, limit − served) + 1` rows; the extra row means there is a next page. After a cursor
`(K, I)`, with `>` for `asc` and `<` for `desc`:

| last key K | nulls | predicate |
| --- | --- | --- |
| not null | last | `({key} IS NOT NULL AND ({key}, {id}) > (K, I)) OR {key} IS NULL` |
| not null | first | `{key} IS NOT NULL AND ({key}, {id}) > (K, I)` |
| null | last | `{key} IS NULL AND {id} > I` |
| null | first | `({key} IS NULL AND {id} > I) OR {key} IS NOT NULL` |

- [ ] **Step 5: Run to verify they pass**, then the guard.
- [ ] **Step 6: Commit** — `git commit -m "Add v2 keyset paging with stable cursors"`

---

### Task 10: Record filters and the record query

**Files:**
- Create: `app/Api/V2/Query/{RecordFilters,RecordQuery,LayerGate}.php`,
  `tests/V2/Feature/RecordQueryTest.php`

**Interfaces:**
- Consumes: Tasks 6–8 value objects; the SQL functions of Tasks 3–4.
- Produces:
  - `RecordFilters::fromParams(ParameterSet $p): RecordFilters` — reads whichever of `name`, `match`,
    `sources`, `layers`, `bbox`, `polygon`, `date_from`, `date_to`, `include_undated`, `state`, `lga`,
    `feature_term`, `record_type`, `extended_data` are present (the endpoint's allowed list decides
    which can be); calls `LayerGate::assertPublic`; `match` without `name` → `rule: requires`;
    `date_from` after `date_to` → `rule: order`. Public read-only properties for each; `toEcho(): array`
    (spec §5.2 — every one of these names, effective value or `null`; `match` `'exact'` only when
    `name` is given, else `null`; `sources` the array; `include_undated` the boolean);
    `hasRecordLevelFilter(): bool` (any of `bbox`, `polygon`, `date_from`, `date_to`).
  - `LayerGate::assertPublic(array $ids): void` — one query; first offending id in the given order →
    `layerNotFound` or `layerPrivate`.
  - `RecordQuery::build(RecordFilters $f): Builder` — `from tlcmap.dataitem as i`,
    `left join tlcmap.dataset as d on d.id = i.dataset_id`,
    `left join tlcmap.recordtype as rt on rt.id = i.recordtype_id`, visibility
    (`i.dataset_id is null or d.public = true`), and every filter. Selects nothing; callers select.
  - `RecordQuery` SQL fragment constants, used by every caller:
    `S = 'tlcmap.v2_udate(i.datestart)'`, `E = 'tlcmap.v2_udate(i.dateend)'`,
    `SE = 'tlcmap.v2_udate_end(i.datestart)'`, `EE = 'tlcmap.v2_udate_end(i.dateend)'`,
    `LO`, `HI` (spec §6.2), `START_KEY = coalesce(S, E)`, `END_KEY = coalesce(E, S)`,
    `NO_DATE`, `UNPARSEABLE_SIDE` (boolean expressions, spec §6.3).
  - `RecordQuery::prepareFuzzy(RecordFilters $f): void` — when `match` is `fuzzy`, runs
    `select set_config('pg_trgm.similarity_threshold', ?, true)` with the configured threshold. Callers
    run it inside `DB::transaction`.

- [ ] **Step 1: Write `RecordQueryTest`**

Each test builds fixtures, runs `RecordQuery::build(...)->pluck('i.id')` and asserts the exact id set.
One test per row:

| Filter | Pinned |
| --- | --- |
| visibility | records of a private layer never appear, with or without `layers`; ANPS records do |
| `sources=anps` / `layers` / `text` | only those datasources |
| `name=Wollombi&match=exact` | matches title `WOLLOMBI` and a record whose `placename` (not title) is `Wollombi`; not `Wollombi Brook` |
| `name=50%&match=exact` | matches title `50%` only, not `500` (no wildcard) |
| `match=contains` | `Wollombi Brook` matches `name=bi br` |
| `match=fuzzy` | `Wolombi` finds `Wollombi`; `Sydney` does not; result identical after `set_config('pg_trgm.similarity_threshold','0.9',false)` then `prepareFuzzy` |
| `layers=private` / `layers=missing` | `LAYER_PRIVATE` / `LAYER_NOT_FOUND` with that `layer_id`; with `layers=missing,private` the first is reported |
| `bbox` | a record on the boundary matches; one outside does not; a record with no coordinates never matches |
| `bbox` across the antimeridian | `179,-20,-179,-10` matches longitudes 179.5 and −179.5 and not 0 |
| `polygon` | a triangle matches inside points only; a self-intersecting bow-tie → `INVALID_GEOMETRY` with a `reason` containing `Self-intersection` |
| `date_from=1820&date_to=1860` | keeps `1860-06`, `1820`, `1700`–`1830`, and undated records; drops `1861`, `1819-12-31` |
| `include_undated=false` | the same minus undated and fully unparseable records |
| reversed record (`datestart 1870`, `dateend 1850`) | matches `date_from=1855&date_to=1856` |
| half-unparseable (`datestart 1850`, `dateend junk`) | dated from 1850: matches `date_to=1850`, not `date_from=1851` |
| `lga=cessnock` | matches stored `CESSNOCK`; `lga=Cessnock Council` matches nothing |
| `feature_term`, `state`, `record_type` | exact, case-insensitive; `record_type=site` matches `Site` |
| `extended_data=Years > 50` | numeric compare; stored `abc` and absent field do not match; malformed stored XML does not match and raises nothing |
| `extended_data=Built before 1850` | stored `1849` matches; `1850-06` does not (period ends after 1850 starts); `1849-12` matches |
| `extended_data=Name textmatch BRID` | case-insensitive contains |
| `extended_data=A = x` on duplicate `A` fields (`x`, `y`) | matches (first wins, Review Focus 5); `A = y` does not |
| several filters | combined with AND |

- [ ] **Step 2: Run to verify it fails.**
- [ ] **Step 3: Implement.** Write `exact` and `contains` as `ILIKE ? ESCAPE '\'` with `%`, `_`, `\`
  escaped, and `fuzzy` as `(i.title % ? OR i.placename % ?)`, so the trigram indexes serve all three.
  The polygon validity check is one query (`select ST_IsValid(g), ST_IsValidReason(g)` over the bound
  ring) before the filter is added.
- [ ] **Step 4: Run to verify it passes**, then the guard.
- [ ] **Step 5: Commit** — `git commit -m "Add the shared v2 record filters"`

---

### Task 11: The summary

**Files:**
- Create: `app/Api/V2/Query/Summary.php`, `tests/V2/Feature/SummaryTest.php`

**Interfaces:**
- Consumes: `RecordQuery::build` and its constants (Task 10), `Sources::keyById` (Task 7).
- Produces: `Summary::compute(Builder $filtered, bool $withByLayer): array` — exactly the shape of
  spec §8; `by_layer` omitted when `$withByLayer` is false. `$filtered` is an unselected
  `RecordQuery` builder; `Summary` clones it.

- [ ] **Step 1: Write `SummaryTest`**

A fixture of 12 records across three layers (one with `warning` `"None"`, one with a null warning,
one with an advisory text) plus three ANPS records: some without coordinates, `no_date`, fully
unparseable, half-unparseable, and dated across 1820–1860 — the earliest `1820`, the latest `1860`,
both at year precision. Expected values computed by hand in the
test from the fixture definitions:
- `total`, `extent` (exact floats), `no_coordinates_count`;
- `date_range.start === Civil::udate(1820)` and `end === Civil::udate(1861) - 1`;
- `undated` `{total, no_date, unparseable_date}` and `unparseable_dates` counting the half-unparseable
  record too;
- `by_source` with all four keys, zeros included;
- `by_layer` ordered by `layer_id`, the `"None"` warning returned as `"None"`, the null as `null`, a
  whitespace-only warning as `null`;
- a filtered set with no coordinates → `extent: null`; with no dated record → `date_range: null`;
- an empty match set → `total 0`, `by_layer []`.

- [ ] **Step 2: Run to verify it fails.** **Step 3: Implement** — two queries over the filtered set:
  one aggregate (`count(*)`, `ST_Extent(i.geom)` read back with `ST_XMin` … `ST_YMax`,
  `count(*) filter (where …)` for each count, `min(LO)`, `max(HI)`, per-datasource counts), and one
  `group by i.dataset_id` joined to the dataset for `name` and `warning`.
- [ ] **Step 4: Run to verify it passes**, then the guard.
- [ ] **Step 5: Commit** — `git commit -m "Compute the v2 summary over the whole match set"`

---

### Task 12: The feature serialiser

**Files:**
- Create: `app/Api/V2/Serialise/FeatureSerialiser.php`, `tests/V2/Unit/FeatureSerialiserTest.php`

**Interfaces:**
- Consumes: `RecordQuery` constants, `Sources::keyById`.
- Produces:
  - `FeatureSerialiser::COLUMNS` — the select list (`i.id`, `i.uid`, `i.title`, `i.placename`,
    `i.description`, `i.datasource_id`, `i.dataset_id`, `rt.type as record_type`, `i.state`, `i.lga`,
    `i.parish`, `i.feature_term`, `i.datestart`, `i.dateend`, `S as udatestart`, `E as udateend`,
    `i.source`, `i.external_url`, `i.image_path`, `i.glycerine_url`, `i.created_at`, `i.updated_at`,
    `i.extended_data`, `i.latitude`, `i.longitude`), as raw expressions.
  - `FeatureSerialiser::feature(\stdClass $row, array $keyById): array` — spec §9 exactly, property
    order as the spec's table.

- [ ] **Step 1: Write `FeatureSerialiserTest`**

From hand-built row objects:
- every property of spec §9 present, in order; `null` where empty;
- `udatestart`/`udateend` passed through as integers or `null` (⑥ is pinned in Task 13 end to end);
- `extended_data` `{"description": "contributor text"}` while `properties.description` keeps the
  built-in value (⑤);
- values are plain: `http://x` stays `http://x` (no `<a>`), `<b>` stays `<b>`, `&amp;` decodes to `&`;
- duplicate field names → first value; CDATA read; `<ExtendedData/>` → `{}` with
  `extended_data_unreadable: false`; malformed XML → `{}` with `true` (Review Focus 5);
- `{}` serialises as a JSON object, not `[]`;
- `latitude` null → `geometry: null`; `0, 0` → `Point [0, 0]` (zero is a coordinate);
- `image_url` = `Storage::disk('public')->url('images/' . $path)`, `url` = `url('places/' . $uid)`;
- `created_at` in ISO 8601 with `Z`.

- [ ] **Step 2: Run to verify it fails.** **Step 3: Implement** (SimpleXML with
  `libxml_use_internal_errors(true)`, cleared afterwards; empty object as `new \stdClass`).
- [ ] **Step 4: Run to verify it passes**, then the guard.
- [ ] **Step 5: Commit** — `git commit -m "Serialise v2 features with namespaced extended data"`

---

### Task 13: Place search

**Files:**
- Create: `app/Api/V2/Query/PlaceSearch.php`, `app/Http/Controllers/Api/V2/PlaceController.php`,
  `tests/V2/Feature/PlaceSearchTest.php`
- Modify: `routes/api_v2.php`

**Interfaces:**
- Consumes: Tasks 6–12.
- Produces:
  - `PlaceSearch::ALLOWED` — the parameter names of spec §10.1.
  - `PlaceSearch::SORT_KEYS` — `['id' => 'i.id', 'title' => 'i.title', 'start' => START_KEY,
    'end' => END_KEY]` with their cursor key types.
  - `PlaceSearch::run(RecordFilters $f, Sort $sort, Paging $paging, ?Sample $sample, string $hash):
    array` → `['rows' => …, 'next_cursor' => ?string, 'per_page' => int]`.
  - `PlaceController::search(Request $request): JsonResponse` — inside `DB::transaction`:
    `ParameterSet::fromRequest`, `RecordFilters`, `Sort`/`Paging`/`Sample`, `prepareFuzzy`,
    `Summary::compute(…, true)`, and unless `summary_only`, `PlaceSearch::run` and
    `FeatureSerialiser`. Builds `page.next` as `$request->url()` + the request's query with `cursor`
    replaced.

- [ ] **Step 1: Write `PlaceSearchTest`** — HTTP tests against `/api/v2/places`, grouped by DESIGN
  item, each test name prefixed with it:

```php
test_2_no_response_is_a_redirect          // a sweep of 10 varied valid and invalid requests: no status 3xx
test_2_unknown_parameter_is_400           // ?datefrom=1850 → INVALID_PARAMETER, parameter 'datefrom'
test_2_bad_extended_data_is_400           // ?extended_data=Years → FILTER_REJECTED, position 5
test_2_private_layer_is_403               // ?layers={private} → LAYER_PRIVATE
test_3_pages_beyond_any_ceiling           // 1,203 records, per_page=500 → 500, 500, 203; ids unique and complete
test_3_cursor_bound_to_query              // next_cursor reused with name=other → INVALID_PARAMETER 'cursor'
test_3_per_page_may_change_between_pages
test_4_summary_describes_match_set_not_page   // per_page=2: summary.total and by_layer equal the unpaged figures
test_4_summary_only                       // body keys exactly ['query','summary']
test_5_extended_data_namespaced           // a 'description' field does not replace properties.description
test_6_udateend_from_dateend              // datestart 1850, dateend 1860 → udateend === Civil::udate(1860)
test_7_sort_keeps_undated                 // sort=start: undated records present, last; nulls=first: first
test_7_date_filter_keeps_undated_by_default
test_7_include_undated_false_drops_them_and_summary_says_so
test_8_limit_is_deterministic             // limit=5 twice → same ids; equals the first 5 of the unlimited order
test_8_limit_across_pages                 // limit=7&per_page=3 → 3, 3, 1; total still the full count
test_8_sample_repeatable_with_seed        // sample=5&seed=abc twice → same ids; no seed → seed echoed, reuse reproduces
test_8_sample_conflicts_with_sort         // → INVALID_PARAMETER, rule 'conflicts'
test_query_echo_has_every_parameter_with_defaults  // match 'exact', sources all four, include_undated true, per_page 500, sort 'id'
test_records_without_coordinates_have_null_geometry
test_empty_search_pages_everything        // no filters → total equals all visible fixture records
test_response_shape                       // type FeatureCollection; keys query, summary, page, features
```

- [ ] **Step 2: Run to verify it fails.** **Step 3: Implement**; register
  `Route::get('places', 'PlaceController@search')`.
- [ ] **Step 4: Run to verify it passes**, then the full V2 suite and the guard.
- [ ] **Step 5: Commit** — `git commit -m "Add v2 place search"`

---

### Task 14: Layer detail and layer features

**Files:**
- Create: `app/Api/V2/Serialise/LayerSerialiser.php`, `app/Http/Controllers/Api/V2/LayerController.php`
  (methods `show` and `features`), `tests/V2/Feature/LayerDetailTest.php`,
  `tests/V2/Feature/LayerFeaturesTest.php`
- Modify: `routes/api_v2.php`

**Interfaces:**
- Consumes: Tasks 6–13 (`LayerGate`, `RecordFilters` built with only `layers` set, `Summary`,
  `KeysetPager`, `FeatureSerialiser`).
- Produces:
  - `LayerSerialiser::layer(\stdClass $dataset, array $keywords, ?string $recordType): array` — the
    spec §10.3 `layer` object, every key present, blank → `null`, other strings verbatim (no trim).
  - `LayerController::show(int $id)`, `LayerController::features(Request $request, int $id)`;
    features' `ALLOWED` = `sort`, `direction`, `nulls`, `limit`, `cursor`, `per_page`; `SORT_KEYS` add
    `order` → `i.dataset_order` (nullable int).

- [ ] **Step 1: Write the tests**

`LayerDetailTest`:
- private → 403 `LAYER_PRIVATE`; missing → 404 `LAYER_NOT_FOUND`; `/api/v2/layers/abc` → 404
  `NOT_FOUND`; any query parameter → 400 `unknown`.
- every key of spec §10.3 present; `warning` `"None"` returned as `"None"`; a licence with leading
  spaces returned untrimmed; an empty `license` → `null`; `subject_keywords` from `Fixtures::keyword`.
- `declared` carries the stored values, including a malformed `latitude_from: -10, latitude_to: -30`
  unchanged, and `summary.extent` comes from the records, not from `declared`.
- `summary` has no `by_layer`.

`LayerFeaturesTest`:
- default order is `dataset_order` then id; `sort=start&nulls=first` honoured.
- paging a 1,203-record layer with `per_page=500` visits every record once.
- every page carries the same `layer` object, with its `warning`.
- `summary` is over the whole layer regardless of `limit`.
- `line=time` → 400 `unknown`; `bbox` → 400 `unknown`; `sample=3` → 400 `unknown`.
- a record without coordinates is present with `geometry: null` (v1's feed drops it).

- [ ] **Step 2: Run to verify they fail.** **Step 3: Implement**; register
  `Route::get('layers/{id}', …)->where('id', '[0-9]+')` and the `features` route likewise.
- [ ] **Step 4: Run to verify they pass**, then the full V2 suite and the guard.
- [ ] **Step 5: Commit** — `git commit -m "Add v2 layer detail and paged layer features"`

---

### Task 15: The catalogue

**Files:**
- Create: `app/Api/V2/Query/LayerCatalogue.php`, `tests/V2/Feature/LayerCatalogueTest.php`,
  `tests/V2/Feature/CatalogueSearchInvariantTest.php`
- Modify: `app/Http/Controllers/Api/V2/LayerController.php` (method `index`), `routes/api_v2.php`,
  `app/Api/V2/Serialise/LayerSerialiser.php` (method `catalogueEntry`)

**Interfaces:**
- Consumes: `RecordFilters`/`RecordQuery` (record-level filters only), `KeysetPager`, `Sort`,
  `Paging`.
- Produces:
  - `LayerCatalogue::ALLOWED` — spec §10.2's names; `SORT_KEYS` `id` → `d.id` (int), `name` →
    `d.name` (nullable string).
  - `LayerCatalogue::run(array $layerFilters, RecordFilters $records, Sort $sort, Paging $paging,
    string $hash): array` → `['total' => int, 'rows' => …, 'next_cursor' => ?string,
    'per_page' => int]`. `$layerFilters` keys: `q`, `record_type`, `creator`, `has_license`,
    `has_warning`, `min_records`.
  - `LayerSerialiser::catalogueEntry(\stdClass $row, bool $withMatching): array` — spec §10.2 entry.

- [ ] **Step 1: Write `LayerCatalogueTest`**

- ① **record-level `bbox`**: layer A with one record inside the Hunter Valley box and the rest
  worldwide (its extent spans the globe) matches; layer B whose extent covers the box but has no
  record in it does not; `matching_count` for A is 1, `record_count` all of A.
- computed `bbox`, `date_range`, `undated`, `record_count` equal the fixture's; contributor-declared
  extents ignored.
- private layers never listed; a layer with no records listed with `record_count 0`, `bbox null`,
  `date_range null` unless a record-level filter or `min_records` excludes it.
- `q` matches name, description, and a subject keyword, case-insensitively; `creator` contains;
  `record_type` exact case-insensitive.
- `has_license=true` includes `"None"`-like text and excludes blank; `has_warning=false` returns only
  null or whitespace-only warnings.
- `min_records=2`.
- `matching_count` is `null` without record-level filters; `include_undated=false` alone leaves it
  `null`.
- paging with `per_page=2` and `sort=name` over names `b`, `B`, `a`, `null`, `null` visits each layer
  once (Review Focus 3).
- `total` is the matching layer count, not the page.

- [ ] **Step 2: Write `CatalogueSearchInvariantTest`**

A fixture of six layers (inside, straddling, outside the box; dated, undated, mixed; one private)
plus ANPS records. For each filter set — `bbox`; `polygon`; `date_from&date_to`; the same with
`include_undated=false`; `bbox` + dates — assert: the catalogue's ids (all pages) equal the sorted
distinct non-null `layer_id`s of `/api/v2/places` with the same filters (all pages).

- [ ] **Step 3: Run to verify they fail.**
- [ ] **Step 4: Implement** — one query: public datasets with the layer filters, joined to a per-layer
  aggregate over each layer's visible records (count, extent, `min(LO)`, `max(HI)`, undated counts),
  and, when `hasRecordLevelFilter()`, inner-joined to `RecordQuery::build($records)` grouped by
  `i.dataset_id` for `matching_count`. `total` is a `count(*)` over the same query before paging.
  Register `Route::get('layers', 'LayerController@index')`.
- [ ] **Step 5: Run to verify they pass**, then the full V2 suite and the guard.
- [ ] **Step 6: Commit** — `git commit -m "Add the v2 layer catalogue with record-level discovery"`

---

### Task 16: The v2 developer documentation

**Repository:** `tlcmap-docs`, branch `api-v2`.

**Files:**
- Create: `docs/developers/v2/index.md`, `places.md`, `layers.md`, `dates.md`, `errors.md`,
  `differences.md`
- Modify: `docs/.vitepress/config.mts` (a `API v2` sidebar group under `/developers/`),
  `docs/developers/index.md` (a row and a sentence pointing to v2, stating it is unstable until
  declared stable)

- [ ] **Step 1: Check out the branch**

```bash
cd /c/MyWork/systemik/projects/tlcmap/tlcmap-docs && git checkout api-v2 && git merge --ff-only main
```

- [ ] **Step 2: Write the pages from the code**

Each page is written from the merged v2 source and the spec, in that repository's style
(Australian English, never "GHAP"):
- `index.md` — base URL `https://tlcmap.org/api/v2`, JSON only, no redirects, strict parameters,
  the query echo, paging with cursors, the interim rate limit, status (unstable).
- `places.md`, `layers.md` — every parameter with its default and limits; the response shape; at
  least two examples each.
- `dates.md` — spec §6: forms, periods, BCE, UTC, undated vs unparseable, filter semantics.
- `errors.md` — the spec §11 table, with an example body for each code.
- `differences.md` — spec §14's list, each as "v1 does X; v2 does Y".

- [ ] **Step 3: Generate the examples from the local application**

Apply the three v2 migrations to `temp_tlcmap` (`php artisan migrate:status`, then
`php artisan migrate`; only the three `2026_11_10_*` migrations may be pending — stop if anything else
is). Serve the application through Laragon (`http://tlcmap-ghap.laragon.local`) with the `a1-api-v2`
branch checked out, and `curl` each documented request — including
`/api/v2/layers?bbox=150.8,-33.1,151.4,-32.6` and a Hunter Valley place search. Paste the responses,
trimmed only by removing features or array entries (mark a cut with a `…` entry); never edit a value.

- [ ] **Step 4: Build the site**

Run: `npm run build`
Expected: completes with no dead-link errors.

- [ ] **Step 5: Commit**

```bash
git add docs && git commit -m "Document API v2: places, layers, dates, errors and differences from v1"
```

---

### Task 17: Performance notes, acceptance and merge

**Files:**
- Create (this repository): `documents/acceptance/A1-API-V2.md`
- Modify (this repository): `README.md`

- [ ] **Step 1: Full verification under PHP 7.4**

Run: `$PHP74 artisan tlcmap:test-db && $PHP74 vendor/bin/phpunit --testsuite V2 && $PHP74 vendor/bin/phpunit --testsuite V1Guard`
Expected: all pass. Record the counts.

- [ ] **Step 2: Measure against `temp_tlcmap`**

The v2 migrations were applied to `temp_tlcmap` in Task 16. Record wall time (three runs, median) and `EXPLAIN ANALYZE` for spec
§13.6's five requests, through Laragon. Scale by 2,118 / 187 for the production estimate, and say that
it is an estimate.

- [ ] **Step 3: Write the acceptance record**

`documents/acceptance/A1-API-V2.md`: date; the application and docs commits; the test counts; a table
mapping each item ① to ⑧ (and the catalogue–search invariant and the v1 guard) to its test class and
methods; the performance table with the plans attached or linked; anything the bootstrap had to work
around in Task 1; and the items deliberately left for later (deployment, ⑱'s real limits).

- [ ] **Step 4: Update the README** — the documents table gains the acceptance record; the Status
  paragraph says the A1 API half is merged (not deployed).

- [ ] **Step 5: Commit both repositories' remaining changes**, then use
  superpowers:finishing-a-development-branch to merge `a1-api-v2` into `tlcmap-api-enhancement` —
  never into `main` or `dev`.

### Task 18: Two slow paths — `extended_data` pre-filter, dates parsed once

Added after Task 17's measurements by the user's decision (2026-11-05): no cache, no stored derived
data, no API change, and every response byte-identical.

- **`extended_data`.** A GIN trigram index on `dataitem.extended_data` (migration
  `2026_11_10_000004`, built concurrently) and a necessary-condition pre-filter on the raw text before
  each clause's XML parse, with a fallback (and a partial index) for records whose raw text may not
  contain an operand literally. Soundness proved from the writing code, by fixtures for each escape
  form, and by comparing matched ids with and without the pre-filter on `temp_tlcmap`.
- **Dates.** `tlcmap.v2_udate_range(text)` gives both bounds from one parse, used wherever a query
  needs both bounds of a column.
- Measure before and after; record the results in the acceptance record; then merge as Task 17
  Step 5.
