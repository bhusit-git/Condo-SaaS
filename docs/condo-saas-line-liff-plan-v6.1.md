# Condo SaaS Phase Roadmap

## Summary

สร้าง Condo Management SaaS แบบ multi-tenant ด้วย `React Vite + Supabase +
Edge Functions` โดยเริ่มจาก v1.0 Core Backoffice ให้ระบบหลังบ้านใช้งานได้จริง
ก่อน แล้วจึงเพิ่ม LINE/LIFF เป็น v1.1 สำหรับแพ็กเกจ Business/Enterprise.

Implementation details that must not be re-decided during build live in
[v1 Implementation Contract](v1-implementation-contract.md). This plan remains
the product and architecture overview; the contract is the source of truth for
schema, RLS, Edge Function contracts, auth, queues, storage, import, audit, and
acceptance tests.

## Phase Model

- v1.0 Core Backoffice: `/admin`, `/staff`, onboarding, room/resident data,
  staff auth, preset permissions, CSV import, parcels, announcements,
  platform-owner controls, commercial package limits, billing settings, and audit.
- v1.1 LINE + LIFF: `/liff`, Shared Platform LINE OA, LINE Binding,
  LINE webhook, parcel/announcement/critical LINE notifications, quota, delivery
  tracking, and LIFF resident views.
- v1.2 Maintenance / Cleaning: Technician and Housekeeping Staff roles,
  assigned work lifecycle, evidence policy, and resident-visible request history.
- v1.3 Resident Billing: billing cycles, meter readings, invoices, invoice line
  items, manual payments, billing LINE jobs, and LIFF invoice views.
- Later phases: Custom LINE OA admin/migration UI, facility booking, visitor QR,
  documents, full custom role builder, payment gateway, PDF/tax invoice,
  accounting export, bank reconciliation, and post-move-out resident portal.

## Key Decisions

- Tenant model: `Organization` = customer/account or owner/HQ, `Condo` =
  โครงการ/ที่พัก/สาขา, and `Building -> Floor -> Unit` is the room layout inside
  that Condo.
- Resident model: `Resident` is organization-scoped. `Unit Resident` is the
  relationship to a Unit, such as owner, tenant, or family. Tenant lease history
  belongs to `lease_agreements`.
- Staff model: v1.0 preset roles are `Condo Admin`, `Condo Manager`,
  `Juristic Staff`, and `Security Staff`. v1.2 adds `Technician` and
  `Housekeeping Staff`.
- Authorization: routes are UX conventions only. Supabase RLS, Edge
  Functions/RPC, and server-side permission checks are the security boundary.
- LINE strategy: v1.0 does not require LINE. v1.1 uses Shared Platform LINE OA
  for Business/Enterprise. Custom LINE OA remains an Enterprise/premium future
  capability.
- Platform control plane: v1.0 includes `/admin/platform` inside the admin app
  for tenant subscription state, suspend/reactivate, delayed usage metrics, and
  platform audit. It is not resident billing.
- Commercial packaging: package, quota, add-on, and support entitlements are part
  of the product contract from v1.0. Entitlement to a future module means the
  customer receives access when that module ships; it does not move that module
  into v1.0 runtime.
- Frontend architecture: v1.0 ships `/admin` and `/staff`. v1.1 adds `/liff`.
  `/admin/platform` stays inside `/admin` rather than becoming a fourth app.

## v1.0 Core Data And Flows

- The v1.0 validation slice is: create condo -> import unit/resident -> configure
  staff/permissions/package state -> receive/pick up parcel -> publish
  announcement with recipient snapshot -> verify audit/history and billing
  settings.
- Bootstrap/onboarding is platform-controlled in v1.0: create Organization and
  Condo, create first Condo Admin, seed protected presets, apply package defaults,
  create required settings, and activate only after required checks pass.
- Platform access status is a runtime flag on Organization/Condo and is separate
  from subscription history. Operational writes check this flag through shared
  guards and RLS helpers.
- Canonical v1.0 permissions:
  - Residents: `residents:read`, `residents:write`, `residents:deactivate`
  - Leases/move-out: `leases:read`, `leases:write`, `leases:end`,
    `move_out_checklists:review`, `move_out_checklists:confirm`,
    `tenant_access:revoke`
  - Announcements: `announcements:draft`, `announcements:publish`
  - Parcels: `parcels:receive`, `parcels:pickup`, `parcels:read`
  - Imports/settings/staff: `imports:run`, `imports:apply_deactivation`,
    `staff:manage`, `condo_settings:manage`
