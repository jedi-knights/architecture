# ADR-0001: Shared HTTP middleware in `go-platform/httpmw`

- **Status:** Accepted (Tier 1 implemented; Tiers 2–3 pending)
- **Date:** 2026-10-08
- **Scope:** every Go HTTP service in the portfolio — the eight services in
  `identity-platform-go/services/` and `jk-metering` ingest.

## Context

All nine Go services build the same HTTP plumbing independently. An audit on
2026-10-08 found:

| Concern | State today |
|---|---|
| Recovery / logging / trace-ID chain | Copy-pasted into six `routes.go` files; `example-resource-service` orders it differently; `entitlements-service` and `jk-metering` have none |
| Request ID | `LoggingMiddleware` reads `RequestIDFromContext`, but nothing in any repo ever sets it — `request_id` is always `""` |
| `/health` handler | Re-defined in every service (`{"status":"ok"}`); `jk-metering` and `entitlements-service` hand-roll their own JSON writer |
| `http.Server` setup, graceful shutdown | Copy-pasted into every `main.go`; timeouts have drifted (four services lack `ReadHeaderTimeout`; `jk-metering` sets only that one and no shutdown-on-signal beyond its own goroutine) |
| Static-secret bearer check | Three copies (`auth-server` ×2, `token-introspection-service`) |
| Rate limiter, body-size cap, `Cache-Control: no-store` | Inline in `token-introspection-service` / `jk-metering` handlers |
| RS256/HS256/introspection bearer auth, scope/ACR/DPoP enforcement | `example-resource-service` and `jk-metering` implement the same flow twice with different context keys and hardcoded realms |

### Defects found, not just duplication

1. **`trace_id` is empty in the logs of six services.** Those services wire
   `Recovery(Logging(TraceID(mux)))`. `LoggingMiddleware` reads the trace ID
   from the request context *before* calling `next`; with `TraceID` innermost
   it is not yet set. `httputil`'s own docs require `TraceID` outermost, and
   only `example-resource-service` complies. `RecoveryMiddleware` logs
   `trace_id` the same way and has the same defect.
2. **`request_id` is never populated** (see above).
3. **`entitlements-service` has no panic recovery.** A handler panic is
   handled by `net/http`'s default recovery: unstructured log, dropped
   connection.

All three are consequences of ordering being a convention each service must
re-implement, not something the library enforces.

## Decision

Add a new package **`github.com/jedi-knights/go-platform/httpmw`**, delivered
in three tiers. Each tier ships as its own go-platform release.

