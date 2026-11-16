# ADR-0002: Promotion/relegation across tiered European leagues

- **Status:** Accepted (domain metadata to land immediately in
  cutline; season-end automation deferred until after the first
  European multi-tier wave ships)
- **Date:** 2026-11-16
- **Scope:** cutline — `internal/domain/`'s Division catalog, the
  `seed/<sport>.yaml` team registry, the ingest-worker's season-
  roll responsibilities, and any downstream consumer that asks
  "which league is this team in this season?"
- **Issue:** [jedi-knights/cutline#444](https://github.com/jedi-knights/cutline/issues/444)

## Context

cutline's current data model is a closed-franchise model. Every
sport × division tuple (soccer-women × d1, soccer-women × nwsl,
soccer-men × mls, ...) carries a fixed team registry in
`seed/<sport>.yaml`. The registry is a snapshot — teams don't move
between divisions in-season — and the only maintenance signal is
"did the seed change in a PR?", so it holds up fine for the current
product.

The European leagues initiative (issue #478) breaks that model. In
European football, each country has a **tiered pyramid**: a top
flight, one or more lower tiers, and promotion/relegation movement
between them at the end of every season. Examples in scope:

| Country | Tier 1 | Tier 2 | Tier 3 | Tier 4 |
|---|---|---|---|---|
| England (men's) | Premier League | EFL Championship | EFL League One | EFL League Two |
| Spain (men's) | La Liga | Segunda División | — | — |
| Italy (men's) | Serie A | Serie B | — | — |
| Germany (men's) | Bundesliga | 2. Bundesliga | — | — |
| France (men's) | Ligue 1 | Ligue 2 | — | — |

At the end of each season, the bottom N teams of each tier are
**relegated** to the tier below and the top N of the tier below
are **promoted** up. Specific counts vary (EPL has 3-up/3-down,
Bundesliga has 2-up/2-down + one playoff spot, etc.). The committed
set of teams in "Premier League" is therefore a snapshot of
*this season* — it is not the identity of the league. A valid
query like "who was in the Premier League in 2022-23?" is
unanswerable from the current snapshot.

Shipping any multi-tier European country into cutline without a
plan for this produces three concrete failures:

1. **Registry drifts.** `seed/soccer-men.yaml` currently has one
   division per team. When Leicester drops from EPL to the
   Championship, the seed needs an edit that touches both leagues
   on the same line. Doing that by hand in a once-a-year PR is
   feasible but error-prone; doing it without any convention is
   guaranteed to drift.
2. **No historical shape.** The product line has already said
   yes to questions like "has this school ever been in the top
   flight?" in NCAA contexts (which never has to answer it,
   because NCAA tiers are stable). For European clubs the
   answer is both interesting and inherently multi-season.
3. **Standings handlers can't tell siblings apart.** Nothing in
   the current `Division` enum says "EFL Championship is below
   Premier League and above League One." Every join or roll-up
   that wants to express "England men's second tier" has to
   hardcode the slug.

## Options considered

### Option 1 — Season-scoped `league_memberships` table

Introduce `league_memberships(team_slug, division, season)` with
one row per team per season. Teams keep a stable `team_slug`
across tiers. Reads of "who is in X right now?" filter by current
season.

- **Pros**: Fully captures history. Clean query shape ("WHERE
  season = 2026"). Supports both the hot path and the historical
  path with the same table.
- **Cons**: New persistent schema, new migration, new ingest path
  to populate it. The historical queries are hypothetical right
  now — we have no committed product surface that asks "what tier
  was this team in during 2022-23?". Building schema for a
  hypothetical is exactly the kind of over-commitment cutline
  has been disciplined about avoiding (see `proSoccerLeagueSport`
  — a small catalog in `internal/domain/`, not a new table).

### Option 2 — Division tier metadata + manual annual reseed

Annotate each pro-soccer Division with its country, tier level, and
the sibling Divisions it promotes to / relegates from. Example:

```go
MLS:                 {Country: "usa", Tier: 1}
Eredivisie:          {Country: "ned", Tier: 1}
Primeira:            {Country: "por", Tier: 1}
EPL:                 {Country: "eng", Tier: 1, RelegatesTo: "efl-championship"}
EFLChampionship:     {Country: "eng", Tier: 2, PromotesTo: "epl", RelegatesTo: "efl-league-one"}
EFLLeagueOne:        {Country: "eng", Tier: 3, PromotesTo: "efl-championship", RelegatesTo: "efl-league-two"}
EFLLeagueTwo:        {Country: "eng", Tier: 4, PromotesTo: "efl-league-one"}
```

Cross-season team movement stays a manual YAML edit at season
end. Documented as a maintenance operation: open a PR that moves
each relegated team from its old seed block to its new seed
block, merge, re-ingest.

- **Pros**: Zero new schema. Immediate value — the metadata
  answers "is this team top-flight?", "which country?", "what is
  this league's second tier?" for every read today. Matches the
  shape precedent cutline already uses for `proSoccerLeagueSport`
  (`internal/domain/pro_soccer.go`) and `espnSlugOverride`
  (`internal/domain/espn_slug_override.go`): a small catalog in
  `internal/domain/` with a pure-Go accessor. The manual reseed
  is a known cost, annual in frequency, and matches the existing
  seed workflow.
- **Cons**: No history. Historical questions like "what tier was
  Luton Town in during 2024-25?" return "unknown"; the registry
  reflects the current snapshot only. Season-end reseed is
  manual and operator-driven, so a late reseed shows last-season
  rosters in the new tier.

### Option 3 — Separate `LeagueTier` domain entity

Redefine `Division` as "country × league identity" (keeping the
slug) and introduce a new `LeagueTier` domain concept that
records per-season membership. The relationship becomes
`Team <-many-to-many-> Division` via a `LeagueTier` row-set.

- **Pros**: Most normalized. Clean domain story for anyone
  learning the model cold.
- **Cons**: Over-engineered for current scale. The relationship
  split `Division = identity + membership` is conceptually
  cleaner but adds a layer of indirection every reader has to
  learn before the hot path can be written. Both the current
  product surface and every roadmap item through Q1 can be
  served with a flat table + metadata catalog.

## Decision

**Hybrid of Option 2's metadata + Option 1's schema path as a
future upgrade.**

Ship Option 2 immediately: add a `DivisionTier` catalog in
`internal/domain/` that annotates each pro-soccer Division with
its country, tier level, and the Divisions it promotes to /
relegates from. Keep the seed YAML as a current-season snapshot.
Document the annual reseed as a deliberate operator step tied to
the European season calendar (May–July for most leagues).

**Defer Option 1's `league_memberships` schema** until a
committed product surface asks "what tier was this team in
during season X?" That question is hypothetical today. When it
becomes concrete, the migration path is clean: `league_memberships`
can be back-filled from historical seed states captured in git,
and the `DivisionTier` catalog from this ADR stays load-bearing
on top of it.

**Do not pursue Option 3.** The gain over Option 2 is
conceptual-cleanness, not operator value. cutline's precedent is
flat catalogs in `internal/domain/` with accessor functions, and
that scale continues to hold here.

## Rationale

### The question driving the decision

The forcing question is "what does the next European per-league
PR need to know that the current Division enum doesn't tell it?"
Answer: country, tier, sibling-tier Divisions. All three are
metadata, not history. Option 2 delivers them.

### Why not persist history now

cutline's product surface today is "RPI standings, schedules,
scoreboards, projected and real tournament brackets, team pages
with per-sport stats." Not one of those surfaces requires
"team-in-tier as of season X" to render today's view. A snapshot
is both sufficient and matches how the NCAA, NWSL, MLS, and USLSL
seeds already work. Shipping schema for a hypothetical is exactly
the trade cutline has been disciplined about avoiding; see the
project CLAUDE.md:

> Don't add features, refactor, or introduce abstractions beyond
> what the task requires. A bug fix doesn't need surrounding
> cleanup; a one-shot operation doesn't need a helper. Don't
> design for hypothetical future requirements.

### Why metadata here, not elsewhere

The `DivisionTier` catalog lives in `internal/domain/` because
that is the one consumer (handlers, SPA deep-link canonicalisers,
the gateway's OG display map) that reads it. The precedent is
explicit:

- `proSoccerLeagueSport` in `pro_soccer.go` — one catalog, one
  accessor, every pro-tier predicate consults it (#482's refactor
  made this the single source of truth, replacing six case-list
  sites).
- `espnSlugOverride` in `espn_slug_override.go` — one catalog
  per upstream-collision pair, shared between the standings and
  scoreboard adapters.

`DivisionTier` is the same shape and the same reader audience.

### Why manual reseed is acceptable for v1

Season ends happen annually — usually in May across European
leagues, July for the summer-cadence ones (MLS, NWSL, Damallsvenskan,
Norwegian Eliteserien, Swedish Allsvenskan). That cadence is slow
enough that a manual reseed PR against `seed/<sport>.yaml` is a
reasonable operator task — the same shape as adding a new NCAA
program when a school creates a team, which cutline does by hand
today. Automating promotion/relegation detection in the
ingest-worker is a sensible v2 — detectable from NCAA.com's own
standings ("who's in the Championship Round group" for European
Super-League-style tables, final-table positions elsewhere) — but
is deferred until (a) we have shipped the first multi-tier
country and (b) we've done at least one manual reseed and can
state what we want the automation to do.

## Implementation plan

### What lands immediately (1 cutline PR)

1. `internal/domain/division_tier.go` — the new catalog, modeled
   as a `map[Division]DivisionTier` + `DivisionTierFor(Division)
   (DivisionTier, bool)` accessor. Populate for every currently-
   shipped pro-soccer Division. Standalone top-flights (Eredivisie,
   Primeira, Jupiler, Scottish Premiership, Süper Lig, Austrian
   Bundesliga, Danish Superliga, Greek Super League, WSL, Liga F,
   Première Ligue, MLS, NWSL, USLSL) get `Tier: 1` and empty
   `PromotesTo` / `RelegatesTo`.
2. `internal/domain/division_tier_test.go` — pin the catalog
   against `proSoccerLeagueSport`: every pro-soccer Division
   must have a tier entry, and no NCAA tier does.
3. `docs/season-boundary.md` (or extension of `deploy/README.md`)
   — a one-page operator guide describing the manual reseed.

No handler changes yet — the metadata is read-ready but no caller
needs it until Wave 1b actually ships a multi-tier country. The
catalog landing pre-wave means the first multi-tier PR is
mechanical (add tier + sibling pointers) rather than schema-plus-
mechanical.

### What lands with the first Wave 1b country (per-league PRs)

Each multi-tier-country league PR fills in the `PromotesTo` /
`RelegatesTo` slots for its own Division and the siblings in scope.
Example for Spain, shipping La Liga (#447) + Segunda (#464) together:

```go
LaLiga:    {Country: "esp", Tier: 1, RelegatesTo: Segunda},
Segunda:   {Country: "esp", Tier: 2, PromotesTo: LaLiga},
```

### What deliberately stays manual for v1

- **Annual seed reseed at season end.** Documented process. The
  operator pulls the published final standings for each in-scope
  division, moves relegated teams down + promoted teams up in
  `seed/<sport>.yaml`, opens a PR, merges, runs `/ingest daily`.
- **Mid-season roster changes** (loans, transfers, name changes)
  — already manual today, no change.

### What waits for the Option-1 upgrade

A `league_memberships` table with history. The path is: when a
product surface needs it, add the migration, back-fill from git
history of `seed/<sport>.yaml` over the committed-main window,
and keep the `DivisionTier` catalog as the metadata layer on top.
Not blocking current work.

## Consequences

### Positive

- Wave 1b (10 PRs across 5 multi-tier countries) can proceed
  without waiting on any schema work. Each PR is a per-league
  Division entry + one row in the `DivisionTier` catalog.
- The metadata unblocks every current reader: "is this top-
  flight?", "what country?", "what league does the bottom of
  this league drop into?" — all derivable from Go at no runtime
  cost.
- No new persistent state means no new migration, no new ingest
  path, and no new rollback surface. The catalog is pure code.
- The catalog composes with existing single-source-of-truth
  patterns (`proSoccerLeagueSport`, `espnSlugOverride`), which
  the codebase already groups together.

### Negative

- Historical questions ("what tier was this team in during
  season X?") return "unknown" until Option 1 lands. Accepted
  because no product surface asks today.
- Annual reseed is a manual operator step. If the operator
  forgets or runs late, the SPA shows last-season rosters in
  the new tier until the reseed lands. Acceptable at current
  stakes.
- The transition to Option 1 (when it comes) will require
  back-filling `league_memberships` from git history. This is
  tractable (`git log --format=%H -- seed/<sport>.yaml` plus
  per-commit YAML diff), but it is the delayed-complexity cost
  of not committing to Option 1 up front.

## Open questions resolved

- **Do we need per-season team-in-tier queries today?** No —
  none of the committed roadmap items through Q1 need them.
- **Can wave 1a (standalone top-flights) proceed without this
  ADR landing?** Yes; it already has (eight leagues shipped).
  The ADR gates wave 1b (multi-tier countries).
- **Where does the catalog live?** `internal/domain/` as a Go
  catalog, same shape as `proSoccerLeagueSport`.

## Review signals

Audit against this ADR on every per-European-league PR:

- **Must Fix**: a multi-tier country shipped without filling in
  `PromotesTo` / `RelegatesTo` for every sibling tier that is in
  scope.
- **Should Fix**: a reader hardcodes a country code or tier
  number rather than reading from the catalog.
- **Consider**: a new query against the catalog that could be
  expressed as a helper (e.g. `DivisionsInCountry(country)`),
  and is being spelled out at the call site more than twice.

## References

- Issue #444 — Promotion/relegation across tiered European leagues (this ADR's backing issue)
- Issue #478 — European leagues initiative (umbrella)
- cutline `internal/domain/pro_soccer.go` — the precedent
  single-source-of-truth catalog this ADR composes with
- cutline `internal/domain/espn_slug_override.go` — the
  precedent cross-adapter-shared catalog pattern
- cutline PR #482 — refactor that made `proSoccerLeagueSport`
  the single source of truth, the pattern this ADR's catalog
  joins
