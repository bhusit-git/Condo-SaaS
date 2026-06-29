# Deployment Flow

This document defines the target v1.0 Core Backoffice deployment flow for the
Condo SaaS system, plus the v1.1 additions needed for LINE/LIFF.
It is a deployment contract, not an implemented CI workflow. GitHub Actions YAML,
Cloudflare project settings, and Supabase project setup should be created later
to match this document. LINE channel setup is required only when v1.1 LINE/LIFF
ships.

## Deployment Shape

v1.0 uses one Cloudflare Pages project for the two React Vite app entrypoints:

- `/admin`
- `/staff`

The Super Admin control plane lives at `/admin/platform` inside the admin app for
v1.0. It does not create a third frontend entrypoint or a separate Pages project
until platform release ownership, isolation, or traffic requires that split.

The apps remain separate Vite frontends inside the workspace, but they are built
and published under one Pages site for v1.0. This keeps the public URL, Supabase
redirect configuration, and release version aligned during the pilot.

v1.1 adds `/liff` under the same Pages site unless resident traffic, LINE setup,
or release ownership requires a separate deployment.

Stable environment URLs use placeholders until real domains are selected:

- Staging: `https://staging.<domain>`
- Production: `https://app.<domain>`

Cloudflare preview URLs are allowed for pull request review only. They must not
be treated as production callback URLs. They must not be treated as stable LIFF
endpoints after v1.1 ships.

## Environments

v1.0 has three environments:

- Local: developer machines, local Vite, and local Supabase where practical.
- Staging: online release rehearsal, QA, and test data.
- Production: real condo, staff, resident records, and backoffice traffic.

Staging and production must use separate Supabase projects. Each project owns its
own database, Auth users, Storage buckets, Edge Functions, secrets, schedules,
and API keys. Do not share one Supabase project with schema or tenant flags for
staging and production.

Starting in v1.1, staging and production must also use separate LINE channels and LIFF endpoints.
Staging notifications and webhooks must never share the production LINE channel
used by real residents.

## Source Of Truth

GitHub Actions is the source of truth for deployment orchestration.

- GitHub Environments store deployment secrets separately for `staging` and
  `production`.
- Production deployment requires GitHub Environment approval.
- Supabase CLI is used from GitHub Actions for migrations and Edge Function
  deploys.
- Cloudflare Pages hosts the frontend. The CI workflow may deploy through
  Cloudflare's Git integration, API, or deploy hooks, but the release policy must
  remain the same.

Secrets and environment variables should not be configured only by memory or
one-off local commands. Dashboard changes are acceptable for emergency operations,
but the intended steady-state flow is Actions-driven and environment-scoped.

## Branch And Release Flow

Pull requests:

- Build the frontend and publish a Cloudflare Pages preview.
- Use staging backend configuration only for safe review.
- Do not run staging or production migrations from pull request previews.
- Do not use preview URLs as stable LINE LIFF endpoints once v1.1 ships.

`main` branch:

- Merge to `main` deploys staging automatically.
- Staging deployment applies migrations, runs bootstrap, deploys Edge Functions,
  deploys frontend, and runs smoke checks.

Version tags:

- Production deploys only from version tags such as `v1.0.0`.
- Production deploy requires GitHub Environment approval before secrets are
  available and before migrations run.
- The deployed production version should be traceable to the tag.

## Deployment Order

For full-stack releases, deploy in this order:

1. Apply backward-compatible database migrations.
2. Run idempotent post-migration bootstrap.
3. Deploy Supabase Edge Functions.
4. Deploy Cloudflare Pages frontend.
5. Run post-deploy smoke checks.

Database migrations must follow an expand/contract style. A release may add
tables, columns, indexes, policies, RPCs, or compatible behavior before new code
uses them. Breaking renames, removals, or policy changes should be staged across
multiple releases so the previous frontend and Edge Functions can survive during
rollout and rollback.

Frontend-only releases may deploy the frontend without running migrations or
Edge Function deployment, as long as the frontend does not depend on new backend
contract changes.

## Supabase Deployment

Migrations:

- Staging migrations run automatically after merge to `main`.
- Production migrations run only during approved tag deployments.
- Production down migrations are not part of the normal rollback flow.

Edge Functions:

- Deploy with Supabase CLI from GitHub Actions.
- Keep function deployment after database migrations so functions can rely on the
  expanded schema.
- Required functions include the v1.0 contract functions such as parcel workflows,
  import workflows, announcement workflows, and platform owner workflows.
  v1.1 adds `line_webhook_ingest`, `notification_worker_tick`, and LIFF workflows.

Bootstrap:

- Run as an idempotent post-migration step.
- Seed or verify protected preset permission defaults, package defaults, required
  Storage buckets, and scheduled worker configuration. v1.1 additionally seeds
  or verifies Shared Platform LINE channel references and quota defaults.
- Seed or verify platform owner grant/revocation metadata needed to reject stale
  Super Admin tokens after access is removed.
- It must be safe to run bootstrap repeatedly in staging and production.

Scheduled workers:

- `notification_worker_tick` scheduling is managed by CI/bootstrap once v1.1
  LINE/LIFF ships.
- `platform_aggregate_usage_tick` scheduling is managed by CI/bootstrap and
  writes delayed usage snapshots. It must not be replaced by per-row database
  triggers on high-volume operational tables.
- Do not rely on manual dashboard setup as the only record of worker scheduling.

## Frontend Deployment

Cloudflare Pages hosts one site with route entrypoints for `/admin` and `/staff`.
v1.1 adds `/liff`.

`/admin/platform` is served by the `/admin` app and protected by platform
authorization checks. Route visibility is not a security boundary.

The deployment must support SPA fallback per app route, for example:

- `/admin/*` falls back to `/admin/index.html`
- `/staff/*` falls back to `/staff/index.html`
- `/liff/*` falls back to `/liff/index.html` starting in v1.1

Environment variables must be environment-specific and must point to the matching
Supabase project, Edge Function base URL, and public app URL. v1.1 also requires
LINE LIFF configuration for that environment.

## LINE And LIFF Configuration (v1.1)

Staging:

- Uses the staging LINE channel.
- Uses the staging LIFF endpoint under `https://staging.<domain>/liff`.
- Uses staging Supabase Edge Function webhook URLs.
- May use test condos, staff, residents, and LINE accounts only.

Production:

- Uses the production LINE channel.
- Uses the production LIFF endpoint under `https://app.<domain>/liff`.
- Uses production Supabase Edge Function webhook URLs.
- Handles real resident notification and binding traffic.

When real domains are selected, update Cloudflare DNS and Pages custom domains,
Supabase Auth redirect allowlists, public app URL environment variables, and any
generated links that embed the old URL. Starting in v1.1, also update LINE LIFF
endpoint settings, webhook settings, and generated QR/deep links.

## Verification

Every staging and production deployment should run automated smoke checks after
deployment.

Minimum checks:

- Cloudflare Pages serves `/admin` and `/staff`. v1.1 smoke checks also verify
  `/liff`.
- Supabase migration state matches the release.
- Required Edge Functions respond to safe health or no-op checks.
- v1.1 LINE webhook endpoint is reachable for the active environment.
- v1.1 `notification_worker_tick` schedule exists.
- `platform_aggregate_usage_tick` schedule exists.
- Storage buckets required by the v1.0 contract exist.
- v1.1 staging can run a safe notification queue or worker smoke check without
  sending to production residents.
- Staging can verify that a non-platform user cannot access `/admin/platform` or
  platform owner functions.

Manual QA may still happen after automated checks, but command success alone is
not enough to declare the deployment healthy.

## Rollback Policy

Rollback favors application rollback and database forward-fix.

- Roll back Cloudflare Pages frontend to a previous compatible deployment when
  UI behavior is broken.
- Roll back Supabase Edge Functions to a previous compatible version when server
  behavior is broken.
- Do not automatically run production down migrations.
- Fix database issues with forward migrations or targeted hotfixes.

This policy depends on backward-compatible migrations. If a release needs a
destructive schema change, split it into multiple releases with an explicit
expand, migrate, and contract sequence.
