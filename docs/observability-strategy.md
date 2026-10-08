# Observability Strategy

How metrics, logs, and traces reach Grafana (and any other OTLP
backend) consistently across every Go service in the fleet. Companion
to [`observability.md`](observability.md) — this doc covers *what to
instrument and how*, that one covers *dashboards built on top*.

## Goal

Every Go service in the jedi-knights fleet emits all three
OpenTelemetry signals through one shared package, one HTTP middleware
chain, and one deployment-config pattern:

1. **Metrics** — OTel SDK → Prometheus exporter → `/metrics` →
   Fly's hosted Prometheus scrapes via `[metrics]` in `fly.toml`.
2. **Traces** — OTel SDK → OTLP gRPC → an OTLP backend (Grafana Cloud
   Tempo, self-hosted Tempo, Honeycomb — one choice fleet-wide).
3. **Logs** — `log/slog` + `otelslog` handler bridge → OTLP (same
   endpoint as traces) → Loki or equivalent.

The services do not pick exporters or protocols individually. They
call one function from `go-platform/otel` and get all three signals
wired, with consistent resource attributes
(`service.name`, `service.version`, `deployment.environment`) and
automatic trace↔log correlation.

## Non-goals

- **Picking a specific OTLP backend.** The design is backend-agnostic
  — flip `OTEL_EXPORTER_OTLP_ENDPOINT` to point at Grafana Cloud,
  Honeycomb, Datadog, or a self-hosted collector. The recommendation
  (§ Backend selection) is Grafana Cloud free tier for the first
  deploy, but nothing in the code model is coupled to that choice.
- **Replacing Fly's hosted Grafana datasource.** Fly's hosted
  Prometheus continues to scrape `/metrics`. The OTel work adds a
  second metric path (OTLP push to the backend) only if the chosen
  backend needs it — Grafana Cloud does accept OTLP metrics; the
  Prometheus scrape is still the lowest-friction option and the one
  we standardize on.
- **Migrating Python services (jk-mcp-ecnl, jk-mcp-nwsl, …) in this
  pass.** They need the same treatment eventually, but this document
  is scoped to the Go fleet.

## Current state (audit 2026-10-08)

| Component | Traces | Metrics | Logs | Scrape config |
|---|---|---|---|---|
| `go-platform/otel.Init` | ✅ OTLP (gRPC+HTTP, stdout) | ❌ | ❌ | — |
| `go-platform/httputil` | ⚠️ custom UUID trace-id (not OTel) | ❌ | ✅ slog JSON | — |
| `auth-server` + 6 sibling identity services | ✅ `otelhttp.NewHandler` + go-platform tracer | ❌ | ✅ slog | ❌ `[metrics]` missing |
| `api-gateway` | ✅ `otelhttp` via container | ⚠️ raw `prometheus/client_golang` (not OTel) | ✅ slog | ❌ `[metrics]` missing |
| `jk-metering` + `jk-metering-ingest` | ❌ | ❌ | ✅ slog direct | ❌ |

**Key findings:**

- Tracing is more complete than it looked at first — every identity
  service already wraps its mux with `otelhttp.NewHandler`. The hard
  part (propagator config, resource attributes, OTLP exporter) lives
  in `go-platform/otel/tracer.go`.
- Two parallel trace-id systems coexist in request flow:
  `go-platform/httputil.TraceIDMiddleware` injects a custom UUID into
  `context.Context` and emits it in the `X-Trace-ID` response header,
  while `otelhttp` separately injects the OTel span's trace-id. Logs
  carry the former; distributed traces carry the latter. **This is
  the biggest latent bug**: a trace-id in a log line does not match
  the trace-id a developer looking at the span would use to find the
  log.
- `api-gateway` metrics never reach Grafana today because no `fly.toml`
  anywhere in the fleet declares a `[metrics]` section.
- `go-platform/otel/attrs.go` already defines the portfolio's
  semantic-convention vocabulary (actor, resource, action, decision,
  audit links) — any new instrumentation uses this vocabulary, not
  ad-hoc attribute names.

## Target architecture

