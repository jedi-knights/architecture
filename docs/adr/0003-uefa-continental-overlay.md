# ADR-0003: UEFA continental overlay on domestic leagues

- **Status:** Accepted (domain + seed land immediately in cutline;
  SPA cross-appearance surfaces defer until the first UEFA
  competition PR ships)
- **Date:** 2026-11-16
- **Scope:** cutline — `internal/domain/`'s Division catalog, a
  new `seed/uefa-entries.yaml` participants file, the ingest-
  worker's registry phase, the standings/scoreboard adapters'
  ESPN slug map, the team-page handler's cross-division game
  union, and the SPA's team-page surface.
- **Issue:** [jedi-knights/cutline#445](https://github.com/jedi-knights/cutline/issues/445)

## Context

cutline's current data model is one team ↔ one division. A team
is "in the Premier League" — not "in Premier League AND in
Champions League this season." The European leagues initiative
(issue #478) brings in four UEFA club competitions that overlay
domestic leagues:

| Competition | Sport | ESPN slug | Qualifier shape |
|---|---|---|---|
| UEFA Champions League (men) | soccer-men | `uefa.champions` | Top N of each UEFA country by coefficient |
| UEFA Europa League (men) | soccer-men | `uefa.europa` | Next tier down + domestic-cup winners |
| UEFA Europa Conference League (men) | soccer-men | `uefa.europa.conf` | Lower-coefficient countries + play-ins |
| UEFA Women's Champions League | soccer-women | `uefa.wchampions` | Top 1–2 of each UEFA women's top-flight |

Three facts make these different from every league cutline has
shipped so far:

1. **Participation is derived from prior-season domestic finish.**
   Who plays UCL 2026-27 was decided by 2025-26 EPL / La Liga /
   Serie A / Bundesliga / Ligue 1 standings (plus cup winners
   and the Europa League champion's automatic spot). The set
   shifts every August.
2. **A team appears in two `(sport, division)` tuples
   concurrently.** Arsenal 2026-27 is in both `soccer-men/epl`
   and `soccer-men/uefa-cl`. Every team-page query that filters
   by a single division will miss the other half of the team's
   calendar.
3. **Eliminations happen mid-tournament.** A team in the UCL
   group stage may be eliminated in December and keep playing
   only domestic fixtures from then on. "Who is in the UCL" is
   not a stable set within a season — it has both an "all
   participants" shape and an "alive" shape.

Attempting to render the four UEFA comps with the current one-
team-one-division shape produces three concrete failures:

1. **Registry duplication.** `seed/soccer-men.yaml` would need
   `arsenal` listed under both `epl` and `uefa-cl`, with every
   downstream team-slug uniqueness assertion now broken.
2. **Team page shows only half the fixtures.** The team-page
   handler currently queries games where the team is in a
   specific division; with Arsenal in two divisions, a
   division-filtered query drops the Champions League nights.
3. **Manual seed drift.** UCL/UEL/UECL participants change every
   August based on the previous season's finishes. Without a
   seed convention, the annual refresh is error-prone and the
   "who's in this year" question drifts from reality within
   weeks of the group-stage draw.

## Options considered

### Option 1 — `competition_entries` table

New table `competition_entries(team_slug, competition_slug,
season)` with one row per participation per season. The
ingest-worker populates it; team-page queries join against it to
assemble the union across domestic + continental.

- **Pros**: Captures history ("was Arsenal in the UCL in
  2024-25?"). Supports multiple competitions per team per season
  natively. Mid-season eliminations representable with an extra
  `eliminated_at` column.
- **Cons**: New persistent schema, new migration, new ingest
  write-path. ADR-0002 made the same trade and chose Option 2 —
  a catalog + manual seed — for the same reasons: no committed
  product surface today asks the historical question, and
  building schema for a hypothetical is exactly what the
  cutline CLAUDE.md warns against.

### Option 2 — Hardcoded derivation rules

Encode each country's UEFA qualification rule ("top 4 of EPL →
UCL group stage", "5th → UCL play-off round", "6th + 7th → UEL
group stage", "cup winner → UEL"). Compute participants at
display time from the domestic standings.

- **Pros**: No new seed data to maintain; participants emerge
  from standings automatically.
- **Cons**: UEFA coefficient recalculation changes the rules
  every few seasons (number of CL slots per country, play-off
  routes, Conference-League pathways). Encoding the rules
  guarantees we're always one UEFA announcement behind reality.
  Qualification also depends on *domestic cup* results which
  cutline does not currently ingest. Fragile; not worth the
  effort saved on seed maintenance.

### Option 3 — Catalog + per-season participants seed YAML

Add a `ContinentalCompetition` catalog in `internal/domain/`
listing each UEFA comp by slug + sport + ESPN slug + qualifying
description. Participants per season live in
`seed/uefa-entries.yaml`, listing team slugs per competition.
The ingest-worker reads the seed each pass and writes a
`continental_entries(team_slug, competition_slug, season)`
**projection** for the team-page query to join against — the
same projection pattern `standings.conference_directory` uses
(ADR-0002's precedent of "catalog metadata + seed-driven
membership").

- **Pros**: Matches ADR-0002's pattern exactly — catalog in
  `internal/domain/`, YAML seed for membership, ingest-worker
  projects into a small schema the handler joins against. Every
  reader (handler, SPA, OG metadata) reads the catalog the same
  way. Zero coupling to the volatile qualification rules.
  Annual reseed in August when groups are drawn, tied to the
  existing seed workflow.
- **Cons**: No history (same trade ADR-0002 made). Participants
  are "all teams in the comp this season" with no model of
  mid-tournament elimination at the domain layer — the SPA
  reads elimination from the scoreboard's final-result state on
  the game rows, not from a separate "still alive" column.

## Decision

**Option 3 with a two-table projection.** Participants land in
`seed/uefa-entries.yaml`; the ingest-worker writes two small
projections:

1. `standings.continental_entries(team_slug, competition_slug,
   season)` — who is in each UEFA comp this season. Joined by
   the team-page handler to union domestic + continental games.
2. `standings.continental_directory(competition_slug,
   display_name, sport, espn_slug, qualifier_description)` —
   mirror of the `internal/domain/` catalog so the SPA can
   enumerate UEFA comps without a cross-context read.

UEFA comps **are new `Division` values** (`uefa-cl`, `uefa-el`,
`uefa-conf`, `uefa-wcl`). They are flagged in the catalog as
`IsContinental: true` so handlers and the SPA can tell "this
division's participants come from the overlay, not a domestic
pyramid." The existing ESPN adapter path — slug map in
`internal/scoreboard/adapters/scraper/espn.go` and
`internal/standings/adapters/espn/client.go` — handles
scoreboard + standings with one-line entries each (the ESPN
coverage was confirmed on 2026-11-16 in
`.claude/study/2026-11-16-european-womens-leagues/report.md`'s
pan-Europe probe).

**Defer** the `competition_entries` per-season table until a
committed surface asks the historical question. Same deferral
the promotion/relegation ADR made.

**Defer** automated qualification-rule derivation (Option 2) —
not pursued; brittle against UEFA coefficient changes.

## Rationale

### The question driving the decision

"What does the next UEFA PR need to know that the current
Division enum doesn't tell it?" Three answers:

1. UEFA comps are new divisions that pair with a sport (same
   shape as domestic leagues).
2. Each comp's participants this season — a set, read from seed.
3. Each team's current-season divisions — plural, for the
   team-page query.

Option 3 delivers all three without new schema for the
historical shape nothing reads today.

### Why UEFA comps as new Division values, not a separate entity

Three readers already shape their logic around `Division`:
standings handler, scoreboard adapter, SPA deep-link
canonicaliser. A new `ContinentalCompetition` entity parallel
to `Division` doubles each reader's surface for no gain — every
hot path reader would case over "Division or ContinentalCompetition."
A UEFA competition is a league's-worth of fixtures over a
season; `Division` already models that.

The `IsContinental: true` flag is the only differentiator the
hot paths need. It tells team-page queries to also check
`continental_entries` for the team's participation, and it
tells the SPA to render the comp with a knockout-bracket UI
once the group stage ends rather than a pure round-robin
table.

### Why projection + manual seed, not derivation

UEFA's qualification rules change every 2–3 seasons as
coefficients shift. The 2024-25 Champions League expansion
(from 32 to 36 clubs with a new league-phase format) is the
recent example — every encoded rule would have broken that
August. A seed YAML reseeded annually from UEFA's published
participant list is both more accurate and operationally
cheaper than any derivation we could write.

### Why a projection and not read-at-render from the YAML

The team-page handler joins team_slug to all current-season
divisions to union games across domestic + continental. Doing
that from a YAML file would mean parsing the YAML on every
request or caching it in-process (standings is horizontally
scaled; cache invalidation adds complexity). The projection
pattern — ingest-worker writes a small table, handler joins —
is already how cutline does seed → standings data flow
elsewhere (`conference_directory`, `team_directory`,
`division_directory`). Consistency wins.

### Why defer SPA cross-appearance surfaces

The "show Arsenal's UCL fixtures on their team page" surface is
clear in principle and ambiguous in detail. Options include: a
separate "Continental" tab, a visually-distinct highlight on
existing fixture rows, or a combined calendar with a competition
badge per row. The right answer depends on what users do once
one UEFA comp is actually shipping. Land the domain + seed +
projection in one PR; pick the UI shape in the first UCL PR
when the data is visible for testing.

## Implementation plan

### What lands immediately (1 cutline PR)

1. `internal/domain/continental.go` — new `ContinentalCompetition`
   catalog as a map `Division → ContinentalCompetition`.
   Populate for the four UEFA comps. Fields: `Sport`,
   `ESPNSlug`, `DisplayName`, `QualifierDescription`. Accessor:
   `ContinentalCompetitionFor(Division) (ContinentalCompetition, bool)`.
2. `internal/domain/division.go` — add the four Division
   constants (`uefa-cl`, `uefa-el`, `uefa-conf`, `uefa-wcl`) +
   `Valid()` entries.
3. `internal/domain/pro_soccer.go` — add the four new divisions
   to `proSoccerLeagueSport` with their sport pairing.
4. `internal/domain/sport.go` + tests — matrix entries for each
   new pair; `SupportsRPI` returns false; a new predicate
   `IsContinentalDivision(Division) bool` for hot-path readers.
5. `internal/scoreboard/adapters/scraper/espn.go` — four new
   slug map entries.
6. `internal/standings/adapters/espn/client.go` — same.
7. `seed/uefa-entries.yaml` — new file with the 2026-27
   participants per comp. Shape:
   ```yaml
   season: "2026-27"
   competitions:
     uefa-cl:
       - { team_slug: arsenal }
       - { team_slug: liverpool }
       # ...
     uefa-el: [...]
     uefa-conf: [...]
     uefa-wcl: [...]
   ```
8. `migrations/standings/NNNN_continental_projections.sql` —
   `continental_entries` + `continental_directory` tables.
9. `internal/standings/store/queries/games.sql` — new
   `UpsertContinentalEntry`, `UpsertContinentalDirectory`,
   `PruneContinentalEntriesNotIn` queries (same shape as the
   existing `conference_directory` projection).
10. `cmd/ingest-worker` — new `continentalIngest` phase reading
    `seed/uefa-entries.yaml` and calling the three queries.
    Runs alongside the existing registry phase.

### What lands with the first UEFA competition PR (#474 for UCL)

- Team-page handler extension — union domestic + continental
  games when the team has a row in `continental_entries` for
  the current season. The union is a SQL `UNION ALL` on the
  existing games query with the second arm filtered by the
  team's continental division slug.
- SPA team-page surface — chosen when the data is visible for
  testing. Options listed in "Why defer SPA cross-appearance
  surfaces" above.
- Gateway `/api/v1/*` allowlist entries for the UEFA comps so
  scoreboard + standings fan out correctly.

### What lands with subsequent UEFA PRs

- Each of #475 (UEL), #476 (UECL), #477 (WCL) is a one-file
  seed PR (adding participants to `uefa-entries.yaml`) plus
  any SPA-side labels/logos that weren't part of the first PR.
  No architectural changes — the first UEFA PR delivers the
  shape and each subsequent comp rides it.

### Deferred

- `competition_entries` historical table (no committed product
  surface asks the question).
- Automated qualification-rule derivation (brittle vs. UEFA
  coefficient changes; manual reseed is the right v1).
- Mid-tournament elimination modeling as a domain concept — the
  scoreboard's final-game state already carries the information
  the SPA needs; no additional schema required.

## Appendix: references

- Promotion/relegation companion ADR: `0002-promotion-relegation-across-tiered-leagues.md`
- Catalog-in-`internal/domain/` precedent: `internal/domain/pro_soccer.go`, `internal/domain/espn_slug_override.go`, `internal/domain/division_tier.go`
- Projection pattern precedent: `standings.conference_directory`, `standings.team_directory`, `standings.division_directory` (populated by `cmd/ingest-worker`'s registry phase from seed YAMLs)
- Pan-European women's ESPN probe confirming `uefa.wchampions` coverage: `cutline/.claude/study/2026-11-16-european-womens-leagues/report.md`
