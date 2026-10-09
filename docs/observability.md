# observability

Grafana dashboards and metrics instrumentation for the identity
platform on Fly.io's hosted Grafana. Dashboards are the source of
truth here; copy the JSON into the identity-platform-go repo's
`ops/grafana/` directory when wiring up the deploy workflow.

**For the fleet-wide OpenTelemetry strategy** (how metrics, logs,
and traces get wired into every service consistently) see
[`observability-strategy.md`](observability-strategy.md). This
document covers dashboards; that one covers instrumentation.

## Current state (2026-10-08)

Audited the Go monorepo at `jedi-knights/identity-platform-go` for
Prometheus exposure:

| Service | `/metrics` | fly.toml `[metrics]` | Custom metrics |
|---|---|---|---|
| `jk-api-gateway` | ✅ | ❌ **missing** | `gateway_requests_total`, `gateway_request_duration_ms` |
| `jk-auth-server` | ❌ | ❌ | — (OTel traces only) |
| `jk-identity-service` | ❌ | ❌ | — |
| `jk-client-registry-service` | ❌ | ❌ | — |
| `jk-token-introspection-service` | ❌ | ❌ | — |
| `jk-authorization-policy-service` | ❌ | ❌ | — |
| `jk-example-resource-service` | ❌ | ❌ | — (no OTel either) |
| `jk-login-ui` | ❌ | ❌ | — |

**Implication:** today Grafana only has access to Fly's built-in
infra metrics (`fly_instance_*`, `fly_edge_*`, `fly_app_concurrency`,
`fly_app_http_responses_count`, `fly_app_tcp_connects_count`). No
app-level counters for token issuance, login attempts, policy
decisions, etc. are visible yet.

## Dashboards

### 1. Identity Platform — Fly Infra Overview

**File:** [`observability/jk-identity-fly-infra.json`](observability/jk-identity-fly-infra.json)
**UID:** `jk-identity-fly-infra`

Operates entirely on Fly's hosted Prometheus. Zero app-level
instrumentation required. Covers the full identity platform plus the
gateway. Row layout:

| Row | Panels | Metric source |
|---|---|---|
| Service Health | running instances, instances down (red if > 0), instance count over time | `fly_instance_up` |
| HTTP Traffic — Public Edge | req/s by status, 5xx error ratio, p50/p95/p99 latency | `fly_edge_http_responses_count`, `fly_edge_http_response_time_seconds_bucket` (jk-api-gateway only) |
| HTTP Traffic — Per Service | req/s per app, 5xx per app, concurrent requests per app | `fly_app_http_responses_count`, `fly_app_concurrency` |
| Compute | CPU %, memory %, load avg (1m), memory bytes used | `fly_instance_cpu`, `fly_instance_memory_*`, `fly_instance_load_average` |
| Network | RX bytes/sec, TX bytes/sec per service | `fly_instance_net_recv_bytes`, `fly_instance_net_sent_bytes` |

**Variables:** `datasource` (any Prometheus datasource), `app`
(multi-select; defaults to all identity-platform apps via regex),
`region` (multi-select).

**Import:** Grafana → Dashboards → New → Import → upload JSON →
select the hosted Fly Prometheus datasource.

**Known limitation:** `fly_app_http_responses_count` only exists for
apps declared as `http` service type in `fly.toml`. Internal services
exposed only on `.internal` without an http service block will show
blank on the per-service HTTP row — the Compute/Network/Service
Health rows still work.

### 2. Identity Platform — OAuth/OIDC Flows (deferred)

Planned content: token issuance by grant type, login success/fail,
introspection volume, JWKS cache hits, authz allow/deny rate per
resource.

**Blocked on:** no identity-platform-go service currently exposes
app-level metrics. Shipping this dashboard today would be a wall of
`No data` panels. Build after the Phase A instrumentation below
lands.

## Phase A — instrumentation prerequisites

Before dashboard 2 is worth building. See
[`observability-strategy.md`](observability-strategy.md) §Phased
rollout for the full plan; the summary below is kept in sync with
it:

1. **Wire scraping for `jk-api-gateway`.** The `/metrics` endpoint
   already exists at
   [`internal/adapters/inbound/http/routes.go:90`](https://github.com/jedi-knights/api-gateway/blob/main/internal/adapters/inbound/http/routes.go)
   but `fly.toml` has no `[metrics]` section — Fly's hosted
   Prometheus never discovers it. One-commit fix:
   ```toml
   # fly.toml
   [metrics]
     port = 8080         # or whichever the metrics handler is on
     path = "/metrics"
   ```
   After deploy, `gateway_requests_total` and
   `gateway_request_duration_ms` land in Grafana. Dashboard 1 does
   not yet query these — add a dedicated gateway-metrics row once
   scraping is confirmed working.
2. **Choose a metrics SDK for the 7 identity services.** Either
   `prometheus/client_golang` (direct, simpler) or OTel metrics →
   Prometheus exporter (composes with the OTel tracing already in
   `go-platform/otel/tracer.go`). Decision belongs in a `go-platform`
   ADR, not here.
3. **Add a shared `metrics` package to
   [`go-platform`](go-platform.md)** mirroring the tracer setup, plus
   an HTTP middleware that exports `http_request_duration_seconds` /
   `http_requests_total` with the standard `method`/`route`/`status`
   labels.
4. **Register domain-specific counters per service** at the hexagonal
   boundary (not inside the use case) — e.g., `auth-server` counters
   for `oauth_tokens_issued_total{grant_type, client_id}` and
   `oauth_token_validation_total{active}`, `authz-policy` for
   `authz_decisions_total{decision, resource_type}`, `login-ui` for
   `login_attempts_total{result}`.
5. **Add `[metrics]` to every `fly.<service>.toml`.** Fly will not
   scrape a service that doesn't declare the port + path.

## Conventions

- Dashboard JSON uses `${datasource}` as the datasource variable so
  the same file imports cleanly against any Prometheus datasource —
  Fly hosted, Grafana Cloud, or a self-hosted scraper.
- Panel descriptions explain what the metric means and what a
  concerning value looks like. Keep them short — they render in
  Grafana's panel info popover.
- Service regex in the `app` variable enumerates every current + planned
  identity-platform app. New services added to the platform should be
  appended to the regex in the same PR that adds the service's
  `fly.<service>.toml`.
- Dashboard UIDs are prefixed `jk-identity-` so they sort together in
  the Grafana browser.