```
┌─────────────────────────── identity service (any) ──────────────────────────┐
│                                                                             │
│   main.go                                                                   │
│     ├─ obs, _ := platformotel.Init(ctx, platformotel.Config{…})             │
│     │    returns { Tracer, Meter, Logger, Shutdown }                        │
│     └─ defer obs.Shutdown(ctx)                                              │
│                                                                             │
│   HTTP mux                                                                  │
│     ├─ otelhttp.NewHandler (traces + http.server.* metrics via same SDK)    │
│     ├─ httputil.LoggingMiddleware (slog + span-aware trace_id)              │
│     └─ /metrics ← promhttp.Handler on the exporter registry                 │
│                                                                             │
│   Domain code                                                               │
│     ├─ tracer.Start(ctx, "name") → spans                                    │
│     ├─ meter.Int64Counter("auth.tokens_issued").Add(ctx, 1, attrs…)         │
│     └─ slog.InfoContext(ctx, …) → log record carrying span's trace_id       │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
         ┌────────────────────────────┤
         │  /metrics (Prometheus)     │  OTLP gRPC (traces + logs)
         │                            │
         ▼                            ▼
   Fly hosted Prometheus         OTLP backend (Grafana Cloud / Honeycomb /
   → Fly Grafana                 self-hosted collector → Tempo + Loki)
                                      │
                                      ▼
                                 Grafana (unified view)
```

## Design decisions

### 1. One SDK for all three signals — OTel end-to-end

Rejected: Prometheus client for metrics + OTel for traces + raw slog
for logs. That's the current partial state and it's where
correlation goes wrong. **A single SDK means one resource attribute
set (`service.name`, `service.version`, `service.instance.id`,
`deployment.environment`) applies uniformly to every signal, so a
Grafana query that filters on `service.name=auth-server` returns a
coherent set of logs/traces/metrics for that service.**

### 2. Prometheus exporter for metrics (not OTLP push)

OTel metrics can either expose a `/metrics` endpoint
(`go.opentelemetry.io/otel/exporters/prometheus`) or push via OTLP.
Prometheus-exporter + Fly's hosted scrape wins because:

- Zero new infrastructure. Fly already has a Prometheus scraping
  every app with `[metrics]` declared.
- Backend-agnostic: Grafana Cloud, self-hosted Prometheus, and
  Datadog can all scrape a Prometheus endpoint.
- Push mode (OTLP metrics) adds a collector-or-direct-push question
  that we can revisit once there's a specific need.

The identity-service tier is a classic scrape target (long-lived HTTP
services with stable addresses). Push mode is only materially better
for short-lived jobs — which `jk-metering-worker` is close to being,
but it already runs as a long-lived polling worker, so scrape is
still fine.

### 3. OTLP gRPC for traces and logs (push)

Traces and logs are inherently push-shaped (span lifecycle, log
record stream). OTLP gRPC is more efficient than HTTP for high-volume
spans, and every supported backend accepts it.

### 4. Collapse the two trace-id systems

**Keep the OTel span's trace-id. Delete the custom UUID
trace-id in `httputil/middleware.go`.** The `X-Trace-ID` response
header, when set, is populated from the OTel span's `TraceID()` so
operators can still copy a trace-id out of a response header. A new
`httputil.LoggingMiddleware` enriches every slog record with the
OTel `trace_id` and `span_id` from `trace.SpanContextFromContext` —
or the `otelslog` handler bridge does this automatically.

**Why not keep both:** two trace-ids per request means operators
have to know which one to search for in which tool, and every log
line falsely implies a trace exists at a trace-id that the trace
backend has never heard of.

### 5. One `[metrics]` port, one `/metrics` path, fleet-wide

Every `fly.<service>.toml` gets:

```toml
[metrics]
  port = 9464        # OTel Prometheus exporter default; see §Port
  path = "/metrics"
```

The exporter runs on its own listener (`:9464`), separate from the
main HTTP service port. This isolates `/metrics` from the service's
auth/CORS/rate-limit middleware and prevents accidental exposure of
service endpoints on the scrape path.

Standard metrics exported by every service by default (via
`otelhttp` + the HTTP middleware chain):

- `http.server.request.duration` (histogram, seconds)
- `http.server.active_requests` (gauge)
- `http.server.request.body.size` (histogram, bytes)
- `http.server.response.body.size` (histogram, bytes)
- Standard Go runtime metrics (`runtime.go.*`) via
  `go.opentelemetry.io/contrib/instrumentation/runtime`

