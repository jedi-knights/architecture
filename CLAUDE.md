# architecture — Claude Context

This repo is docs-only. It holds the operator runbook, ADR indexes,
and per-project trajectory pages. No shippable code; all changes are
Markdown.

## Where things live

- `docs/operator-runbook.md` — the authoritative Phase B billing +
  metering deploy playbook. Update in lockstep with any deploy-time
  discovery. If the runbook and the live infra disagree, the runbook
  is wrong.
- `docs/agentic-posture.md` — usage accounting / MCP tool authz.
- `docs/billing-and-metering-setup.md` — design-time sequence diagram.
- `docs/jk-metering.md`, `docs/nvim-tool-portfolio.md`, and other
  per-project trajectories.
- `TODO.md` — gitignored active workstream tracker; always read on
  session start.

## Live infrastructure (as of 2026-08-30)

Fly.io, `personal` org.

| App | Purpose | URL |
|---|---|---|
| `jk-api-gateway` | Public ingress | `https://jk-api-gateway.fly.dev` |
| `jk-auth-server` | OAuth server (internal) | `http://jk-auth-server.internal:8080` |
| `jk-identity-service` | Identity backend (internal) | `http://jk-identity-service.internal:8081` |
| `jk-authorization-policy-service` | Policy decisions (internal) | `.internal` |
| `jk-client-registry-service` | OAuth client registry (internal) | `.internal` |
| `jk-token-introspection-service` | RFC 7662 introspection (internal) | `.internal` |
| `jk-metering` | Worker: drains `audit_events` → Lago | (no HTTP) |
| `jk-metering-ingest` | HTTP event ingest → `audit_events` | `https://jk-metering-ingest.fly.dev` |
| `lago-pg` | Postgres for Lago | `.internal:5432` |
| `lago-redis` | Sidekiq queue | Upstash Pay-as-you-go |
| `jk-lago-api` | Lago Rails API | `https://jk-lago-api.fly.dev` |
| `jk-lago-worker` | Lago Sidekiq worker | (no HTTP) |
| `jk-lago-front` | Lago admin UI | `https://jk-lago-front.fly.dev` |

**Fly Managed Postgres clusters** (do *not* appear in `fly apps list` —
query via `fly mpg list -o personal`):

| Cluster | ID | Purpose | Attached apps |
|---|---|---|---|
| `jk-identity-pg` | `vmkq60998kp035ln` | `audit_events` + all identity-service state (region `iad`, plan `basic`) | `jk-identity-service`, `jk-client-registry-service`, `jk-authorization-policy-service`, `jk-example-resource-service` |

`jk-metering` and `jk-metering-ingest` also read/write `audit_events`
on `jk-identity-pg`, but they got the DSN copy-pasted rather than
attached via `fly mpg attach` (secrets digest confirms same DSN as
`jk-identity-service`'s `IDENTITY_DATABASE_URL`). Not visible in the
MPG cluster's attached-apps list — but they hit the same database.

To tunnel to `jk-identity-pg` from a laptop or CI runner: `fly mpg
proxy vmkq60998kp035ln -p 5432` (default port is `16380`; override to
`5432` if downstream code assumes the standard PG port). Plain
`flyctl proxy -a` does **not** work against MPG.

**Not yet deployed:** Stripe integration (§3; blocked on Stripe
sandbox `whsec_...` secret).

## Lago Phase B signup gotcha (2026-08-02)

Discovered while working through operator-runbook §2 for the first
real deploy. Save future sessions the loop.

**Symptom:** After `jk-lago-api` deploys cleanly and
`rails db:migrate` succeeds, opening `https://jk-lago-front.fly.dev/sign-up`
and submitting the form silently does nothing. The API logs show the
GraphQL `registerUser` mutation returning `status: 404`, and the DB
still has 0 users, 0 organizations after every attempt.

**Root cause:** Lago's `UsersService#register_from_email` (in
`app/services/users_service.rb`) calls `Role.admins.first!` inside the
signup transaction. On a freshly-migrated DB the `roles` table is
empty, `first!` raises `ActiveRecord::RecordNotFound` (surfaced as
404), and the whole transaction rolls back — user, organization,
membership all reverted. Segment tracking jobs get enqueued before the
raise, which is why the API logs show `billing_entity_created` events
that don't match any real DB row.

**Fix:** `fly ssh console -a jk-lago-api -C "bundle exec rails db:seed"`
after the migration. Seeds four roles (`Admin`, `Finance`, `Manager`,
`Accountant`). Signup then works normally and redirects to the admin
dashboard.

**Side effect of the fix:** Lago's `db/seeds.rb` unconditionally seeds
demo data — a placeholder org named "Hooli"
(`11111111-2222-3333-4444-555555555555`) with sample billable metrics,
plans, add-ons, an invoice, and a credit note.

**Hooli is permanent in v1.48 — do not try to purge it.** Verified
2026-08-02 across every layer:

- No `DELETE` route on `/api/v1/organizations` or `/admin/organizations`
  (`rails routes | grep organization`).
- No `Organizations::DestroyService` — the services directory ships
  `create_service.rb` and `update_service.rb` only.
- No admin UI path (Settings → Organization has no delete button).
- Direct SQL cascade is blocked by `entitlement_entitlements` →
  `entitlement_entitlement_values` and other FK relationships that
  lack `dependent: :destroy` on the Rails side and lack `ON DELETE
  CASCADE` on the DB side.
- `SET session_replication_role = 'replica'` (to bypass FK
  enforcement) needs `SUPERUSER`, which Fly's managed Postgres does
  not grant on user apps.

Hooli is harmless — real signups get a different UUID and Hooli never
appears in any user's org switcher unless a `Membership` is explicitly
created for them. Wait for a future Lago version to add org destroy;
until then, ignore.

**Runbook status:** the `db:seed` step + the corrected Hooli caveat
are inline in `docs/operator-runbook.md` §2.3.

## Stripe webhook URL (2026-08-02)

Lago's Stripe webhook route is **org-scoped**:
`POST /webhooks/stripe/:organization_id`. A bare `/webhooks/stripe`
returns 404 (verified via `rails routes | grep stripe`). Both §1.3
and §3 of `docs/operator-runbook.md` are now updated to include the
`<lago-org-id>` placeholder and how to fetch it. Do not paste a
`/webhooks/stripe` URL into the Stripe dashboard — it fails silently
until the first delivery attempt.

**Live Lago org IDs on `jk-lago-api` (verified 2026-08-29):**

| Name | ID | Use? |
|---|---|---|
| Hooli | `11111111-2222-3333-4444-555555555555` | ❌ seed demo org — ignore |
| Jedi Knights | `605bdae2-b087-4a32-b797-ca18c7be4aa3` | ✅ real org — wire Stripe against this |

Stripe webhook URL for the Jedi Knights org:
`https://jk-lago-api.fly.dev/webhooks/stripe/605bdae2-b087-4a32-b797-ca18c7be4aa3`.
Refetch after any org create/delete via
`fly ssh console -a jk-lago-api -C "bundle exec rails runner 'puts Organization.pluck(:name, :id)'"`.

## Conventions

- Every runbook fix that came from live discovery is a **Should Fix**
  or higher at review time — the runbook's job is to work end-to-end
  on a fresh environment, so drift from live is a defect, not a
  polish opportunity.
- Follow the Angular Conventional Commits rule from the global
  workflow. Docs changes use `docs(<scope>)`.
- One `type(scope)` per PR; runbook edits stay separate from ADR
  edits and from per-project trajectory doc edits.