`httpmw` is a new package rather than additions to `httputil` because Tier 3
(auth) depends on `jwtutil`, and `httputil`'s dependencies are deliberately
limited to `apperrors` (see go-platform `CLAUDE.md`: "each package is
independently usable").

**All middleware lives in `httpmw`; none is shared with `httputil`.** The
three existing middleware are moved, not wrapped, and renamed without the
`Middleware` suffix: `httputil.TraceIDMiddleware` → `httpmw.TraceID`,
`httputil.LoggingMiddleware` → `httpmw.Logging`,
`httputil.RecoveryMiddleware` → `httpmw.Recovery` (and the `Logger` alias).
`httputil` keeps only response helpers (`WriteJSON`, `WriteError`,
`HTTPStatus`); `httpmw` imports it for `WriteJSON` and never the reverse, so
there is no forwarding layer and no import cycle. This is a breaking change,
acceptable under go-platform's v0.x policy; consumers migrate when they bump
the module (see Rollout).

### Tier 1 — every service, no required configuration (this ADR's first delivery)

| Export | Purpose |
|---|---|
| `TraceID`, `Logging(logger)`, `Recovery(logger)` | Moved from `httputil` unchanged in behavior. |
| `Stack(logger, opts...) func(http.Handler) http.Handler` | Composes `RequestID → TraceID → Recovery → Logging` in the one correct order. Fixes defects 1–3 by making the order impossible to get wrong. |
| `RequestID` | Sets `X-Request-ID` and the context request ID. Inbound value is reused only if it is a canonical UUID v4 (same log-injection rule `TraceIDMiddleware` applies); otherwise a fresh one is generated. |
| `WithSkipLogPaths(paths ...string)` | `Stack` option. Exact-match paths bypass the access-log line (recovery, trace and request IDs still apply). Default: `/health`. Passing no arguments disables skipping. |
| `HealthHandler()` | Liveness: `200 {"status":"ok"}`. |
| `ReadyHandler(checks ...ReadyCheck)` | Readiness: runs checks with a bounded timeout; `200 {"status":"ok"}` or `503 {"status":"unavailable"}`. Never returns the failing check's error text. |
| `NewServer(addr, handler, opts...)` / `(*Server).Run(ctx)` / `Serve(ctx, ln)` | `http.Server` with standard timeouts (`ReadHeaderTimeout` 5s, `ReadTimeout` 15s, `WriteTimeout` 15s, `IdleTimeout` 60s), SIGINT/SIGTERM handling, and graceful shutdown (default 30s budget). Options: `WithReadHeaderTimeout`, `WithReadTimeout`, `WithWriteTimeout`, `WithIdleTimeout`, `WithShutdownTimeout`. `HTTPServer()` is an escape hatch for TLS config and similar. |

`otelhttp` stays in each service's `main.go`: pulling it into `httpmw` would
add a heavy dependency to every consumer, and the span-name formatter is
service-specific.

### Tier 2 — opt-in, per route or per service

`MaxBody(n)`, `StaticBearer(secret, opts...)` (constant-time), `RateLimit(limiter,
keyFunc)` with a bounded key count, `NoStore()`. Extracted from
`token-introspection-service`, `auth-server` and `jk-metering`.

### Tier 3 — `httpmw/auth`

`Bearer(verifier, opts...)` with `Verifier` implementations for RS256 (JWKS),
HS256, and RFC 7662 introspection; one canonical `Principal` carried via
`auth.FromContext(ctx)`; `RequireScope`, `RequireACR`, `RequireDPoP`.
Configurable axes, taken from where the services actually differ: realm
(`WithRealm`, required), error-body writer (`WithErrorWriter`; `apperrors`
default plus an OAuth `{"error","error_description"}` writer), audience and
issuer (empty disables the check, matching every current implementation), and
client-IP resolution for rate limiting (`WithClientIP`; default `RemoteAddr`,
with a trusted-proxy helper for Fly's `Fly-Client-IP`).

### Conventions for all tiers

- Functional options, matching idioms already used in go-platform.
- Constructors that take a required argument panic on a programmer error
  (empty scope, nil verifier) rather than returning a middleware that can
  never work — the behavior `RequireScopeMiddleware` has today.
- No new external dependencies. Only `go-logging`, `apperrors`, `jwtutil`
  and the standard library.
- Tests are black-box (`package httpmw_test`) through the public surface, with
  an explicit ordering test for `Stack` (the defect above) and an injected
  clock for anything time-based.

## Consequences

**Positive**

- Six services regain populated `trace_id` in access and panic logs; all
  services gain `request_id`; `entitlements-service` and `jk-metering` gain
  recovery and structured access logs.
- One place to change timeouts, shutdown behavior, and middleware order.
- Tier 3 collapses two independent RS256 bearer implementations into one.

**Negative / costs**

- Every service takes a go-platform upgrade. `example-resource-service` is on
  v0.3.0 while `auth-server` is on v0.10.0, so its migration also absorbs seven
  intervening releases.
- `/health` stops appearing in access logs by default. This is intended
  (Fly's health checks dominate the log volume) but is a visible change;
  `WithSkipLogPaths()` restores it.
- A shared `Stack` makes custom ordering slightly less convenient. The
  individual middleware remain exported for the rare service that needs it.
- A panicking request produces the recovery log line but no access-log line
  (`Logging` sits inside `Recovery` and is unwound by the panic). This matches
  the existing documented chain and is not changed here.

## Alternatives considered

1. **Add everything to `httputil`.** Rejected: forces `jwtutil` into a package
   whose callers mostly want response helpers, breaking the
   independently-usable-packages convention.
1b. **Leave the three existing middleware in `httputil` and have `httpmw`
   compose them.** Rejected: splits "middleware" across two packages and
   leaves `httputil` deprecation forwarders that would need `httputil` to
   import `httpmw` (a cycle, since `httpmw` uses `httputil.WriteJSON`).
2. **Adopt a third-party router/middleware set (chi, etc.).** Rejected: the
   services use the stdlib `ServeMux` with Go 1.22 method patterns by design,
   and the needed pieces (OAuth-aware auth, apperrors-shaped bodies) are
   specific to this portfolio.
3. **Leave each service as is and fix the ordering bug six times.** Rejected:
   the defect exists because ordering is per-service convention.

## Rollout

1. **Tier 1** — release go-platform (breaking: middleware moved out of
   `httputil`); migrate each service's `routes.go` to `httpmw.Stack` and `main.go` to `httpmw.NewServer(...).Run(ctx)`; replace
   per-service `Health` handlers where the Swagger annotations allow. One PR
   per service, `refactor(<service>)`. `entitlements-service` first (no
   middleware to replace), then the six services with the ordering defect.
2. **Tier 2** — release, then replace the inline secret check and limiter in
   `auth-server` and `token-introspection-service`.
3. **Tier 3** — release, then migrate `example-resource-service` and
   `jk-metering`. `RequireDPoP` moves last because only one service uses it.

## Open questions

- Should `Stack` add `otelhttp` behind an optional sub-package so the span
  formatter is shared? Deferred until Tier 1 migrations show whether the
  per-service wrapper is actually duplicated in practice.
- Fixed-window versus token-bucket for the Tier 2 limiter; the existing
  behavior is fixed-window and is the default until a service needs more.