Services add domain-specific counters/histograms at the use-case
boundary (not inside the use case) — see § Service checklist.

### 6. Shared package expansion, not per-service rewrite

The `observability.Setup` stub in every service today is a strong
sign the pattern should move into `go-platform`. Target: one call to
`platformotel.Init` returns a struct carrying `Tracer`, `Meter`,
`Logger`, `Shutdown`, and the service never touches an exporter or
SDK type directly.

## Shared package design

Expand `go-platform/otel` from today's tracer-only shape to a
complete observability package. New files (names illustrative):

```
go-platform/
├── otel/
│   ├── tracer.go          (existing — unchanged public API)
│   ├── meter.go           (new — Prometheus exporter + meter provider)
│   ├── logger.go          (new — slog handler + otelslog bridge)
│   ├── attrs.go           (existing — portfolio attribute vocabulary)
│   ├── obs.go             (new — Init() returning composite handle)
│   └── doc.go
├── httputil/
│   ├── middleware.go      (changed — LoggingMiddleware span-aware;
│   │                       TraceIDMiddleware removed or thin shim)
│   ├── metrics.go         (new — wrapper that mounts /metrics via
│   │                       the Prometheus exporter)
│   └── ...
```

**Public API** (shape, not a contract — adjust during implementation):

```go
// go-platform/otel/obs.go
type Config struct {
    ServiceName        string   // required; populates service.name
    ServiceVersion     string   // populates service.version
    Environment        string   // populates deployment.environment
    OTLPEndpoint       string   // defaults to OTEL_EXPORTER_OTLP_ENDPOINT
    OTLPProtocol       string   // "grpc" (default) | "http/protobuf"
    OTLPInsecure       bool
    SamplerRatio       float64  // 0.0..1.0 for parent-based ratio sampler
    MetricsListenAddr  string   // defaults ":9464"; "" disables metrics server
    LogLevel           slog.Level
    LogFormat          string   // "json" (default) | "text"
}

type Observability struct {
    Tracer    trace.Tracer
    Meter     metric.Meter
    Logger    *slog.Logger
    Shutdown  func(ctx context.Context) error
}

func Init(ctx context.Context, cfg Config) (*Observability, error)
```

**Service-side usage reduces to:**

```go
// main.go
obs, err := platformotel.Init(ctx, platformotel.Config{
    ServiceName:    cfg.Service.Name,
    ServiceVersion: version.Build,
    Environment:    cfg.Env,
})
if err != nil { log.Fatal(err) }
defer obs.Shutdown(context.Background())

mux := http.NewServeMux()
// ... register routes ...
mux.Handle("/", otelhttp.NewHandler(appHandler, cfg.Service.Name))

httputil.ServeMetrics(obs, cfg.Service.MetricsAddr) // starts :9464
```

The current per-service `observability/setup.go` either becomes a
thin wrapper that calls `platformotel.Init` and returns `*Observability`,
or is deleted entirely in favor of calling `platformotel.Init`
directly from `main.go`. Either way, the "future tracer/meter setup
may report errors" TODO vanishes.

## Service checklist

Every Go service's `main.go` must look structurally identical for
the observability section:

1. Call `platformotel.Init(ctx, cfg)` before any domain code runs.
2. `defer obs.Shutdown(context.Background())` with a timeout.
3. Wrap the HTTP mux with `otelhttp.NewHandler(mux, serviceName)`.
4. Chain `httputil.LoggingMiddleware` and `httputil.RecoveryMiddleware`
   from the shared library.
5. Start the metrics server via `httputil.ServeMetrics` on `:9464`.
6. Declare `[metrics] port=9464 path="/metrics"` in `fly.<svc>.toml`.
7. Register the service's domain-specific OTel instruments at the
   use-case boundary — adapter layer, never inside the use case
   itself. Metric names follow OTel semantic conventions where one
   exists, portfolio conventions (`actor.*`, `resource.*`, `decision`,
   `audit.*` from `otel/attrs.go`) otherwise.

Services that are *not* HTTP servers (workers, batch jobs — `jk-metering`
today):

