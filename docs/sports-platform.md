# Sports Platform (Go Monorepo)

Status: Design approved 2026-10-29
Target repo: `github.com/jedi-knights/sports-platform`
Replaces: `college-sports-api`, `athletics-ingest-platform`, `wsoccer-stats`

## Goals

Three constraints:

1. **Well segmented** — each service owns one bounded context; no shared mutable state across services; writes come from exactly one place.
2. **Fast** — hot read paths (scoreboard, standings) stay sub-100ms via Redis cache in front of Postgres. The slow path (ingest) is off the request path entirely.
3. **Consistent** — one domain package, one middleware stack, one config loader, one logger, one DB migration tool.

## What the three legacy repos do today

| Repo | Role | Disposition |
|---|---|---|
| `college-sports-api` | Live NCAA.com scoreboard proxy. Go, stateless, Redis cache, sport-pluggable via container pattern, hexagonal, deployed on Fly.io. | Fold into `cmd/scoreboard/` with minimal changes. |
| `athletics-ingest-platform` | Python CLI that scrapes 321 D1 WSOC program sites across 4 CMSes (Sidearm, WMT, WordPress, Presto). Outputs NDJSON files. | Port parsers to Go. Promote to a scheduled worker. Write to Postgres instead of NDJSON (optional NDJSON side-effect for backup). |
| `wsoccer-stats` | Tauri desktop app. React UI + Rust RPI compute. Reads NDJSON from sibling dir. Pain points: no deep links, no auto-refresh, manual CSV copy, no box scores, 15s all-DI refresh. | Replace with a web UI served by the gateway. Port RPI from Rust → Go in the standings service. Tauri becomes optional desktop shell later. |