- v1.1 adds LINE permissions such as `line_bindings:*`,
  `announcements:notify_line`, `critical_notifications:send`, and
  `line_settings:manage`.

## Product Modules

- Onboarding: create organization/condo, capture required Condo profile fields,
  create buildings/floors/units, import CSV, review diff, seed presets, apply
  package state, and activate after v1.0 checks pass.
- Admin operations: unit/resident management, lease/move-out records, preset role
  assignment, permission toggles, announcements, parcels, imports, settings, and
  audit.
- Staff mobile web: optimized `/staff` workflows for parcel receive/pickup and
  operational staff actions.
- Platform owner operations: view tenants, suspend/reactivate access, inspect
  subscription state, review delayed usage metrics, and audit platform actions
  from `/admin/platform`.
- Billing settings: v1.0 includes configuration-only rent, utility, and late-fee
  settings. It does not create invoices, meter readings, persisted charges,
  payment records, accounting entries, or billing LINE jobs.
- Tenant & Move-In / Move-Out: v1.0 supports one active tenant lease per Unit,
  move-out checklist evidence, explicit staff close, and transactional lease/unit
  resident close. `ends_at` never revokes access by itself.
- Parcels: v1.0 stores unit-level parcel records, optional photos, staff pickup
  confirmation, pickup name/note/photo, history, and audit. v1.1 adds resident
  LIFF parcel views and LINE notifications.
- Announcements: v1.0 publishes announcements inside the system and snapshots
  recipients. v1.1 adds LINE notify and critical LINE notification paths.
- LINE + LIFF (v1.1): condo-specific LIFF links/QR, context switching,
  LINE Binding, webhook ingest, parcel/announcement notification, critical
  notification, delivery tracking, and quota enforcement.
- Maintenance (v1.2): assigned request lifecycle for maintenance/cleaning, worker
  evidence, resident-visible history, and optional LINE notification for important
  updates after v1.1 exists.
- Resident Billing (v1.3): lease-backed invoice generation, invoice numbers,
  meter readings, manual payments, billing notifications, and LIFF invoice views.

## Commercial Packaging

Pricing and package entitlement must be designed before implementation because
they affect data model fields, feature flags, quotas, support level, upgrade
paths, and platform-owner controls.

Package rules:

- `Basic` is an operations-first package for small sites. It does not include
  LINE/LIFF and LINE notification entitlement by default.
- `Business` is the recommended package for LINE-enabled sites once v1.1 ships.
- `Enterprise` is custom-priced for organizations that need unlimited or
  contract-specific limits, Custom LINE OA entitlement, Organization HQ
  dashboard, SLA, and advanced admin controls.
- Resident Billing in package copy means entitlement to the billing module once
  v1.3 ships. v1.0 still includes only billing settings and preview math.

| Feature / Limit | Basic | Business | Enterprise |
| --- | --- | --- | --- |
| Base price | THB 199 / month / site | THB 599 / month / site | custom |
| Sites / Condos | 1 | 1 | unlimited / contract-specific |
| Units per site | 30 | 100 | unlimited / contract-specific |
| Staff accounts | 5 | 20 | unlimited / contract-specific |
| Building / floor / unit management | included | included | included |
| Resident management | included | included | included |
| CSV import | included | included | included |
| Parcel receive / pickup | included | included | included |
| Announcements | included | included | included |
| Preset roles and permissions | included | included | included |
| Audit log | included | included | included |
| Billing settings (v1.0) | included | included | included |
| Resident Billing runtime entitlement (v1.3) | included when shipped | included when shipped | included when shipped |
| LINE Binding + LIFF app (v1.1) | not included | included when shipped | included when shipped |
| LINE parcel + announcement notifications (v1.1) | not included | included when shipped | included when shipped |
| Critical LINE notifications (v1.1) | not included | included when shipped | included when shipped |
| Monthly LINE quota | not included | 1,000 messages | unlimited / contract-specific |
| Custom LINE OA | not included | not included | included when shipped |
| Income / expense tracking | not included | included when shipped | included when shipped |
| Inventory | not included | included when shipped | included when shipped |
| Technician stock issue / Work Order | not included | included when shipped | included when shipped |
| Maintenance Request | not included | included from v1.2 | included from v1.2 |
| Facility Booking | not included | included when shipped | included when shipped |
| Visitor QR | not included | included when shipped | included when shipped |
| Custom Role Builder | not included | not included | included when shipped |
| Documents module | not included | not included | included when shipped |
| Organization HQ dashboard | not included | not included | included when shipped |
| Email support | included | included | included |
| Priority support | not included | included | included |
| SLA + onboarding support | not included | not included | included |

