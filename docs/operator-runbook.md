# Operator runbook — billing + metering deployment

Concrete steps to bring the Phase B billing pipeline live: deploy
self-hosted Lago, connect Stripe, deploy the two metering services,
configure login-ui, define the first SKUs, and run an end-to-end smoke
test. Run the steps top-to-bottom on a fresh environment; resume from
the matching section when re-running parts.

> All the code is merged. This runbook is the missing operator-side work
> — accounts, secrets, deployments, and Lago configuration.

- **Design context:** [`agentic-posture.md`](agentic-posture.md#usage-accounting),
  [identity-platform-go ADR-0019](https://github.com/jedi-knights/identity-platform-go/blob/main/docs/adr/0019-usage-accounting-and-billing.md)
- **Design-time sequence:** [`billing-and-metering-setup.md`](billing-and-metering-setup.md)
- **Per-project pages:** [`jk-metering`](jk-metering.md)

## Pipeline once everything is live

```mermaid
graph LR
    User[End user] -->|sign up| LUI[login-ui]
    LUI -->|list plans| Lago[(self-hosted Lago<br/>on Fly.io)]
    LUI -->|Checkout session| Stripe[(Stripe<br/>Checkout • Tax)]
    User -->|card| Stripe
    Stripe -.webhook.-> Lago

    User -->|tool call + bearer| MCP[jk-mcp-* / auth-server]
    User -->|web app event| Ingest[jk-metering-ingest]
    MCP -->|audit| Audit[(audit_events)]
    Ingest -->|audit| Audit
    Audit --> Meter[jk-metering<br/>worker]
    Meter -->|usage events| Lago
    Lago -->|invoice| Stripe
    Stripe -.charge result.-> Lago
```

## 0. Prerequisites

- Fly.io org with billing enabled
- Stripe account (start in test mode; flip to live last)
- `flyctl` and `gh` CLIs authenticated
- `psql` for one Lago admin task that doesn't fit in `fly ssh`
- Domain on the gateway (e.g. `auth.jediknights.dev`) — referenced by
  the JWKS URL and the Stripe success/cancel URLs

Decide once, write down:

| Decision | Recommended default | Why |
|---|---|---|
| Lago region | `iad` | Matches existing identity services |
| Stripe mode | test until end-to-end smoke passes | Test invoices stay separate from real |
| Billing identity | `subject` | "User owns the cost" per ADR-0019 |
| Polling interval | 5 s | Worker default; tighten when billing latency matters |
| Currency | USD | Switch in step 5 if different |

## 1. Stripe setup

Test mode unless noted.

```bash
# Toggle test mode at https://dashboard.stripe.com (top-right)
```

1. **Enable products** in the Stripe dashboard:
   - **Stripe Checkout** (Settings → Payments → Checkout) — default config is fine
   - **Customer Portal** (Settings → Billing → Customer portal) — allow:
     card update ✅, invoice history ✅, subscription cancellation per
     business rules
   - **Stripe Tax** (Settings → Tax) — register origin address, set
     automatic tax to ON

2. **Create a restricted API key** at Developers → API keys:
   - **Permissions:** Customers (write), PaymentMethods (write),
     PaymentIntents (write), Invoices (write), Subscriptions (write),
     Webhooks (read).
   - Save the secret as `STRIPE_API_KEY_TEST` in your password manager.

3. **Configure a webhook endpoint pointing at Lago** (you'll set the
   exact URL after Lago deploys in step 2; come back to this then):
   - Endpoint URL: `https://api.billing.<your-domain>/webhooks/stripe`
   - Events: `checkout.session.completed`,
     `invoice.payment_succeeded`, `invoice.payment_failed`,
     `customer.subscription.updated`,
     `customer.subscription.deleted`
   - Save the webhook signing secret as `STRIPE_WEBHOOK_SECRET_TEST`.

## 2. Deploy self-hosted Lago to Fly.io

Lago ships three Docker images. Four Fly apps total (two Lago Rails
apps + admin UI + Postgres), plus an Upstash Redis for the Sidekiq
queue.

**Naming.** `lago-api`, `lago-worker`, and `lago-front` are all
globally-taken app names on Fly. Use the `jk-lago-*` prefix that
matches the rest of the jedi-knights suite. `lago-pg` was still free
at time of writing but is grabbed here without the prefix — rename to
`jk-lago-pg` if it clashes at deploy time.

**URLs.** These steps deploy against Fly's default `*.fly.dev`
hostnames. If you own a domain and want branded URLs, add a `CNAME`
after §2.7 and run `fly certs create` per app — then rotate the
`LAGO_API_URL` / `LAGO_FRONT_URL` / `API_URL` secrets and redeploy.
The Stripe webhook URL (§3) needs to be updated in lock-step.

### 2.1a Provision Postgres

Lago's canonical `docker-compose.yml` runs `getlago/postgres-partman`,
not vanilla Postgres — the image bundles `pg_partman` for partitioning
the `events` table. `fly postgres create` provisions vanilla Postgres
and Lago's migrations expect the partman extension, so deploy the
partman image as its own Fly app instead.

**Wrap the entrypoint.** Two Fly-side gotchas stack on this image:

1. Fly mounts volumes as `root:root 0755` on every boot, overriding
   any filesystem-level ownership you set. `postgres:alpine`'s
   `docker-entrypoint.sh` `gosu`s to the `postgres` user before
   creating `$PGDATA`, so it cannot `mkdir` a subdir of a root-owned
   mount root — the machine restart-loops on `Permission denied`.
2. `$PGDATA` cannot point at the mount root itself, because the ext4
   filesystem leaves a `lost+found` there and `initdb` refuses to
   initialize a non-empty directory.

Fix both by (a) using a subdirectory for `$PGDATA` and (b) building a
tiny wrapper image that chowns the mount before delegating to the
stock entrypoint.

```bash
mkdir -p /tmp/lago-pg-build && cd /tmp/lago-pg-build

cat > Dockerfile <<'DOCKERFILE'
FROM getlago/postgres-partman:15.0-alpine
USER root
COPY entrypoint.sh /usr/local/bin/wrapped-entrypoint.sh
RUN chmod +x /usr/local/bin/wrapped-entrypoint.sh
ENTRYPOINT ["/usr/local/bin/wrapped-entrypoint.sh"]
CMD ["postgres"]
DOCKERFILE

cat > entrypoint.sh <<'SH'
#!/bin/sh
set -e
# Fly re-chowns the mount root to root:root on every boot; fix it here
# before the stock entrypoint drops privileges to `postgres`.
chown -R postgres:postgres /data/postgres
chmod 0700 /data/postgres
exec docker-entrypoint.sh "$@"
SH

cat > fly.toml <<'TOML'
app = "lago-pg"
primary_region = "iad"

[build]
  dockerfile = "Dockerfile"

[env]
  POSTGRES_DB = "lago"
  POSTGRES_USER = "lago"
  # PGDATA must be a subdir of the mount, not the mount root itself
  # (initdb refuses non-empty dirs; the mount root has lost+found).
  PGDATA = "/data/postgres/pgdata"

[mounts]
  source = "lago_pg_data"
  destination = "/data/postgres"

[[vm]]
  cpu_kind = "shared"
  cpus = 1
  memory = "1gb"
TOML

fly apps create lago-pg --org <your-org>
fly volumes create lago_pg_data -a lago-pg --region iad --size 10 --yes

POSTGRES_PASSWORD="$(openssl rand -hex 32)"
fly secrets -a lago-pg set POSTGRES_PASSWORD="$POSTGRES_PASSWORD"

fly deploy -a lago-pg -c fly.toml --remote-only

# Construct the DATABASE_URL for downstream apps. Fly's 6PN routes any
# TCP port on <app>.internal automatically; no public listener needed.
DATABASE_URL="postgres://lago:${POSTGRES_PASSWORD}@lago-pg.internal:5432/lago"
```

Trade-off: this Postgres is single-instance with a volume. There's no
Fly-managed HA or automated backup. Schedule `pg_dump` to object
storage as a follow-up, or accept the trade for the (small) billing
volume Phase B starts at. Switching to managed Postgres later means
re-deploying with a `pg_dump | psql` migration — straightforward but
not zero-downtime.

### 2.1b Provision Redis

```bash
fly redis create --name lago-redis --region iad --org <your-org> \
  --plan "Pay-as-you-go" --disable-eviction --no-replicas
# Interactive: answer `n` to the ProdPack ($200/mo) prompt.
# This flag cannot be scripted around; the command needs a TTY.

# Note the REDIS_URL from the output — save it to your vault.
REDIS_URL="redis://default:<password>@fly-lago-redis.upstash.io:6379"
```

**Pricing watch.** Pay-as-you-go is $0.20 per 100K commands, and
Sidekiq polls Redis constantly. On any real workload this beats the
$10/mo Fixed 250MB plan within days. Migrate with `fly redis update
--plan "Fixed 250MB"` after §2.6 smoke passes.

### 2.2 Generate encryption keys

Lago needs three encryption secrets and an RSA signing key.

```bash
LAGO_RSA_PRIVATE_KEY="$(openssl genrsa 2048 | openssl base64 -A)"
ENCRYPTION_PRIMARY_KEY="$(openssl rand -hex 32)"
ENCRYPTION_DETERMINISTIC_KEY="$(openssl rand -hex 32)"
ENCRYPTION_KEY_DERIVATION_SALT="$(openssl rand -hex 32)"
SECRET_KEY_BASE="$(openssl rand -hex 64)"
```

Save all five plus `POSTGRES_PASSWORD`, `REDIS_URL`, `DATABASE_URL`,
and `STRIPE_API_KEY_TEST` to a vault — Fly's secret store is
write-only, so a lost value cannot be recovered without a key rotation
that invalidates every issued JWT.

### 2.3 Deploy jk-lago-api

```bash
fly apps create jk-lago-api --org <your-org>
fly secrets -a jk-lago-api set \
  DATABASE_URL="$DATABASE_URL" \
  REDIS_URL="$REDIS_URL" \
  LAGO_RSA_PRIVATE_KEY="$LAGO_RSA_PRIVATE_KEY" \
  ENCRYPTION_PRIMARY_KEY="$ENCRYPTION_PRIMARY_KEY" \
  ENCRYPTION_DETERMINISTIC_KEY="$ENCRYPTION_DETERMINISTIC_KEY" \
  ENCRYPTION_KEY_DERIVATION_SALT="$ENCRYPTION_KEY_DERIVATION_SALT" \
  SECRET_KEY_BASE="$SECRET_KEY_BASE" \
  LAGO_FROM_EMAIL="noreply@jk-lago-api.fly.dev" \
  LAGO_API_URL="https://jk-lago-api.fly.dev" \
  LAGO_FRONT_URL="https://jk-lago-front.fly.dev"

# Minimal fly.toml. Pin to an explicit Lago release — there is no
# floating `v1` tag on Docker Hub; bump this when upgrading.
cat > /tmp/jk-lago-api.fly.toml <<'TOML'
app = "jk-lago-api"
primary_region = "iad"

[build]
  image = "getlago/api:v1.48.1"

# RAILS_ENV=production is load-bearing: the image defaults to
# `development`, which references the dotenv-rails and annotate_rb
# gems that only exist in the development bundle. Rails will abort at
# boot without this. LAGO_DISABLE_SEGMENT keeps analytics quiet.
[env]
  RAILS_ENV = "production"
  RAILS_LOG_TO_STDOUT = "true"
  LAGO_DISABLE_SEGMENT = "true"

[http_service]
  internal_port = 3000
  force_https = true
  auto_stop_machines = false
  auto_start_machines = true
  min_machines_running = 1

  [[http_service.checks]]
    grace_period = "60s"     # Rails boot is slow; short grace churns
    interval = "15s"
    method = "GET"
    timeout = "10s"
    path = "/health"

[[vm]]
  cpu_kind = "shared"
  cpus = 1
  memory = "1gb"
TOML
fly deploy -a jk-lago-api -c /tmp/jk-lago-api.fly.toml --remote-only

# Run Lago's DB migration once. The `lago` database is created by
# postgres-partman on first boot from POSTGRES_DB; only the schema
# needs bootstrapping here.
fly ssh console -a jk-lago-api -C "bundle exec rails db:migrate"
```

### 2.4 Deploy jk-lago-worker (background jobs)

Same image and same secret set as `jk-lago-api`, with an overridden
CMD to run the worker script:

```bash
fly apps create jk-lago-worker --org <your-org>
# Re-run the same `fly secrets set` block from §2.3, targeting
# jk-lago-worker. There is no cross-app secret copy on Fly.

cat > /tmp/jk-lago-worker.fly.toml <<'TOML'
app = "jk-lago-worker"
primary_region = "iad"

[build]
  image = "getlago/api:v1.48.1"

[env]
  RAILS_ENV = "production"
  RAILS_LOG_TO_STDOUT = "true"
  LAGO_DISABLE_SEGMENT = "true"

# Overrides the image's default CMD (which starts the Rails server).
# The image has no ENTRYPOINT, so [processes] fully replaces CMD.
[processes]
  app = "./scripts/start.worker.sh"

[[vm]]
  cpu_kind = "shared"
  cpus = 1
  memory = "512mb"

[[restart]]
  policy = "always"
TOML
fly deploy -a jk-lago-worker -c /tmp/jk-lago-worker.fly.toml --remote-only
```

### 2.5 Deploy jk-lago-front (admin UI)

```bash
fly apps create jk-lago-front --org <your-org>
fly secrets -a jk-lago-front set \
  API_URL="https://jk-lago-api.fly.dev" \
  APP_ENV="production"

cat > /tmp/jk-lago-front.fly.toml <<'TOML'
app = "jk-lago-front"
primary_region = "iad"

[build]
  image = "getlago/front:v1.48.1"

[http_service]
  internal_port = 80
  force_https = true
  auto_stop_machines = false
  auto_start_machines = true
  min_machines_running = 1

  [[http_service.checks]]
    grace_period = "30s"
    interval = "15s"
    method = "GET"
    timeout = "5s"
    path = "/"

[[vm]]
  cpu_kind = "shared"
  cpus = 1
  memory = "256mb"
TOML
fly deploy -a jk-lago-front -c /tmp/jk-lago-front.fly.toml --remote-only
```

### 2.6 First-run smoke

```bash
curl -fsS https://jk-lago-api.fly.dev/health
# {"version":"v1.48.1","github_url":"...","message":"Success"}
```

Open `https://jk-lago-front.fly.dev` and sign up **immediately** — the
first account to register becomes the admin. The URL is publicly
reachable; treat that time window as sensitive.

### 2.7 Mint a Lago API key

Lago admin → Settings → API keys → Create. Save as `LAGO_API_KEY` in
the vault — every downstream service needs it.

## 3. Connect Stripe to Lago

In Lago admin:

1. **Settings → Integrations → Add Stripe.**
2. Paste `STRIPE_API_KEY_TEST` and the webhook signing secret from
   step 1.3.
3. Click Test connection.
4. Go back to the Stripe dashboard webhook from step 1.3; the URL is
   now `https://jk-lago-api.fly.dev/webhooks/stripe` (or your custom
   domain if you attached one after §2.7). Confirm the endpoint shows
   recent successful pings.

## 4. Deploy the metering services

Both binaries live in [`jedi-knights/jk-metering`](https://github.com/jedi-knights/jk-metering)
and share one Dockerfile. The worker has no public listener; the
ingest service runs HTTP behind the gateway.

### 4.1 Worker — `jk-metering`

```bash
cd /path/to/jk-metering
fly apps create jk-metering --org <your-org>
fly secrets -a jk-metering set \
  METERING_AUDIT_DSN="<identity audit_events DSN>" \
  METERING_LAGO_BASE_URL="http://jk-lago-api.internal:3000" \
  METERING_LAGO_API_KEY="$LAGO_API_KEY"
fly deploy -a jk-metering --remote-only -c fly.toml
```

The `audit_events` table is whatever Postgres the identity services
write to via `go-platform/audit/durable`. It can be the
identity-platform-go DB or its own dedicated Postgres.

### 4.2 Ingest — `jk-metering-ingest`

```bash
fly apps create jk-metering-ingest --org <your-org>
fly secrets -a jk-metering-ingest set \
  METERING_AUDIT_DSN="<identity audit_events DSN>" \
  METERING_INGEST_JWKS_URL="http://jk-auth-server.internal:8080/.well-known/jwks.json" \
  METERING_INGEST_EXPECTED_ISSUER="identity-platform"
fly deploy -a jk-metering-ingest --remote-only -c fly.ingest.toml
```

**On the JWKS URL.** Internal over Fly's 6PN (HTTP, port `8080` per
`fly.auth-server.toml`) is preferred: no public-routing dependency, no
TLS termination in the path, and the JWKS document is cached in the
ingest for an hour so latency is irrelevant. Use the public
`https://auth.jediknights.dev/.well-known/jwks.json` only if you have
a specific reason to bypass 6PN.

**On the expected issuer.** The value must match exactly what
`jk-auth-server` mints into the `iss` claim of access tokens, which
comes from `AUTH_JWT_ISSUER` on that service. The deployed setting is
the bare string `identity-platform` (see `fly.auth-server.toml`, and
where the value flows in through `container.go` →
`NewRS256TokenGenerator(keys, cfg.JWT.Issuer, cfg.JWT.Audience)`). If
the platform later switches to an OIDC-compliant URL issuer, update
both `AUTH_JWT_ISSUER` on the auth-server *and* this value in
lock-step — a mismatch fails every token verification silently at the
ingest.

Route `https://auth.jediknights.dev/metering/events` →
`jk-metering-ingest:8090` on the `jk-api-gateway`.

## 5. Wire login-ui to Lago

```bash
fly secrets -a login-ui set \
  LOGIN_UI_BILLING_LAGO_URL="http://jk-lago-api.internal:3000" \
  LOGIN_UI_BILLING_LAGO_API_KEY="$LAGO_API_KEY" \
  LOGIN_UI_BILLING_SUCCESS_URL="https://auth.jediknights.dev/billing/portal?subject={CHECKOUT_SESSION_SUBJECT}" \
  LOGIN_UI_BILLING_CANCEL_URL="https://auth.jediknights.dev/billing/plans?subject={CHECKOUT_SESSION_SUBJECT}"
fly deploy -a login-ui --remote-only
```

Replace `{CHECKOUT_SESSION_SUBJECT}` with whatever Lago substitutes for
the customer's external ID in URLs — check the Lago docs for the exact
placeholder syntax.

## 6. First SKUs in Lago

Set up enough Lago configuration to bill one full SKU shape per
category, so the smoke test exercises every audit event_type the
portfolio currently emits.

### 6.1 Billable metrics

In Lago admin → Billable metrics → Create one per row:

| Code | Aggregation | Filter |
|---|---|---|
| `mcp_tool_call` | count | `event_type = tool_invoked` |
| `nwsl_server_use` | count | `resource_parent = jk-mcp-nwsl` |
| `auth_token_issuance` | count | `resource_kind = token` AND `action = issue` |
| `webapp_feature_use` | count | `resource_kind = feature` |

Each metric reads against the single `usage` event code that
`jk-metering` emits — Lago discriminates entirely via filters.

### 6.2 Plans

In Lago admin → Plans → Create three plans:

- **Free** ($0/mo). Add every billable metric from 6.1 with a generous
  free quota and no overage.
- **Starter (PAYG)** ($0/mo). Add billable metrics with `standard`
  per-unit pricing, no minimums.
- **Pro** (e.g. $19/mo). Flat fee + generous included quotas + overage
  at Starter pricing.

## 7. End-to-end smoke

The acceptance scenario for Phase B. Run it as a new user in test mode.

```bash
# 1. Sign up via login-ui (creates a user in identity-service).
open https://auth.jediknights.dev/sign-up

# 2. Sign in (the identity-service emits user_authenticated).
# 3. Land on /billing/plans?subject=<user-id>.
# 4. Pick Starter → POST /billing/checkout → redirect to Stripe Checkout.
# 5. Pay with Stripe test card 4242 4242 4242 4242, any future date, any CVC.
# 6. Stripe webhook → Lago → subscription provisioned.

# 7. Obtain an access token (client-credentials for a test agent
#    registered in client-registry-service with actor_type=agent).
curl -sX POST https://auth.jediknights.dev/oauth/token \
  -d grant_type=client_credentials -d client_id=<id> -d client_secret=<secret>

# 8. Call an MCP tool with the bearer token.
# 9. Verify the audit_events row.
psql "$AUDIT_DSN" -c "SELECT event_id, event_type, actor_type, resource_path
                       FROM audit_events ORDER BY created_at DESC LIMIT 5"

# 10. Within METERING_METERING_POLL_INTERVAL_SECONDS (5s default), the
#     row's consumed_at goes non-null. Confirm a matching event in Lago:
open https://jk-lago-front.fly.dev  # Events → filter by subject
```

Then advance Lago's clock (admin → Subscriptions → … → Issue invoice
now in test mode) and confirm Stripe shows a charge against the test
card.

## 8. Flip to live mode

After the test-mode smoke passes:

1. Generate a **live** Stripe restricted API key + webhook signing
   secret.
2. `fly secrets -a jk-lago-api set STRIPE_API_KEY=$STRIPE_API_KEY_LIVE
   STRIPE_WEBHOOK_SECRET=$STRIPE_WEBHOOK_SECRET_LIVE` and redeploy.
3. Update the Lago Stripe integration to use the live key.
4. Stripe will require activation (business details, bank account) —
   complete that in the Stripe dashboard.

## 9. Operations

### Monitoring

- **`jk-metering` worker.** Watch the `Skipped` and `Failed` counters
  exposed on the `Stats()` snapshot; emit them as Prometheus / OTel
  metrics in the follow-up that adds OTel.
- **`audit_events` table size.** A consumed row stays in the table —
  add a periodic prune in a future iteration, or set up Postgres
  partition-by-week now.
- **Lago dashboards.** Lago admin → Analytics shows invoiced amount,
  active subscriptions, MRR.
- **Stripe radar.** Card-decline patterns surface here, not in Lago.

### Key rotation

- **Lago API key.** Mint a new key in Lago admin, set it on every
  consumer (`jk-metering`, `jk-metering-ingest` (if it ever calls
  Lago — today it does not), `login-ui`), then revoke the old one in
  Lago admin.
- **Stripe API key.** Generate the new restricted key, update on Lago
  (`fly secrets -a jk-lago-api set`), redeploy `jk-lago-api`, revoke
  the old key.
- **Stripe webhook secret.** Roll the new one into Lago via the same
  Settings → Integrations form; old webhook calls fail-fast.

### Scaling

- **`jk-metering` worker.** One process is enough through several
  hundred events/second on a `shared-cpu-1x`. When lag grows past a
  full polling interval, add machines and Postgres will round-robin
  via `SKIP LOCKED` on the partial index (`audit_events_unconsumed`).
- **`jk-metering-ingest`.** Stateless; scale on request rate. Behind
  the gateway, the rate limit is the natural ceiling.
- **Lago.** `jk-lago-api` scales horizontally; `jk-lago-worker` is
  per-process — add more instances when the Sidekiq queue grows.

## 10. Troubleshooting

| Symptom | Likely cause | First check |
|---|---|---|
| `jk-metering` logs "no billing identity" | Event missing `subject_id`, `client_id`, and `actor_id` | Inspect the row — emitter bug |
| Lago events never arrive | Worker can't reach Lago | `fly logs -a jk-metering` for `POST /api/v1/events` errors |
| Lago events arrive but stay "pending" | Lago worker is down | `fly status -a jk-lago-worker` |
| Checkout works, no invoice | Stripe webhook misconfigured | Stripe dashboard → Webhooks → recent attempts |
| Invoice raised, no charge | Stripe Tax not configured for region | Stripe Tax → Registrations |
| `consumed_at` never set after Lago push | DB connection lost mid-tick | `jk-metering` will retry; verify Lago dedupes on retry |
| `/billing/plans` shows empty | Lago has no active plans, or `LOGIN_UI_BILLING_LAGO_API_KEY` is unset | `curl -H "Authorization: Bearer $LAGO_API_KEY" .../api/v1/plans` |
| `/billing/checkout` returns 500 | Lago can't reach Stripe | `fly logs -a jk-lago-api` |

## Done state

The runbook is complete when:

- [ ] A new user can sign up via `login-ui`.
- [ ] They pick a plan and complete Stripe Checkout (test card).
- [ ] An agent / web app emits an event that lands in `audit_events`.
- [ ] `jk-metering` picks it up; a `usage` event appears in Lago
      within `METERING_METERING_POLL_INTERVAL_SECONDS`.
- [ ] At the end of a billing cycle Lago issues an invoice; Stripe
      charges the card; Lago marks the invoice paid.

At that point the gap matrix in [`agentic-posture.md`](agentic-posture.md#gap-matrix)
can flip the **Usage accounting / metering / billing** row from
*Phase B (prerequisite)* to ✅.