The two "ingest" repos pull from different sources:
- `college-sports-api` scrapes **NCAA.com** (national aggregator — today's live scores)
- `athletics-ingest-platform` scrapes **per-program official sites** (deep roster/schedule/stats history)

These are not the same job. Keep both ingestion paths; separate services own them.

## Service boundaries

Four long-running Go services + one scheduled worker, sharing one Postgres + one Redis.

```
┌─────────────────────────────────────────────────────────┐
│  Clients (web UI from gateway, Tauri shell, MCP, …)     │
└───────┬──────────────┬──────────────┬──────────────┬────┘
        │              │              │              │
        ▼              ▼              ▼              ▼
┌─────────────┐ ┌────────────┐ ┌────────────┐ ┌───────────┐
│ scoreboard  │ │  catalog   │ │ standings  │ │  gateway  │
│ (NCAA.com   │ │ (programs, │ │  (RPI +    │ │ (web UI + │
│  proxy;     │ │  teams,    │ │  table;    │ │  BFF)     │
│  Redis-only │ │  conf.)    │ │  Postgres  │ │           │
│  stateless) │ │  Postgres  │ │  reader)   │ │           │
└──────┬──────┘ └─────┬──────┘ └──────┬─────┘ └─────┬─────┘
       │              │               │             │
       └──────────────┴────────┬──────┴─────────────┘
                               ▼
                      ┌─────────────────┐
                      │   Postgres      │  ← rosters, schedules,
                      │   + Redis       │     box scores, stats
                      └────────▲────────┘
                               │ writes
                      ┌────────┴────────┐
                      │  ingest-worker  │  ← scheduled scraper
                      │  (CMS parsers)  │     fleet (Go port of
                      │                 │     athletics-ingest)
                      └─────────────────┘
```

### Per-service responsibilities

| Service | Hot path? | Reads from | Writes to | Owns |
|---|---|---|---|---|
| **scoreboard** | yes | NCAA.com (via proxy) | Redis (cache) | daily scoreboard scraper, cache keys, sport registry for live events |
| **catalog** | yes | Postgres | Postgres | programs, teams, conferences, aliases; URL-hint registry for ingest |
| **standings** | yes | Postgres, Redis | Redis (cache) | NCAA RPI formula, division standings, conference standings, head-to-head |
| **gateway** | yes | scoreboard/catalog/standings (HTTP) | — | web UI static assets, aggregation endpoints, (future) auth |
| **ingest-worker** | no | per-program sites, catalog (HTTP) | Postgres | CMS parsers (sidearm, wmt, wp, presto), fetcher, pipeline, incremental `--since` |

### Why not a different number of services

- **Not 1 or 2** — one service that owns live scraping *and* DB-backed queries violates "well segmented": a slow DB query would starve the live scoreboard path. Separate scoreboard preserves the stateless cheap-to-scale property it already has today.
- **Not 6+** — a separate `teams` or `rosters` service adds a hop with no boundary benefit; they're read-only slices of catalog data.
- **No standalone BFF gateway on day one if the web UI is simple.** Starting with gateway lets the frontend talk to one host. If it stays trivial, it can later demote to a static asset host.

## Day-one scope decisions

| Decision | Choice | Rationale |
|---|---|---|
| wsoccer-stats disposition | Web app served by gateway | Fixes deep-link, auto-refresh, CSV-export pain points for free. Tauri shell optional later. |
| Persistence | Postgres (source of record) + Redis (cache) | History, incremental refresh, box scores need a DB. NDJSON export stays as optional backup. |
| Sport scope | D1 women's soccer only (seed) | Everything else is empty rows in catalog + unregistered sport descriptors. |
| Architecture scope | Sport/division pluggable from day one | Matches college-sports-api's existing container pattern; cheap to preserve, expensive to retrofit. |

## Repo layout

```
sports-platform/
├── cmd/
│   ├── scoreboard/main.go
│   ├── catalog/main.go
│   ├── standings/main.go
│   ├── gateway/main.go
│   └── ingest-worker/main.go
├── internal/
│   ├── domain/              # Team, Game, Program, Player, Sport, Division (no external imports)
│   ├── scoreboard/          # app, adapters/scraper, adapters/http
│   ├── catalog/
│   ├── standings/
│   ├── gateway/
│   ├── ingest/
│   │   ├── parsers/         # sidearm, wmt, wp, presto, wmt_api, sidearm_boxscore
│   │   ├── fetcher/
│   │   └── pipeline/
│   └── platform/            # cross-cutting shared code
│       ├── httpserver/      # router, mw, health, openapi
│       ├── cache/           # redis + memory
│       ├── store/           # postgres (sqlc or pgx)
│       ├── config/          # viper
│       └── observability/   # slog, prom
├── api/                     # OpenAPI 3.1 per service
├── migrations/              # per-service SQL migrations
├── deploy/                  # Dockerfile per service, fly.toml, k8s (if applicable)
├── seed/                    # ex-YAML registries, loaded by catalog on first run
├── web/                     # wsoccer-stats replacement (React/Vite, built → gateway static)
├── go.mod
└── Taskfile.yml
```

**Single Go module** (not a workspace). Easier CI, one `go.sum`, no inter-module replace directives. Each service `cmd/<svc>/` depends on `internal/<svc>/` + `internal/domain/` + `internal/platform/`. No cross-service imports of other `internal/<svc>/` packages — services talk over HTTP, period.

## Shared domain (`internal/domain/`)

- `Sport` interface + `SportID` enum
- `Division` enum (`d1`, `d2`, `d3`)
- `Team`, `Program`, `Conference`
- `Game`, `GameState`, `GameSide`
- `Player`, `PlayerStats`

Rules:
- No imports except stdlib (`time` is OK).
- No I/O methods. These are value types.
- Changes here require review across every service — they're the shared contract.

## Ingest shape (Go port of athletics-ingest-platform)

Preserve the properties that make the Python version work:

- **Scheduled worker, not request-driven.** Cron (Fly cron or k8s CronJob).
- **Never-raising pipeline.** Per-program result struct with error field; one program's failure doesn't fail the batch.
- **CMS dispatch via `detect(html) → cmsID` + registry lookup.** Type-safe parser table.
- **Injectable fetcher.** Interface for the HTTP client so fixture-driven tests work.
- **Fixture-driven parser tests.** One test fixture per CMS variant; preserve the current `tests/fixtures/` layout.
- **ThreadPool → errgroup.** Bounded worker pool via `errgroup.Group` with `SetLimit`.

Changes from Python version:

- Writes to Postgres directly, not NDJSON. Optional NDJSON export as a side-effect for backup.
- `--since` filter for incremental refresh (addresses wsoccer-stats pain point: 15s → seconds).
- Catalog reads come from the catalog service over HTTP, not from local YAML. YAML seeds the catalog on first run.

Go libraries:
- HTML parsing: `github.com/PuerkitoBio/goquery` (jQuery-style, closest to BeautifulSoup).
- HTTP: stdlib `net/http` with injected `*http.Client`.
- Concurrency: `golang.org/x/sync/errgroup`.
- Regex (exhibition filter): stdlib `regexp`.

## Scoreboard shape (ex-college-sports-api)

Keep the hexagonal structure as-is. The port is close to a lift-and-shift:

- `internal/domain/` moves into the monorepo's shared `internal/domain/` (merging with ingest's domain types).
- `internal/ports/` → `internal/scoreboard/ports/`.
- `internal/adapters/inbound/http/` → `internal/scoreboard/adapters/http/`.
- `internal/adapters/outbound/scraper/` → `internal/scoreboard/adapters/scraper/`.
- `internal/adapters/outbound/cache/` → migrated to shared `internal/platform/cache/`.
- `internal/container/` → `internal/scoreboard/container/` (keep per-service).
- `internal/sports/soccer/` → `internal/scoreboard/sports/soccer/`.

Scoreboard **does not** depend on Postgres. It's a stateless NCAA.com proxy with Redis cache, same as today.

