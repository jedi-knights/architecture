# Stripe — Phase B billing setup

Stripe-specific setup for the Phase B billing pipeline: CLI install and
authentication, test-mode vs. live-mode discipline, dashboard settings
that Lago's connector depends on, restricted API keys, webhooks, and the
CLI operations Phase B needs day-to-day.

- **Phase gate context:** [`agentic-posture.md`](agentic-posture.md#usage-accounting)
- **Design-time sequence:** [`billing-and-metering-setup.md`](billing-and-metering-setup.md)
- **Execution runbook (Stripe is step 1):** [`operator-runbook.md`](operator-runbook.md#1-stripe-setup)

The operator runbook covers Stripe in the narrow form that the deploy
sequence needs. This page is the full Stripe setup — install, project
model, test/live boundary, and the operations you'll reach for once the
pipeline is live.

## Prerequisites

- Stripe account (`https://dashboard.stripe.com/register`) — no card
  required to work in test mode
- Homebrew on macOS, or [Stripe CLI install instructions](https://docs.stripe.com/stripe-cli)
  for other platforms

## Installing the CLI

```bash
brew install stripe/stripe-cli/stripe
stripe --version   # stripe version 1.43.6 or later
```

The CLI ships shell completions on install; restart the shell (or
`source` its rc file) to activate them.

## Authenticating

`stripe login` starts a browser-based OAuth flow:

```bash
stripe login
```

The CLI prints a verification code, opens the Stripe dashboard, and
asks you to confirm that the code matches. On approval it stores an
API session at `~/.config/stripe/config.toml` (per-user, per-project)
and pins the session to **test mode by default**. Every subsequent
`stripe` command implicitly hits the test API unless you pass `--live`.

Verify the active session:

```bash
stripe config --list
```

### Named projects

Use `--project-name` to keep multiple Stripe accounts (or the same
account under different roles) separate:

```bash
stripe login --project-name jk-test    # named test-mode session
stripe login --project-name jk-live    # separate session, still test until --live
```

Every CLI command then takes `--project-name <name>`; commands without
the flag use the `default` project (the unnamed session created by a
plain `stripe login`). If you only log in under named projects, every
command must carry `--project-name` or the CLI will error with "no
project configured." This is the safest way to keep a live-mode
session on the machine without risking a stray command against live
data.

## Test mode vs. live mode

Stripe enforces the boundary at the API-key level: test-mode keys start
with `sk_test_` and cannot touch live data, and vice versa. The CLI
adds a second boundary via named projects and the `--live` flag.

Rules for this portfolio:

- **Default is test mode.** Every command in the runbook runs against
  test until the Phase B end-to-end smoke passes (see
  [`operator-runbook.md`](operator-runbook.md#7-end-to-end-smoke)).
- **Live mode is opt-in per command.** Never `stripe login` a live
  session as the default project. Use `--project-name jk-live` and
  `--live` together, explicitly, when the moment comes.
- **Restricted keys, not the account-wide secret.** The runbook's
  Stripe key (see [operator-runbook.md § 1. Stripe
  setup](operator-runbook.md#1-stripe-setup)) is a restricted key
  narrowed to the permissions Lago's connector needs. Never share the
  account-wide `sk_live_...` with any service.

## Dashboard settings (no CLI equivalent)

Three settings live only in the dashboard and must be enabled before
Lago can drive checkout, portal, or tax through the connector:

| Setting | Path | Notes |
|---|---|---|
| Stripe Checkout | Settings → Payments → Checkout | Default configuration is fine; the connector uses hosted Checkout Sessions |
| Customer Portal | Settings → Billing → Customer portal | Enable card update, invoice history, subscription cancellation per business rules |
| Stripe Tax | Settings → Tax | Register origin address, set automatic tax to ON |

See [`operator-runbook.md`](operator-runbook.md#1-stripe-setup) for the
minimum-viable configuration.

## Restricted API keys

Lago authenticates to Stripe with a **restricted key** — a scoped
alternative to the account-wide secret key.

Create at Developers → API keys → Create restricted key. Grant only:

| Resource | Permission |
|---|---|
| Customers | Write |
| PaymentMethods | Write |
| PaymentIntents | Write |
| Invoices | Write |
| Subscriptions | Write |
| Webhooks | Read |

Save the resulting key as:

- `STRIPE_API_KEY_TEST` — test-mode key; used in every non-production
  environment
- `STRIPE_API_KEY_LIVE` — live-mode key; used only after the
  end-to-end smoke passes

Both go into your password manager, not into any repo or `.env` that
lands on disk beyond the specific service that needs them.

## Webhooks

Stripe posts asynchronous events to Lago via webhook. The endpoint
lives on `lago-api`:

```
https://api.billing.<your-domain>/webhooks/stripe
```

Subscribe to the exact event set Lago's connector consumes:

- `checkout.session.completed`
- `invoice.payment_succeeded`
- `invoice.payment_failed`
- `customer.subscription.updated`
- `customer.subscription.deleted`

The dashboard mints a **signing secret** (`whsec_...`) when you save
the endpoint. Store as:

- `STRIPE_WEBHOOK_SECRET_TEST` — paired with the test key
- `STRIPE_WEBHOOK_SECRET_LIVE` — paired with the live key

Lago verifies every webhook signature; a mismatched or missing secret
manifests as `checkout.session.completed` events arriving in Stripe
but never provisioning a subscription in Lago.

## Local webhook forwarding

While developing changes to the billing surface (login-ui plan
picker, metering ingest endpoint), point Stripe webhooks at your
laptop instead of Lago:

```bash
stripe listen --forward-to http://localhost:3000/webhooks/stripe
```

The CLI prints a **temporary signing secret** — export it as
`STRIPE_WEBHOOK_SECRET` for the local process. This secret only
authenticates the forwarded stream; production webhooks continue
using the dashboard-configured secret.

Trigger synthetic events without a real payment:

```bash
stripe trigger checkout.session.completed
stripe trigger invoice.payment_succeeded
stripe trigger invoice.payment_failed
```

Useful during the Phase B smoke test when the goal is to exercise the
Lago → subscription-provisioning path without walking through Checkout
by hand each time.

## Test cards

The Phase B smoke uses one of:

| Card number | Behavior |
|---|---|
| `4242 4242 4242 4242` | Approved |
| `4000 0000 0000 9995` | Declined (insufficient funds) |
| `4000 0025 0000 3155` | Requires 3D Secure authentication |

Any future expiration date and any three-digit CVC work.

For the full catalogue (authentication challenges, disputes, refunds,
by-region variants), see [Stripe's testing
documentation](https://docs.stripe.com/testing).

## Common CLI operations

Verifying the pipeline is live:

```bash
# List Lago-managed customers on Stripe (paginate with --limit)
stripe customers list --limit 5

# Confirm a specific PaymentIntent by ID
stripe payment_intents retrieve pi_...

# Replay a webhook event Stripe already tried to deliver
stripe events resend evt_...

# Tail webhook attempts in real time
stripe listen --print-json
```

Diagnosing a failed Checkout:

```bash
# Recent Checkout sessions
stripe checkout sessions list --limit 5

# The full session — includes success/cancel URLs, subscription ID, tax lines
stripe checkout sessions retrieve cs_...
```

Diagnosing a stuck subscription:

```bash
stripe subscriptions list --status all --limit 10
stripe subscriptions retrieve sub_...
```

## Key rotation

Restricted keys and webhook secrets rotate independently of the
account-wide credentials:

1. Mint the new restricted key or webhook secret in the dashboard.
2. Update it in Lago admin (**Settings → Integrations → Stripe**) —
   Lago tests the new credentials before persisting.
3. Redeploy `lago-api` if the credentials live in Fly secrets rather
   than Lago's encrypted store.
4. Revoke the previous key in the dashboard.

A blown rotation manifests as `401 Unauthorized` in `fly logs -a
lago-api` on the next connector call; the fix is to re-paste the
current secret in Lago admin.

## Flipping to live mode

Do this only after the test-mode end-to-end smoke in
[`operator-runbook.md`](operator-runbook.md#7-end-to-end-smoke)
passes cleanly at least once.

1. Complete Stripe **activation** (business details, bank account, tax
   registrations) at Settings → Business.
2. Mint a **live** restricted API key with the same permissions as
   the test key.
3. Create a **live** webhook endpoint pointing at the same Lago URL;
   save the live signing secret.
4. Update Lago's Stripe integration to the live key + webhook secret.
5. Run a real card through Checkout end-to-end, then refund it
   immediately from the Stripe dashboard. Confirm both the charge
   and refund reconcile in Lago before opening general availability.

## Reference

- [Stripe CLI documentation](https://docs.stripe.com/stripe-cli)
- [Stripe restricted API keys](https://docs.stripe.com/keys#limit-access)
- [Lago Stripe connector](https://docs.getlago.com/integrations/payment-providers/stripe)
- [Stripe Checkout](https://stripe.com/docs/payments/checkout) and
  [Stripe Customer Portal](https://stripe.com/docs/customer-management)
- `stripe:stripe-best-practices` skill for integration decisions and
  security guidance
- `stripe:stripe-directory` skill for discovering additional
  Stripe-scoped skills and tools
