# Acceptance record: A1 API v2

The API half of milestone A1 — the first API v2 endpoints, items ① to ⑧ of
[DESIGN.md §5.2](../DESIGN.md#52-tier-1--required-by-a1) — as built to
[the spec](../specs/A1-API-V2.md) and [the plan](../plans/A1-API-V2-PLAN.md). Written against
[spec §15](../specs/A1-API-V2.md#15-acceptance).

Recorded 2026-11-05. The `EXPLAIN ANALYZE` plans are in
[A1-API-V2-PLANS.md](A1-API-V2-PLANS.md).

**Status: implemented and reviewed. Not merged, not deployed.** The merge into
`tlcmap-api-enhancement` waits on the user's decision on the two open items under
[Left for later](#6-left-for-later): the date-filtered catalogue and the `extended_data` filter
without `layers`.

## 1. Commits

| Repository | Branch | Commits |
| --- | --- | --- |
| `tlcmap-ghap` (the TLCMap application) | `a1-api-v2`, cut from `tlcmap-api-enhancement` at 310deea | 30 commits, 310deea..b1aa4c6 (head b1aa4c6, "Guard the test database rebuild and record its datasource adjustment"). 69 files, +8,259 −1; 36 under `app/`, `routes/`, `database/` (+3,256 −1) |
| `tlcmap-docs` | `api-v2` | 337b7ce, ca73f1a, 7d7cf0c on 5dd3e22 (8 files, +2,564), not pushed |
| `tlcmap-skills` (this repository) | `main` | spec and plan: this record's parent commit 20014f3 and before |

Files outside the v2 tree that the branch changes: `app/Exceptions/Handler.php`,
`app/Http/Kernel.php`, `app/Providers/RouteServiceProvider.php` and `phpunit.xml`, each to register
v2 and nothing else. No v1 controller, view or route file is touched (`git diff --name-only
310deea..HEAD` matches none of `public/`, `resources/views`, `routes/web`, `routes/api.php`); the v1
guard below is what proves the responses did not change.

## 2. Verification

Run 2026-11-05 at b1aa4c6, PHP 7.4.19 (`php-7.4.19-Win32-vc15-x64`, the production version), the
working tree clean:

```
$PHP74 artisan tlcmap:test-db                    Test database 'tlcmap_test' is ready.
$PHP74 vendor/bin/phpunit --testsuite V2         OK (319 tests, 2294 assertions)
$PHP74 vendor/bin/phpunit --testsuite V1Guard    OK (5 tests, 15 assertions)
```

`tlcmap:test-db` rebuilt `tlcmap_test` from nothing: every migration (the 36 that existed plus the
three `2026_11_10_*` v2 migrations) ran from scratch. The command refuses a database whose name
does not end in `_test`.

Spec §15, point by point:

| §15 | Met |
| --- | --- |
| 1. Code, migrations and tests merged on `tlcmap-api-enhancement` | **Not yet** — on `a1-api-v2`, awaiting the user |
| 2. `tlcmap:test-db` builds from nothing; `V2` and the v1 guard pass under PHP 7.4 | Yes (above) |
| 3. Every item ① to ⑧ has its §13.4 tests; the invariant passes | Yes (§3 below) |
| 4. The §14 documentation committed on `api-v2` in `tlcmap-docs` | Yes (337b7ce, ca73f1a, 7d7cf0c) |
| 5. This record | Yes |

## 3. Items to tests

Test classes are under `tests/V2` (in `tlcmap-ghap`) unless noted. The first column is the item's
requirement in [DESIGN.md §5.2](../DESIGN.md#52-tier-1--required-by-a1).

| Item | Test class | Methods |
| --- | --- | --- |
| ① Layer catalogue with computed extent and facets | `Feature\LayerCatalogueTest` | `test_1_bbox_matches_records_never_extents`, `test_1_response_and_entry_shape`, `test_1_echo_carries_every_given_value`, `test_1_rejected_parameters`, `test_1_computed_figures_ignore_declared_fields`, `test_1_private_layers_never_and_empty_layers_listed`, `test_1_q_creator_and_record_type`, `test_1_record_type_is_the_layers_not_its_records`, `test_1_licence_and_warning_presence`, `test_1_min_records`, `test_1_matching_count_only_with_a_record_level_filter`, `test_total_counts_matching_layers_not_the_page`, `test_figures_are_computed_for_the_returned_page_only`, `test_min_records_counts_without_date_functions_and_keeps_empty_layers_at_zero` |
| ① (layer detail) | `Feature\LayerDetailTest` | `testEveryKeyIsPresentAndTextIsVerbatim`, `testDeclaredIsAsStoredAndSummaryComesFromRecords` |
| ② Honest errors, no redirects | `Feature\PlaceSearchTest` | `test_2_no_response_is_a_redirect`, `test_2_unknown_parameter_is_400`, `test_2_bad_extended_data_is_400`, `test_2_private_layer_is_403` |
| ② | `Feature\LayerDetailTest`, `Feature\LayerFeaturesTest` | `testPrivateMissingAndNonNumericLayers`, `testAnyQueryParameterIsRejected`; `testErrorsAndRejectedParameters` |
| ② | `Feature\SkeletonTest` | `test_unknown_v2_route_is_json_404`, `test_post_is_json_405`, `test_options_preflight`, `test_cors_on_errors`, `test_unexpected_error_is_opaque_500`, `test_api_exception_rendered` |
| ② | `Feature\RequestBytesTest` | `test_value_that_is_not_text_is_rejected`, `test_parameter_name_that_is_not_text_is_rejected_and_named_readably`, `test_nul_never_shortens_a_name_match`, `test_id_longer_than_eighteen_digits_is_not_found`, `test_eighteen_digit_id_reaches_the_layer_gate` |
| ② (the `extended_data` grammar and its rejections) | `Unit\ExtendedDataFilterTest` | `testRejections`, `testElevenClauses`, `testFieldMayBeginWithTheLettersAnd`, `testNumberOfAThousandDigitsEitherSideIsAccepted` |
| ③ Paging across every source | `Feature\PlaceSearchTest` | `test_3_pages_beyond_any_ceiling`, `test_3_cursor_bound_to_query`, `test_3_cursor_with_summary_only_is_400`, `test_3_per_page_may_change_between_pages`, `test_3_every_sort_and_nulls_visits_each_record_once_in_order` |
| ③ | `Feature\LayerCatalogueTest`, `Feature\LayerFeaturesTest` | `test_3_paging_by_name_visits_each_layer_once`; `testPagingVisitsEveryRecordOnceWithTheLayerOnEveryPage` |
| ③ (the pager and the cursor) | `Feature\KeysetPagerTest`, `Unit\CursorTest` | `testFollowingCursorsVisitsEveryRowOnceInTheUnpagedOrder`, `testACursorFromAnotherQueryIsRejected`, `testInsertingARowBeforeTheCursorDoesNotServeItOrChangeAnythingElse`, `testDeletingTheLastServedRowContinuesWithTheFollowingRow`, `testOnlyNullableKeysCarryTheNullPlacement`; `testQueryHashIgnoresCursorAndPerPageAndOrder`, `testRejectsWrongHash` |
| ④ A cheap summary | `Feature\PlaceSearchTest` | `test_4_summary_describes_match_set_not_page`, `test_4_summary_only` |
| ④ | `Feature\SummaryTest` | `testWholeMatchSetFigures`, `testByLayerOmittedWhenNotWanted`, `testFilteredSetAndCallerBuilderUntouched`, `testNoCoordinatesGivesNullExtent`, `testNoDatedRecordGivesNullDateRange`, `testEmptyMatchSet` |
| ⑤ Namespaced extended data | `Feature\PlaceSearchTest` | `test_5_extended_data_namespaced` |
| ⑤ | `Unit\FeatureSerialiserTest` | `testExtendedDataIsNamespacedAndDescriptionKept`, `testDuplicateNamesFirstWins`, `testParityWithSqlExtendedValue` |
| ⑥ A correct `udateend` | `Feature\PlaceSearchTest` | `test_6_udateend_from_dateend` |
| ⑥ (the date functions it rests on) | `Sql\DateFunctionsTest` | `test_valid_inputs`, `test_spec_reference_values_as_literals`, `test_safe_integer_boundary`, `test_unparseable_inputs_are_null` |
| ⑦ Never silently discard records | `Feature\PlaceSearchTest` | `test_7_sort_keeps_undated`, `test_7_date_filter_keeps_undated_by_default`, `test_7_include_undated_false_drops_them_and_summary_says_so` |
| ⑦ | `Feature\RecordQueryTest` | `testDateRangeKeepsOverlappingPeriodsAndUndatedRecords`, `testIncludeUndatedFalseDropsUndatedAndUnparseableRecords`, `testIncludeUndatedHasNoEffectWithoutADateFilter`, `testHalfUnparseableRecordIsDatedFromItsGoodSide` |
| ⑧ Deterministic `limit`; `sample` | `Feature\PlaceSearchTest` | `test_8_limit_is_deterministic`, `test_8_limit_across_pages`, `test_8_sample_repeatable_with_seed`, `test_8_sample_conflicts_with_sort` |
| ⑧ | `Feature\KeysetPagerTest` | `testLimitCapsTheResultAcrossPages` |
| Catalogue–search invariant | `Feature\CatalogueSearchInvariantTest` | `test_catalogue_lists_exactly_the_layers_place_search_draws_on` |
| v1 guard | `tests\V1Guard\V1ResponsesTest` (`tlcmap-ghap`) | `test_places_name_search`, `test_places_bbox_search`, `test_places_date_search`, `test_layer_feed`, `test_layer_catalogue` — v1 responses against committed golden files in `tests/V1Guard/golden/`, captured before the first v2 migration |

Beneath these, the unit and SQL tests (§13.2, §13.3) — `Unit\ParameterSetTest`,
`Unit\PagingParamsTest`, `Unit\ValueParsersTest`, `Sql\ExtendedValueFunctionTest`,
`Sql\DateFunctionShapeTest`, `Sql\IndexesTest` — and the shared `RecordQueryTest` cover the filters of
§7. The rate limit and CORS (§12) are pinned by `Feature\SkeletonTest::test_rate_limited`,
`test_v1_requests_do_not_count_against_v2`, `test_v2_requests_do_not_count_against_v1` and
`test_non_v2_paths_untouched`.

## 4. Performance notes (spec §13.6)

Not a gate. Measured 2026-11-05 at b1aa4c6, through Laragon (Apache, PHP 7.4.19) at
`http://tlcmap-ghap.laragon.local`, against `temp_tlcmap` (PostgreSQL 14.5, `shared_buffers`
128MB) with the three v2 migrations applied. `temp_tlcmap` holds 187 layers (155 public), 687,235
records, of which 61,392 are in layers (the rest are gazetteer records). The laptop is noisy by about
±0.1 s between runs.

Wall time is `curl`'s `time_total`, three runs, median, in seconds. The framework floor (a v2 404,
no database) is 0.326 s. `EXPLAIN (ANALYZE, BUFFERS)` of every statement each request issues is in
the [plans appendix](A1-API-V2-PLANS.md); the third column below names the statement that dominates.

| # | Request | Wall, three runs | Median | Dominant statement (plan execution) |
| --- | --- | --- | ---: | --- |
| 1 | `GET /api/v2/layers` | 0.730, 0.675, 0.638 | **0.675 s** | page figures over the 100 layers on the page, 319 ms |
| 2 | `GET /api/v2/layers?bbox=150.8,-33.1,151.4,-32.6` | 0.779, 0.706, 0.842 | **0.779 s** | page figures for the 8 matching layers, 297 ms |
| 3 | `GET /api/v2/places?bbox=150.8,-33.1,151.4,-32.6` | 0.523, 0.442, 0.464 | **0.464 s** | summary over 1,318 matching records, 15 ms |
| 4 | `GET /api/v2/places?name=Wollombi&match=fuzzy` (all four sources) | 0.511, 0.485, 0.556 | **0.511 s** | summary over the fuzzy matches, 25 ms |
| 5 | a deep page: `GET /api/v2/places?cursor=…` resuming at id 500,500 (100 pages of 5,000 in) | 1.397, 1.403, 1.332 | **1.397 s** | summary over every visible record, 856 ms; the page itself 125 ms |
| 6 | `GET /api/v2/layers?date_from=1820&date_to=1860` | 1.176, 1.079, 1.108 | **1.108 s** | the record-level matching join, 554 ms, plus page figures, 453 ms |
| 7 | `GET /api/v2/places?extended_data=State = "NSW"` (no `layers`) | 50.94, 51.57, 50.57 | **50.94 s** | summary: a sequential scan of `dataitem` applying `v2_extended_value` to every row, 46,536 ms |

Request 6 is the sixth request beyond §13.6's five (the catalogue with a date range), added because
it is a known residual (below). Request 7 is §13.6's `extended_data` filter without `layers`; the
field `State` is the most common in the data after two identifiers and `"NSW"` matches 3,308 records.
Request 5's cursor was obtained by following `next_cursor` through 100 pages at `per_page=5000`;
the request itself uses the default `per_page` of 500.

### Production estimates

An estimate, not a measurement: production has 2,118 layers against 187 here, a ratio of 2,118 / 187
= 11.33. A cost is scaled by that ratio only where it grows with the number of layers; where it
grows with records, the layer ratio does not apply, and production's record count was not measured
for this record, so no figure is given.

| # | Cost grows with | Production estimate |
| --- | --- | --- |
| 1 | the layers in the layer-set statements (6.6 ms here, ×11.33 = 74 ms); the page figures grow with the records in the 100 layers of the page, which are not scaled | **about 0.75 s** (0.675 − 0.007 + 0.074); a page of unusually large layers costs more |
| 2 | the layers in the matching join (44 ms reported here); the page figures grow with the records of the matching layers | **about 0.8–1.3 s** (0.8 s if the join does not grow with layers; 0.779 − 0.050 + 0.050 × 11.33 = 1.3 s if it grows fully with them) |
| 3 | the records inside the box (GiST index) | about as here if production has a similar number of records in the box; not scaled by layers |
| 4 | the records matching the trigram (GIN) | about as here if the number of fuzzy matches is similar; not scaled by layers |
| 5 | **all visible records**: the summary scans the whole table | not estimated; 1.4 s at 687,235 records, and the summary cost (0.86 s) grows with the record count |
| 6 | the records in all layers: the date predicate is evaluated on every dated record to know which layers match | **up to about 6 s** if the matching join grows fully with layers (1.108 − 0.463 + 0.463 × 11.33 = 5.9 s); [the earlier analysis](#6-left-for-later) split the join into a table-size part and a layer part and put the low end at about 3.7 s. A target of about 2 s is **not met** |
| 7 | **all records**: every row of `dataitem` is read and its XML parsed | not estimated; 50.9 s at 687,235 records (344,350 of them with extended data, 103 MB of text). At least this slow in production unless production has far fewer records. The target is not met by a factor of about 25 |

The other four requests (1, 2, 3, 4) are at or well under the accepted ~2 s on this estimate. For
reference, before the page-first rewrite in the final fix wave the unfiltered catalogue computed every
layer's figures on every request: 1.51 s here and about 16 s estimated for production
(`final-fix-report.md`, 2026-11-05).

## 5. What the bootstrap had to work around (Task 1)

The plan's first task established whether every migration runs from scratch (spec §13.1). It does,
with no change to any migration. Four things were done in the test bootstrap, none of them in a
migration production has run:

- **Schemas and extensions are created first.** `tlcmap:test-db` creates `tlcmap_test`, drops and
  recreates the `tlcmap` and `public` schemas, and creates `postgis` and `pg_trgm` in `public`
  before running the migrations.
- **The GEOCODER datasource moves to id 4.** The `create_text_table` migration inserts a GEOCODER row
  using the next sequence value, which is 1 on a fresh database; production's is 4. After migrating,
  the command moves it to id 4, then upserts datasources 1 to 3, and resets the `datasource` and
  `recordtype` sequences. The command's docblock records this.
- **Datasources 1–4 and recordtypes 1–9 are seeded** with production's `search_param_name` values
  and names, because the base schema appears to have come from a conversion rather than the
  migrations (spec §2).
- **The test database is the only target.** The command and `TestCase` refuse a database not named
  `*_test`; the command also checks the open connection's configured name before dropping anything.

The brief for Task 1 had two BCE reference values (for `udate(-399)` and `udate(-51020)`) one day
too negative; spec §6.1 and the plan were corrected (-74758377600000 and -1672202880000000, checked
against PostgreSQL's `make_timestamp` and an independent days-from-civil routine). The v2 date
functions were later compared with PostgreSQL's calendar on 1,607,433 days and 4,401 years: no
mismatches.

## 6. Left for later

### Deployment

Nothing here is deployed. tlcmap.org has none of the v2 migrations, code, or documentation.

- Deploy order: the three `2026_11_10_*` migrations, then the code. The trigram and GiST indexes
  are built `CONCURRENTLY` outside a transaction; **a build that fails part-way leaves an invalid
  index, and rerunning the migration skips it** (`IF NOT EXISTS`). Check `indisvalid` after
  migrating and drop and rebuild any invalid index.
- The `tlcmap-docs` branch `api-v2` is committed, not pushed or published.

### ⑱ Real rate limits

v2 ships with a single configured limit, 60 requests a minute per address (`config/api_v2.php`,
`API_V2_RATE_LIMIT`), with its own counter separate from v1's. It is not a quota granted on a
credential. The MCP server's upstream **must not move to v2** until ⑱ is built: the server alone
would exceed 60 a minute and be throttled. Raising the limit is configuration until then.

### Release prerequisites

Found in review; deliberately not fixed on this branch (ledger ruling R14).

- **Apache trailing slash.** The rule in `public/.htaccess` redirects `/api/v2/places/` with a
  `301`. The rule is in a shared file outside this plan, and v2 promises no redirects.
- **`TrustProxies`.** The rate-limit key (the client address) and `page.next` (scheme and host)
  depend on it being configured for the production proxy. Behind it unconfigured, every client
  shares one key and `next` may carry the wrong scheme or host.
- **Maintenance mode.** A `503` from `artisan down` renders as a `500` under v2; it needs a line in
  spec §11 and a mapping.
- **The new indexes may change v1.** `v2_dataitem_title_trgm` and `v2_dataitem_placename_trgm` on
  `dataitem` could change the plans, and so the ordering where v1 has no `ORDER BY`, of existing v1
  queries. The v1 guard cannot see this at fixture size. Compare v1 plans on a production-sized copy
  before release.
- **GIN index cost on bulk import.** Two trigram GIN indexes on `dataitem.title` and `placename`
  slow large layer imports; measure an import before release.
- **MCP upstream stays on v1** until ⑱ (above).

### Residuals for the user's decision

1. **Date-filtered catalogue** (request 6). 1.1 s here, up to about 6 s estimated for production
   against a target of about 2 s. Its cost is the record-level matching join, which must evaluate the
   date predicate on every dated record to know which layers match and to give `total` and
   `matching_count`; page-first cannot reduce it. The option within the ruling: parse each date
   string once for both bounds — a function returning both the first and last instant of a date
   would halve the date work in the join (the earlier analysis put 3.0 s at about 1.7 s). Not done:
   the target is the user's call (ledger rulings R16).
2. **`extended_data` without `layers`** (request 7). 50.9 s here, found while taking this record. The
   filter applies `v2_extended_value`, an XML parse, to every row of `dataitem` (344,350 with extended
   data), and nothing narrows the scan; an unindexable expression. Options for the user, none built:
   require `layers` or another narrowing filter with `extended_data`; add a `max` on the matching
   work with a `400` that names it; or a stored, indexed extraction of named fields (which contradicts
   the design's "no derived data" and so needs the user's decision).

### Parked minors (review residuals, ledger ruling R18)

Non-blocking; diagnostics or development only.

- The test-database guard in `BuildTestDatabase` reads the configured database name, not
  `current_database()`.
- The catalogue runs two statements (the layer set, then the page's figures) under `READ COMMITTED`;
  a concurrent write between them can make `min_records` and `record_count` disagree for a layer.
  The place search's summary and page have the same property.
- A trailing bare `AND` in `extended_data` is reported as "an operator" rather than "a clause".
- A development database migrated with the pre-fix version of migration `2026_11_10_000001` keeps an
  orphan function `v2_civil_ms`; harmless, removed by rebuilding.

### Deferred minors (ledger)

One line each; none blocks.

- Task 1 — `BuildTestDatabase` ignores `migrate`'s return code; moving the GEOCODER row is silent if
  it is absent; `TestCase`'s `_test` guard runs after `DatabaseTransactions` opens.
- Task 2 — v1 goldens assume `APP_URL` `http://localhost`, the Australia/Sydney timezone and tzdata
  for 1856; none exercises an empty-date record; goldens depend on insertion order (v1 has no
  `ORDER BY`); the `{feed}` sentinel is indirect.
- Task 3 — the immutability test covers the two public functions only; plpgsql wrappers cannot be
  inlined (since replaced by `sql` wrappers for the public pair).
- Task 4 — `//Data` matches at any depth and misses default-namespaced `Data` (comment it); `Data`
  with no `<value>` returns `''`, untested; no test of `&lt;` `&gt;` `&amp;` in a name.
- Task 5 — `ErrorRenderer` maps `ModelNotFoundException`, other `HttpException`s and
  `ValidationException` to an opaque 500; the 500 test does not pin `details` as `{}`; a malformed
  `API_V2_RATE_LIMIT` yields a maximum of 0.
- Task 6 — the raw-query name counter does not trim leading spaces as PHP does (`%20name` repeat
  missed); no tests pin raw-query edge cases (`&&`, trailing `&`, `=x`, encoded names).
- Task 7 — `Sources::idsByKey` queries on every call and silently drops a key whose datasource row is
  missing; no tests for the 400-digit decimal and the 19-digit layer id (the latter now covered).
- Task 8 — word-operator-named fields are inconsistent (`after = 1` parses at offset 0, not after
  `AND`); `Name textmatch` at the end reports "an operator", not "a value"; several grammar cases
  untested.
- Task 9 — `ORDER BY` uses expressions, not the `_k` and `_id` aliases, and breaks under `DISTINCT`;
  the 18-digit guard reports a misleading range for valid 19-digit values; no test for a cursor
  `n` at or beyond `limit`.
- Task 10 — the combined-filter test lacks a polygon with `extended_data`; a missing datasource row is
  dropped silently (as Task 7).
- Task 11 — no filtered summary test with a binding filter; an unused loop variable.
- Task 12 — the timestamp test depends on `app.timezone` being Australia/Sydney; no pre-1900 case.
- Task 13 — the layer gate and polygon checks run before `Sort` and `Paging` parse (a mixed-bad
  request gets 403 before 400); `sample` with `per_page` reports `page.per_page` different from the
  number returned; no feature test of the 404 `LAYER_NOT_FOUND` body through `/api/v2/places`;
  `page.next` depends on `TrustProxies`.
- Task 14 — parameter validation runs before the layer gate (400 before 403); duplicated `ID`
  constant and `per_page` config lookup; no test of cross-layer cursor replay or a whitespace-only
  text field.
- Task 15 — the catalogue's aggregate expressions duplicate `Summary`'s; the echo relies on `ALLOWED`
  order; no test of a stale cursor past the end of the catalogue; the null-name paging test drops
  `NOT NULL` so the real path (identical or empty names across a page boundary) is untested.
- Task 16 — in `tlcmap-docs`: a run-on sentence in `differences.md`; `dates.md` does not say `D/M/YYYY`
  needs exactly four year digits; a ~3 KB description in the BCE example; the `413` note in
  `errors.md`; long pages lack jump links. The example URLs show `https://tlcmap.org` where the
  responses were generated on the local host (disclosed on the index page).

### Spec text to follow the final fix wave

The 1,000-digit bound either side of the decimal point in an `extended_data` comparison, the
`encoding` rule, and the route bound of 18 digits on the layer id are in the spec (rulings R17, R18).