## Standings shape (new, replaces Rust RPI)

- Port NCAA RPI formula from `wsoccer-stats/src-tauri/src/standings.rs::compute_rpi_full`.
- Reads from Postgres via sqlc-generated queries (or pgx direct, TBD).
- Caches computed standings in Redis; invalidates on ingest-worker completion signal (webhook or Redis pub/sub).
- API: conference standings, division standings, team RPI, head-to-head.

RPI is the one piece of code where behavior parity with the Rust implementation matters most — snapshot tests against known good standings output are mandatory.

## Catalog shape (new)

- Replaces `conferences.yaml` + `programs/d1-women-soccer.yaml`.
- Seeded from `seed/d1-wsoc.yaml` on first run (idempotent).
- Writable via admin API — programs, aliases, URL hints can be edited without a deploy.
- Owned by catalog service. Ingest-worker reads via HTTP (not shared DB access).
- Low write volume, high read volume → heavily Redis-cached with short TTL.

## Gateway + web UI

- Serves the web UI (wsoccer-stats replacement) as static assets built from `web/`.
- Aggregates scoreboard + standings + catalog where the dashboard view needs multiple services.
- Deep-link friendly: all state in the URL (fixes wsoccer-stats pain point #1).
- Scheduled background refresh in the client (fixes pain point #2).
- CSV export client-side (fixes pain point #4).
- Future auth terminator lives here.

Web UI stack: React + Vite + TypeScript (port from wsoccer-stats). Routing via `react-router` for proper URL state. Optionally re-wrap in Tauri later for desktop install — the webapp is the source of truth, Tauri becomes a shell.

## Deployment

Target: **Fly.io** (continues college-sports-api's current model).

- One Fly app per service: `sports-scoreboard`, `sports-catalog`, `sports-standings`, `sports-gateway`, `sports-ingest`.
- Managed Postgres on Fly (shared across catalog, standings, ingest-worker). Scoreboard does **not** connect.
- Upstash Redis (college-sports-api's current). Shared across scoreboard (cache), standings (cache), catalog (cache).
- One CI pipeline, per-service matrix build + deploy.
- Per-service Dockerfile under `deploy/<svc>/Dockerfile` (multi-stage, distroless, static binary — college-sports-api's pattern).

## Open questions (decide before Phase 2)

- `go-platform` shared lib (used by college-sports-api) — adopt as a dep, or build fresh in `internal/platform/`?
- SQL layer: sqlc vs. pgx direct vs. GORM. Recommend sqlc for type safety + no runtime overhead.
- Migrations: goose vs. atlas. Recommend atlas (declarative + diffing against live schema).
- OpenAPI codegen: oapi-codegen vs. hand-written. Recommend oapi-codegen for both server stubs and typed clients that services use to call each other.
- Observability: Fly's built-in Prom + logs, or external (Honeycomb, Datadog)?

## Phases

### Phase 1 — scaffold + port scoreboard

- [x] Scaffold monorepo (go.mod, Taskfile, dir layout) — this commit
- [ ] Decide: `go-platform` shared lib vs. roll own
- [ ] Port `college-sports-api` into `cmd/scoreboard/` with minimal changes
- [ ] CI pipeline (lint, test, build matrix)
- [ ] Fly.io deploy for scoreboard

### Phase 2 — catalog + Postgres foundation

- [ ] Choose migrations tool (atlas vs. goose)
- [ ] Choose SQL layer (sqlc vs. pgx direct)
- [ ] Schema: programs, teams, conferences, aliases
- [ ] Seed loader from `seed/d1-wsoc.yaml`
- [ ] Catalog service: CRUD + read API
- [ ] Fly managed Postgres provisioning

### Phase 3 — ingest-worker

- [ ] Port parsers from Python → Go: sidearm, wmt, wordpress, presto
- [ ] Port wmt_api, sidearm_boxscore JSON API fallbacks
- [ ] Fetcher with injectable HTTP client
- [ ] Never-raising pipeline (per-program result struct)
- [ ] `--since` filter for incremental refresh
- [ ] Write to Postgres instead of NDJSON (optional NDJSON export)
- [ ] Scheduled run (Fly cron)
- [ ] Snapshot tests from existing fixture layout

### Phase 4 — standings service

- [ ] Port NCAA RPI formula from Rust → Go
- [ ] Postgres reader interface
- [ ] Redis cache + invalidation on ingest completion
- [ ] API: conference standings, division standings, team RPI, head-to-head
- [ ] Snapshot tests against known good standings

### Phase 5 — gateway + web UI

- [ ] Port wsoccer-stats React views to a web app in `web/`
- [ ] Deep links / URL state via react-router
- [ ] Scheduled background refresh in client
- [ ] CSV export
- [ ] Gateway serves static assets + aggregates from backend services
- [ ] Optional Tauri shell for desktop install
- [ ] Deprecate `wsoccer-stats` repo