Add-ons:

- Basic extra site: THB 199 / month / site, each extra site receives Basic
  entitlement with 30 included units.
- Basic extra units: THB 6 / unit / month for units above 30 per site.
- Business extra site: THB 500 / month / site, each extra site receives Business
  entitlement with 100 included units and LINE entitlement when v1.1 ships.
- Business extra units: THB 6 / unit / month for units above 100 per site.
- Enterprise has no standard add-on schedule; site, unit, LINE, SLA, support, and
  advanced feature limits are contract-specific.

## Imports

- v1.0 supports CSV UTF-8 only.
- Split templates: `unit_layout.csv` for building/floor/unit and `residents.csv`
  for resident identity and unit relationships.
- Resident import cannot apply rows referencing unknown building/floor/unit.
- Missing rows are warnings/proposed changes, never automatic deactivation.
- Deactivation is soft delete with explicit confirmation, audit, and access
  impact summary. v1.1 adds LINE Binding impact summary when LINE is enabled.
- Stale CSV conflicts are hard-blocked by default and require explicit audited
  override.

## Notification Delivery

- v1.0 has backoffice announcement publishing, recipient snapshots, and audit.
- v1.1 adds LINE delivery through reply, individual push, and multicast.
- v1.1 persists notification jobs and deliveries in Postgres queue tables and
  processes them with a scheduled Edge Function worker.
- v1.1 delivery statuses distinguish delivery certainty, including
  `sent_confirmed`, `sent_assumed`, `failed_individual`, `outcome_unknown`,
  `dead_letter`, `skipped_no_line_binding`, and `skipped_blocked_or_unknown`.
- No LINE broadcast in SaaS v1.1; recipients come from platform snapshots.
- Platform usage metrics are delayed hourly/daily aggregates from source tables,
  not per-row realtime trigger counters.

## Test Plan

- v1.0: authorization/RLS, staff/admin auth, platform owner auth, tenant
  suspension, bootstrap, permission matrix, import, parcels, announcements,
  lease/move-out, billing settings, audit, and frontend boundaries for `/admin`
  and `/staff`.
- v1.1: LIFF auth boundary, LINE Binding, LINE webhook, LINE notification
  delivery, quota, unreachable-state handling, and `/liff` frontend boundaries.
- v1.2: Maintenance/Cleaning roles, assignment lifecycle, evidence policy,
  resident-visible status history, and audit.
- v1.3: Resident billing generation, invoice numbers, mid-cycle tenant changes,
  LIFF isolation, manual payment balance updates, voiding, overdue transitions,
  and billing notification targeting.

## Assumptions

- v1.0 uses Supabase Auth/session for staff/admin with custom username +
  password, globally unique usernames, and internal email surrogate mapping.
- v1.0 has no public self-signup for Organizations or Condos.
- v1.1 Shared Platform LINE OA is the default LINE mode for Business/Enterprise.
- Custom LINE OA is schema-ready but not productized until a later
  Enterprise/premium phase.
- CSV is the only bulk import format in v1.0.
- SaaS owner subscription management controls platform access and metrics only;
  resident billing and accounting remain separate.
- Lease contract files and move-out checklist evidence are limited operational
  evidence files, not the full Documents module.

## Related Docs

- [Domain glossary](../CONTEXT.md)
- [v1 Implementation Contract](v1-implementation-contract.md)
- [Deployment Flow](deployment-flow.md)
- [ADR 0001: Support shared and custom LINE Official Accounts](adr/0001-support-shared-and-custom-line-oa.md)
- [ADR 0002: Use LINE multicast for large announcement notifications](adr/0002-use-line-multicast-for-large-announcements.md)
- [ADR 0003: Resident LIFF access goes through Edge Functions](adr/0003-resident-liff-access-through-edge-functions.md)
