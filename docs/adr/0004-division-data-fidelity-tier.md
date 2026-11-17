# ADR-0004: Division data-fidelity tier

- **Status:** Accepted (catalog + SPA mirror land first; per-league
  Wikipedia-adapter PRs ride the tier flag)
- **Date:** 2026-11-17
- **Scope:** cutline — `internal/domain/`'s Division metadata, the
  standings/catalog response surfaces that consume it, the SPA's
  per-tab rendering decisions, and any future adapter authored
  against a non-ESPN data source.
- **Issue:** [jedi-knights/cutline#503](https://github.com/jedi-knights/cutline/issues/503)

## Context

Every pro-soccer league cutline has shipped so far rides the ESPN
adapter path and lands at a uniform coverage level: standings,
scoreboard, teams, per-team logos, identity-level rosters. The SPA
assumes that shape and renders every surface uniformly — a
Standings tab, a Schedule tab, a Team page with a Roster tab and a
team-logo column.

Issue #503's investigation (report at
`cutline/.claude/study/2026-11-17-503-alternate-source-decision/report.md`)
confirmed that the three ESPN-less European women's leagues
(#471 Frauen-Bundesliga, #472 Serie A Femminile, #473
Damallsvenskan) have **no free source with coverage comparable to
ESPN**. The only universally-reachable free source is Wikipedia,
which serves standings + per-round results and nothing else — no
rosters, no logos, no identity-level player data.

Shipping those three leagues at Wikipedia fidelity against the
current SPA produces three concrete failures:

1. **Broken team-logo column.** Every row renders a 404-hidden
   placeholder in the Standings tab because no logo URL resolves.
2. **Empty Roster tabs.** Every team page shows a Roster tab that
   produces an empty list, misleading the user into thinking the
   data is pending rather than unavailable by design.
3. **No honest attribution.** Wikipedia content is CC-BY-SA 3.0;
   shipping it without a credit line violates the licence.

The underlying question the SPA needs to answer per division is:
"which surfaces should I render, and against what expectations?"
The current model has no way to express the answer. Every division
is implicitly `ESPN-full`.

A future licensed-feed adapter (Opta / Stats Perform, the NWSL
shape) would introduce a *richer* tier than ESPN — per-player
season stats, per-match event data. That's the symmetric
problem: a licensed-tier division should surface per-player stat
leaders where an ESPN-tier division would hide the section.

So the unit of expression is a tier, and it affects multiple
surfaces at once.

## Options considered

### Option 1 — Boolean `IsFullTier`

Add `IsFullTier bool` to Division metadata. True for ESPN-covered
and NCAA leagues; false for Wikipedia-only. The SPA reads the
boolean and toggles the Roster tab / team-logo column.

- **Pros**: Simplest possible shape. One boolean, one branch.
- **Cons**: Collapses too many axes onto one bit. A future licensed
  tier (richer than ESPN) has no way to express itself. If the
  SPA eventually grows a third-way UI ("show season stat leaders"),
  the boolean needs replacing — same churn we'd pay once now.

### Option 2 — Named enum `DataFidelity`

Add a `DataFidelity` enum with three canonical values today
(`espn-full`, `wikipedia-standings`, `licensed`). The catalog in
`internal/domain/` maps each Division to its fidelity. The SPA
consumes a per-tier surface config: what tabs to show, which
columns to hide, what credit line to render.

- **Pros**: Each tier is a named, documented combination of
  surface decisions. New tiers are additive. The enum values are
  the same vocabulary the SPA, the handler, and the adapter-author
  documentation share. Matches the precedent of `SelectionMode`
  (ADR not written, but see `internal/domain/division.go` — the
  national/regional enum does exactly this for NCAA bracket
  selection).
- **Cons**: Slightly more schema than Option 1. Three values today
  rather than a boolean; an eventual fourth or fifth doesn't fit
  as cleanly as a straight bit-vector.

### Option 3 — Capability flags

Each Division carries a struct of booleans: `HasRosters`,
`HasLogos`, `HasPerPlayerStats`, `HasFixtures`, etc. SPA branches
on each flag independently.