- Steps 3–4 skip (no HTTP mux).
- Step 5 still applies — a worker still exposes a metrics listener;
  it's often the only observable surface besides logs.
- Replace step 1 with explicit `tracer.Start` wrapping the main
  work-loop iteration so each polling cycle has a parent span.

### Per-service instrument catalog (starter set)

The catalog below is a minimum; services may add more. Names follow
`<domain>.<entity>.<action>` snake-case, with units in the suffix
when ambiguous. Attributes live in the OTel span and metric
attribute set, not in metric names.

| Service | Metric | Type | Attributes |
|---|---|---|---|
| `auth-server` | `oauth.tokens_issued` | counter | `grant_type`, `client_id`, `scope_count` |
| `auth-server` | `oauth.token_validation` | counter | `active` (bool), `reason` on failure |
| `auth-server` | `oauth.authorize_requests` | counter | `response_type`, `prompt` |
| `identity-service` | `identity.registrations` | counter | `result` ("ok" / "duplicate" / "invalid") |
| `identity-service` | `identity.password_verifications` | counter | `result` ("match" / "mismatch" / "unknown_user") |
| `client-registry-service` | `oauth.clients` | up-down counter | — (gauge of registered clients) |
| `client-registry-service` | `oauth.client_lookups` | counter | `result` ("hit" / "miss") |
| `token-introspection-service` | `oauth.introspection.requests` | counter | `active`, `token_type_hint` |
| `authorization-policy-service` | `authz.decisions` | counter | `decision` ("allow" / "deny"), `resource_kind`, `action` |
| `authorization-policy-service` | `authz.decision_duration` | histogram (ms) | `decision`, `cache` ("hit" / "miss") |
| `login-ui` | `login.attempts` | counter | `result` ("ok" / "bad_password" / "unknown_user" / "locked") |
| `login-ui` | `consent.decisions` | counter | `decision` ("approve" / "deny") |
| `api-gateway` | `gateway.requests` | counter | `route`, `status` (preserve existing name as `gateway_requests_total` in Prometheus exposition) |
| `api-gateway` | `gateway.request_duration` | histogram (ms) | `route` (preserve as `gateway_request_duration_ms`) |
| `jk-metering-worker` | `metering.events_drained` | counter | `result` ("ok" / "error") |
| `jk-metering-worker` | `metering.lago_push_duration` | histogram (ms) | `status_class` |
| `jk-metering-ingest` | `metering.events_accepted` | counter | `event_code` |

These are starter names — the owning service's ADR or module doc is
the source of truth once the service lands them. The point of
listing them here is to establish the *vocabulary* before the first
PR goes in.

## Deployment configuration

### fly.toml additions per service

```toml
# fly.auth-server.toml (and every other service)

[metrics]
  port = 9464
  path = "/metrics"

[env]
  OTEL_SERVICE_NAME       = "auth-server"
  OTEL_SERVICE_VERSION    = "@@VERSION@@"      # replaced by deploy workflow
  OTEL_RESOURCE_ATTRIBUTES = "deployment.environment=prod,service.namespace=identity-platform"
  OTEL_EXPORTER_OTLP_PROTOCOL = "grpc"
  # OTEL_EXPORTER_OTLP_ENDPOINT comes from fly secrets
```

### Fly secrets (one per service, same value)

```bash
fly secrets set \
  OTEL_EXPORTER_OTLP_ENDPOINT="https://otlp-gateway-prod-us-east-0.grafana.net/otlp" \
  OTEL_EXPORTER_OTLP_HEADERS="Authorization=Basic <base64(user:apikey)>" \
  -a auth-server
```

(Repeat for each app. A small shell helper in the identity-platform-go
Makefile is appropriate.)

### Port 9464

9464 is OTel's conventional default for the Prometheus exporter.
Standardizing on it means every `[metrics]` section is identical,
which pays off every time someone writes a new service or audits
the fleet.

## Backend selection — recommendation

**For the first deploy: Grafana Cloud free tier.**

- Already paying for Grafana (Fly bills it); upgrading to Cloud is a
  short step.
- Managed Tempo (traces) + Loki (logs) + Prometheus-compatible
  metrics endpoint. Everything in one Grafana.
