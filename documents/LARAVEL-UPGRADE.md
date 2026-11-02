# TLCMap Laravel upgrade

**Status:** deferred to a later phase · **Date:** 2026-11-02

Companion to **[DESIGN.md](./DESIGN.md)**. The plan there does not depend on this upgrade: the MCP
server is a separate TypeScript service that reaches TLCMap through its public API
([DESIGN.md §6.1](./DESIGN.md#61-architecture)), and the API work is built on the application as
it stands ([§10](./DESIGN.md#10-plan)). This document covers why, to what, how and how long, for
when the upgrade is scheduled. It is not an upgrade checklist; that is written when the work
starts.

---

## 1. Why

The TLCMap application runs **Laravel 5.8 on PHP 7** (`laravel/framework 5.8.*`, `php ^7.1.3` in
`composer.json`). Both are long out of support: Laravel 5.8's last release was 5.8.38 on
2020-04-14, and PHP 7.4, the last 7.x line, stopped receiving security fixes in November 2022.

That alone justifies the upgrade: security fixes, and access to the maintained PHP ecosystem,
much of which now requires PHP 8.

## 2. Target

**Laravel 13 on PHP 8.3 or later.** The current major (13.39.0 at 2026-11-02, requiring
PHP ^8.3), so the longest support window.

In scope: the framework, the PHP runtime, and every dependency those force — including replacing
abandoned packages and removing unused ones.

Out of scope:

- **Restructuring to the Laravel 11+ application skeleton.** The existing structure remains
  supported; adopting the new one is optional and can follow later.
- **Front-end build tooling.** Laravel Mix stays unless a hop forces a change.
- **Feature or API work.** Nothing from [DESIGN.md §5](./DESIGN.md#5-layer-1--the-api) is mixed
  into the upgrade, so any regression is attributable to the upgrade alone.

## 3. Approach

### 3.1 Production data

A dump of the production database is loaded into both the local development environment and
staging, so the upgraded code meets real data before release, not only chosen test data. It is the
source for the fixtures (§3.2) and the subject of the catalogue comparison (§3.3).

The dump holds user accounts, so its copies are handled as production data, and outgoing mail is
disabled outside production — registration, verification and notification emails must not reach
real users from a test environment.

### 3.2 Tests first

The application has no tests today — `tests/` holds the two framework example tests. Before any
version changes, **characterisation tests** record what the application does now, so each hop can
be checked against it:

- public pages and search;
- every public output format — GeoJSON, KML, CSV and RO-Crate — compared against stored expected
  output;
- authentication, registration, email verification and administration;
- contributor workflows: creating and editing layers, adding and importing places.

They must pass on Laravel 5.8 before the first hop begins. They outlive the upgrade: the
output-format tests become Layer 1's regression suite for the API work that follows.

**Fixtures.** The tests run against a PostgreSQL / PostGIS test database loaded from a fixed
fixture set, and a test can only check behaviour on the data it runs against. A suite that passes
on tidy data proves the upgraded application handles tidy data, and nothing about the rest. So the
fixtures are chosen to represent what TLCMap actually holds, including the awkward cases:

- **Dates** — records with no dates, partial dates and unusual formats. `sort` silently drops
  undated records ([DESIGN.md §2.3](./DESIGN.md#23-behaviours-that-break-a-naive-client)), and a
  fixture in which every record is dated would never notice that behaviour change.
- **Names** — apostrophes, diacritics and other non-ASCII characters, especially in Indigenous
  placenames, where encoding and escaping faults surface in KML, CSV and GeoJSON.
- **Rights** — layers with a contributor warning, with a licence, and with neither; public,
  private and restricted layers, so attribution and access control are both exercised.
- **Messy values** — numbers stored as text, empty strings, stray whitespace. PHP 8 changed how a
  string such as `"12 "` or `""` compares with a number, without raising an error, so this is the
  only place that change can be seen. Hand-written test data almost never contains it.
- **Users** — one in each role, including an unverified account.
- **Size** — layers near the 5,000-record ceiling, and searches that paginate.

The layers are drawn from public layers in the production dump, which supplies the messy values for
free; synthetic records fill only the cases the real data lacks. The users are synthetic, because
the fixtures are committed to the repository and real accounts must not be.

Two judgements stay with the developer. The agent can propose cases from the schema and the code,
but only someone who knows the data can say which occur in production and which users rely on. And
the expected output records what the application does **today, faults included**. A fault the
tests expose is noted for after the upgrade, not fixed during it, so the upgrade changes nothing a
user can see.

### 3.3 Whole-catalogue comparison

The fixtures cover chosen cases; the comparison covers the real catalogue. Against the production
dump, a script requests the public outputs for a set of layers and collections, plus a fixed set of
gazetteer searches, and stores the responses as a snapshot. Every snapshot is compared with the
Laravel 5.8 **baseline** — not with the previous hop, so differences never accumulate unseen.

Two sizes of run:

- **Full** — every public layer and collection. Three times: the baseline on 5.8; after the 8 → 9
  hop, the first on PHP 8, where silent behaviour changes are likeliest and are cheapest to trace
  before four more hops land on top; and on Laravel 13, before staging.
- **Sampled** — after every other hop and every runtime change. About 200 layers and 50
  collections, drawn at random once with a fixed seed and reused unchanged, so every run compares
  like with like. The awkward layers are added on top: the largest, those with a warning or no
  licence, those with non-ASCII names, and every layer that fails in the baseline.

A random sample alone would miss what the comparison exists to catch. Drawing 200 of 2,169 layers
misses a fault confined to five layers about 60% of the time, and one confined to twenty about 15%
of the time. The sample keeps each hop quick; the full runs mean nothing reaches staging unchecked
against the whole catalogue.

- **What is requested.** Per layer: GeoJSON, KML, CSV, RO-Crate, and the statistics and analysis
  outputs. Then the same for collections, the layer and collection catalogues, and the gazetteer
  searches — `name`, `containsname` and `fuzzyname` over a few hundred placenames.
- **Status codes count.** The baseline records failures as well as successes. On 2026-11-02, one
  layer in a sample of five returned HTTP 500 for GeoJSON in production while its KML and CSV
  returned 200. A hop that changes a 500 to a 200, or the reverse, is a difference to explain.
- **Normalisation before comparison.** Some differences are noise: generation timestamps, dates
  written into RO-Crate metadata, zip packaging (compared by extracted contents), row order where
  the query sets none, and analysis outputs with random initialisation such as k-means. These are
  normalised or compared structurally. Writing that normalisation is most of the work, and it is
  done once, at the first comparison.
- **Every remaining difference is explained** — an intended change from the upgrade guide, or a
  regression to fix — before the hop is tagged.

**Disk and time.** Modest. Measured 2026-11-02 on five production layers sampled at random, a
record costs about 2.2–2.6 KB in GeoJSON, 1.1–1.5 KB in KML and 0.25–0.6 KB in CSV — about 5 KB a
record across all outputs. Production has 2,169 public layers; the record total will come from the
dump, but the partial local copy averages 374 records a layer (median 15), which puts the upper end
at about 800,000 records — around 4 GB per snapshot, and much less compressed, since the outputs
are text. Only two snapshots are ever kept: the baseline and the latest.

Each production request took 0.2–0.5 seconds, so a full run of about 25,000 requests is a few hours
unattended, under an hour with requests in parallel, and a sampled run is about a quarter of an
hour. That is machine time; the full runs go overnight. The developer's time goes on the
normalisation, written once, and on reading the differences — and one cause typically explains a
difference repeated across many layers, so that time grows far more slowly than the run does.

### 3.4 One major version at a time

Seven hops, all in the local development environment, each following that version's official
upgrade guide and each finished only when the full suite passes and the catalogue comparison
(§3.3) shows no unexplained difference. The hops are made by hand — no Laravel Shift — by a
developer working with an AI coding agent. Each hop is its own commit series on a single upgrade
branch, tagged when it passes, so a regression found later at staging can be bisected to the hop
that introduced it.

| Hop | PHP required by the target | Notes |
| --- | --- | --- |
| 5.8 → 6 | ^7.2.5 or ^8.0 | PHP 8 supported from 6.20 |
| 6 → 7 | ^7.2.5 or ^8.0 | |
| 7 → 8 | ^7.3 or ^8.0 | Class-based model factories |
| 8 → 9 | ^8.0.2 | First hop that requires PHP 8; mailer and filesystem libraries replaced |
| 9 → 10 | ^8.1 | |
| 10 → 11 | ^8.2 | `doctrine/dbal` dropped; migration column changes behave differently |
| 11 → 12 | ^8.2 | |
| 12 → 13 | ^8.3 | |

*PHP requirements are those of the latest release of each major on Packagist, 2026-11-02.*

**The framework and the PHP runtime never change in the same step.** Where a hop raises the PHP
minimum — 8 → 9, 9 → 10, 10 → 11 and 12 → 13 — the runtime moves first, on the framework version
already installed, which supports it, and the suite and the catalogue comparison run before the hop
begins.

### 3.5 Dependencies

Several packages are abandoned (`chumper/zipper`, `laravelcollective/html`, `fzaninotto/faker`,
`fideloper/proxy`), one is pinned to an old release (`symfony/translation 4.3.8`), and Scout with
the Algolia client appears to be unused — no class in `app/` uses it. Each is dealt with at the
hop that forces it, or before the first hop where that is simpler.

### 3.6 Release

**One release, at the end.** Nothing reaches a server until the local branch is on Laravel 13.

1. **Staging.** The upgraded application is deployed to the staging server for review. Staging and
   production both run Ubuntu 22.04 with PHP 7.4.33 under Apache `mod_php`. PHP 8.3 is installed
   from Ondřej Surý's PPA alongside 7.4, not in place of it. Apache loads one PHP module at a time,
   so the switch is disabling `php7.4` and enabling `php8.3`, plus moving the CLI default
   (`update-alternatives`) that Artisan, the scheduler and any queue workers use. No other
   application on either server uses PHP, so the switch affects TLCMap alone.
2. **Review.** Staging is the first environment outside development to run the new code, and the
   only one before production, so the review covers what the tests cannot: the interface end to
   end, email delivery, file exports and the web server and queue configuration.
3. **Production.** Released once the review signs off, the same way. The previous release and
   PHP 7.4 stay installed, so rollback is switching the release directory, the Apache module and
   the CLI default back, then restarting Apache — minutes, not a reinstall.

Rollback only stays that simple if the upgrade leaves the database schema alone. Any migration a
hop requires must be reversible, and is called out in the staging review.

## 4. The codebase, measured

From the local checkout of the TLCMap application, 2026-11-02:

| | |
| --- | --- |
| PHP in `app/` | 81 files, 14,352 lines |
| Blade views | 79 files, 9,284 lines |
| Routes | 299 lines in `routes/web.php`, 18 in `routes/api.php` |
| Migrations | 36 |
| Raw SQL (`DB::raw`, `DB::select`, `whereRaw` and similar) | about 50 call sites |
| Tests | none beyond the framework examples |

A small application. What makes the work larger than its size suggests is the missing test suite
and the raw SQL — PostGIS queries the framework cannot check for you.

## 5. Time estimate

Developer days for one developer familiar with Laravel, working with an AI coding agent.

The agent does the writing: the tests, the dependency changes and each hop's mechanical upgrade
work, which is most of what a by-hand estimate is made of. It does not remove the developer's
part, and that part is the floor on the estimate — deciding what the fixtures must represent,
reviewing every change, confirming that a passing suite means the application behaves the same,
checking the interface by hand after the hops that touch it, and the server work the agent has no
access to.

| Work | Developer days |
| --- | --- |
| Test environment and fixture data | 1–2 |
| Characterisation tests (§3.2) — review and gap-filling | 2–4 |
| Catalogue comparison (§3.3) — normalisation, reading the differences | 1–2 |
| Dependency preparation on 5.8 (§3.5) | 0.5–1 |
| The seven hops — review, runtime changes, hand checks | 4–6 |
| Staging: PHP 8.3, deployment, configuration | 1–2 |
| Fixes arising from the staging review | 1–2 |
| Production release | 0.5–1 |
| **Total** | **11–20 days — two to four weeks** |

For comparison, the same work without an agent is 25–40 days. The saving is in writing, not in
verifying; the riskiest stretches — the 8 → 9 hop, the first on PHP 8, where stricter typing and
changed comparison semantics can alter behaviour without raising an error, and the staging review —
shrink least.

The estimate assumes Scout and Algolia can be removed rather than migrated and that the raw SQL
needs no changes. It excludes elapsed time waiting on the TLCMap team's staging review.

## 6. Risks and open questions

**Risks.**

- **Behaviour the tests do not cover.** Characterisation tests catch what they exercise. Admin and
  rarely used contributor paths are the likeliest to be missed.
- **PHP 8 semantics.** Changes that do not error — string-to-number comparison, stricter internal
  function signatures — are the main source of silent regressions.
- **One release concentrates the risk.** Seven hops of change meet a real server for the first time
  at staging. The catalogue comparison (§3.3), the tagged hops (§3.4) and the side-by-side
  runtimes (§3.6) are the mitigation.
- **Ubuntu 22.04's remaining life.** Its standard support ends in 2027, and the PPA supplies PHP for
  supported releases only. Moving the servers to a newer Ubuntu LTS is outside this upgrade, but
  should be planned soon after it.
- **More to carry the longer it waits.** The API work in
  [DESIGN.md §5](./DESIGN.md#5-layer-1--the-api) is built on 5.8 meanwhile, and every line of it
  goes through every hop. It ships with its own tests, which join the characterisation suite (§3.2),
  and the measurements and estimate here (§4, §5) are re-taken when the upgrade is scheduled.
- **Freeze on the application.** Feature work merged during the upgrade has to be carried through
  every subsequent hop. A short upgrade beats a parallel one.

**Open questions.**

1. **Does the TLCMap team approve the upgrade, and who reviews it on staging?** The application and
   its production deployment are theirs.