- **Pros**: Maximum expressiveness. Any combination representable.
- **Cons**: Combinatorial surface area. Most combinations are
  nonsense (`HasPerPlayerStats=true` + `HasRosters=false` is
  incoherent). The SPA would pay the complexity on every tab
  without a corresponding product benefit — today's three real
  shapes (`espn-full` / `wikipedia-standings` / `licensed`) are
  more usefully named directly than built from flags.

## Decision

**Option 2.** Add a `DataFidelity` enum to `internal/domain/`
with three values:

```go
type DataFidelity string

const (
    // FidelityESPNFull is the default: standings, fixtures,
    // teams list, team logos, and identity-level rosters.
    // Every ESPN-sourced league ships at this tier.
    FidelityESPNFull DataFidelity = "espn-full"

    // FidelityWikipediaStandings is the ESPN-less degraded tier:
    // standings + per-round results scraped from Wikipedia.
    // No team logos, no rosters, no identity-level athlete data.
    // Requires a CC-BY-SA credit line on surfaces that render it.
    FidelityWikipediaStandings DataFidelity = "wikipedia-standings"

    // FidelityLicensed is the richer-than-ESPN tier reserved for
    // leagues on a licensed Opta / Stats Perform feed (NWSL
    // today, via internal/standings/adapters/nwslsdp). Adds
    // per-player season statistics, per-match event data, and
    // broadcast/attendance metadata beyond what ESPN exposes.
    FidelityLicensed DataFidelity = "licensed"
)
```

A `divisionDataFidelity` map catalogs each Division's tier; a
`DataFidelityFor(Division) DataFidelity` accessor returns the
value with a documented default (`FidelityESPNFull`) for any
Division not explicitly listed. Same shape as `divisionTiers`
(ADR-0002) and `continentalCompetitions` (ADR-0003).

The standings service exposes the fidelity on the per-division
response envelope (same place `sport`, `division`, `fieldSize`
live today). The SPA mirrors the enum in a TypeScript module
(`web/src/dataFidelity.ts`) and consumes a per-tier surface
config.

NCAA divisions (D1/D2/D3) are deliberately exempt from the
catalog. The fidelity enum is a pro-tier metadata concept — NCAA
tiers surface a different set of pages (RPI, bracket, etc.) that
have their own invariants independent of this axis. The accessor
returns the `FidelityESPNFull` default for NCAA calls because the
NCAA.com scoreboard + school-site roster pipeline lands at that
tier equivalent (logos + identity rosters + standings).

## Rationale

### Why a named enum, not a boolean

The forcing question: can a boolean carry the three meaningful
distinctions cutline already has today?

- ESPN-sourced (14 leagues shipped) → full standings + full
  rosters + logos
- Wikipedia-only (3 blocked leagues) → standings only, no roster,
  no logo
- Licensed (NWSL today) → ESPN-full + per-player season stats

Any two of those collapse into the boolean cleanly; all three do
not. A named enum captures the third distinction without
reinventing the shape when the licensed adapter ships.

### Why not capability flags

Capability flags give us three surface booleans
(`HasLogos`, `HasRosters`, `HasPerPlayerStats`) that would exactly
encode the three tiers above. They also permit 2^3 = 8
combinations, five of which are incoherent. Every SPA reader
would then branch on three flags rather than one enum value, and
the documentation would need to describe each incoherent
combination as "do not use." The enum removes the cost of
naming the valid combinations once rather than at every reader.

### Why the fidelity axis lives separately from the existing tier catalog

`internal/domain/division_tier.go` (ADR-0002) names a different
tier: country × pyramid position. Premier League is `Country: eng,
Tier: 1`; EFL Championship is `Country: eng, Tier: 2`. That
metadata answers "where does this league sit in its national
pyramid?"

`DataFidelity` answers "how much can we tell the user about this
league?" Two completely different questions. Combining them into
one field mixes axes the reader has to disentangle at every lookup.

Separate catalogs, keyed on the same `Division` type. Follows the
proSoccerLeagueSport + divisionTiers + continentalCompetitions
precedent: each catalog answers one question; readers consult the
catalog that matches their question.

### Why the SPA mirrors instead of fetching