- OTLP gRPC endpoint accepts all three signals.
- Free tier limits (as of 2026-10): 50 GB logs, 50 GB traces, 10k
  active metrics series — comfortably over the fleet's current
  footprint.

Alternatives if the fleet outgrows free tier or the user prefers
otherwise:

- **Self-hosted OTel Collector on Fly** (`jk-otel-collector` app)
  that fans out to Tempo + Loki + Prometheus on Fly. More control,
  more operational burden.
- **Honeycomb** — strong trace query experience, less first-class
  logs/metrics story.

Backend selection is a one-env-var change in production; the code
model doesn't care.

## Phased rollout

**Phase A — plumbing (go-platform)**. One PR against go-platform.

- [ ] Add `meter.go` with Prometheus exporter + meter provider.
- [ ] Add `logger.go` with `otelslog` handler bridge + slog
      configuration.
- [ ] Add `obs.go` with the composite `Init` + `Observability` type.
- [ ] Rewrite `httputil/middleware.go.LoggingMiddleware` to pull
      `trace_id` / `span_id` from the OTel span context, not from
      the custom UUID system.
- [ ] Delete `httputil/middleware.go.TraceIDMiddleware` (or shrink
      to a shim that writes the OTel trace-id to `X-Trace-ID`).
- [ ] Add `httputil.ServeMetrics` convenience that mounts the
      Prometheus handler on a separate listener.
- [ ] Add runtime instrumentation via
      `go.opentelemetry.io/contrib/instrumentation/runtime`.

**Phase B — fleet migration**. One PR per repo (parallelizable).

- [ ] `api-gateway`: migrate `prometheus/client_golang` to OTel SDK
      with the Prometheus exporter. **Preserve the existing metric
      names** (`gateway_requests_total`, `gateway_request_duration_ms`)
      via explicit instrument names so Grafana panels built against
      them survive the migration unchanged. Add `[metrics]` to
      fly.toml.
- [ ] `identity-platform-go` services (7 PRs or one umbrella):
      replace `observability.Setup` with `platformotel.Init`; add
      `[metrics]` to each `fly.<svc>.toml`; wire
      `httputil.ServeMetrics`; register the starter instruments from
      the catalog above.
- [ ] `jk-metering` + `jk-metering-ingest`: add `platformotel.Init`
      to `cmd/*/main.go`; add `[metrics]` + env vars to both
      `fly.toml` and `fly.ingest.toml`; register the metering
      instruments.

**Phase C — Grafana surface**. Follow-up docs work in this repo.

- [ ] Extend [`observability.md`](observability.md) Dashboard 1 with
      a new row group for `gateway.requests` and
      `http.server.request.duration{service=…}`.
- [ ] Build Dashboard 2 (OAuth/OIDC Flows) using the service-level
      counters from the catalog.
- [ ] Build Dashboard 3 (Authorization + Entitlements) from
      `authz.decisions` and metering counters.

## Open questions (tracked, not blocking)

1. **Does the api-gateway migration preserve the gateway's isolated
   Prometheus registry pattern?** Current code uses a per-instance
   registry, not the global default. OTel's Prometheus exporter
   defaults to a dedicated registry too, but confirm during
   implementation that the migration path is clean.
2. **Entitlements-service was in the audit (not CLAUDE.md).** Needs
   adding to the service inventory before Phase B.
3. **Python MCP servers** (`jk-mcp-ecnl`, `jk-mcp-nwsl`) are out of
   scope for this doc. The equivalent story — OpenTelemetry Python +
   `opentelemetry-exporter-prometheus` — fits the same shape and will
   get its own strategy doc when the Go rollout stabilizes.
4. **Sampler ratio** — current tracer defaults to `AlwaysSample`.
   100% sampling is right during Phase A/B rollout; revisit after
   production traffic is visible in Tempo and the storage bill is
   knowable.

## Cross-references

- [`observability.md`](observability.md) — Grafana dashboards built
  on the metrics this strategy produces.
- [`cross-cutting.md`](cross-cutting.md) §Agent identity and tool
  authorization — the audit event schema that complements OTel
  traces.
- [`go-platform.md`](go-platform.md) — the shared library this
  strategy expands.
- [`identity-platform-go.md`](identity-platform-go.md) — the primary
  consumer of the strategy.