The SPA's per-tab render decisions (which columns to draw, which
tabs to hide) need to be known at component-render time without
waiting for an API round-trip. The `proSoccerLeagueSport` map is
already mirrored this way (`web/src/proSoccerLeagues.ts`); the
same mirror pattern applies here. The handler exposes the value
on the per-division response envelope as a backup signal — a
future SPA component that needs tier-aware behavior based on a
URL-derived division can read it from the response rather than
knowing the enum values at compile time.

### Why NCAA is exempt from the catalog, not assigned `FidelityESPNFull`

`FidelityESPNFull` is a pro-league concept. NCAA tiers ride a
different data pipeline (NCAA.com scoreboard + school-site
rosters) and surface different pages (RPI tables, committee
bracket, selection modes). Assigning an NCAA division to any
value of the fidelity enum would imply a comparison against
pro-tier leagues the NCAA tiers don't share a surface with. The
accessor's default-return pattern is honest about this: a reader
asking "what fidelity does D1 ship at?" gets the default (the
same tier value the equivalent pro-league path renders) without
polluting the catalog.

## Implementation plan

### What lands immediately (1 cutline PR)

1. `internal/domain/data_fidelity.go` — new catalog file.
   - `DataFidelity` string type + three constants.
   - `divisionDataFidelity` map populated for every currently-
     shipped pro-soccer Division (14 at `espn-full`; NWSL at
     `licensed` because `nwslsdp` is in production today — this
     is the first time the distinction surfaces).
   - `DataFidelityFor(Division) DataFidelity` accessor; returns
     `FidelityESPNFull` for Divisions not in the map.
2. `internal/standings/http/handler.go` — add `dataFidelity` to
   the standings response envelope. Zero-value is the accessor's
   `FidelityESPNFull` so existing callers see no behavioural
   change.
3. `internal/catalog/http/handler.go` — same addition on the
   `/v1/divisions` response.
4. Tests — pin the catalog contents (every pro-soccer Division
   has a fidelity assignment; the three ESPN-less Divisions do
   not exist yet so the Wikipedia-tier value appears in the next
   PR; NWSL is assigned `licensed`).
5. SPA mirror — `web/src/dataFidelity.ts` with the three enum
   values + a `surfaceConfig(DataFidelity) → { showRosterTab,
   showTeamLogoColumn, creditLine }` function.

### What lands with the Wikipedia-adapter PR

1. `internal/standings/adapters/wikipedia/` — one general adapter
   driven by a `wiki_leagues.yaml` seed. Each row: league slug,
   Wikipedia season-page URL pattern, which wikitable index on
   the page, which column indexes map to GP/W/D/L/GF/GA/Pts.
2. `internal/domain/data_fidelity.go` catalog updated: Frauen-
   Bundesliga, Serie A Femminile, Damallsvenskan tagged
   `FidelityWikipediaStandings`.
3. SPA — Standings tab hides the team-logo column and renders
   the CC-BY-SA credit line for `wikipedia-standings` divisions;
   Team page hides the Roster tab for the same.
4. Per-league `Division` constants added (`frauen-bundesliga`,
   `serie-a-fem`, `damallsvenskan`) and wired through the ESPN
   adapter shape (even though the data comes from Wikipedia —
   the adapter dispatch happens at the handler on the fidelity
   flag).

### Deferred

- Licensed-adapter author documentation: a sibling ADR when a
  second licensed league ships beyond NWSL.
- Per-user tier preferences (hiding Wikipedia-tier leagues in the
  nav): no committed product surface asks for this today.

## Appendix: references

- Catalog-in-`internal/domain/` precedent: `pro_soccer.go`,
  `espn_slug_override.go`, `division_tier.go`,
  `continental.go`.
- SPA mirror precedent: `web/src/proSoccerLeagues.ts`.
- Issue #503 — alternate free data-source investigation that
  motivated this ADR. Three parallel investigation agents on
  2026-11-17 confirmed Wikipedia is the only universal free
  source for the three ESPN-less leagues.
- ADR-0002 — `DivisionTier` catalog for country/pyramid position.
- ADR-0003 — UEFA continental overlay (participants-in-two-
  divisions pattern, separate from this fidelity axis).
