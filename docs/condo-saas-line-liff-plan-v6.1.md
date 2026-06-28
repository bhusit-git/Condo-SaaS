# Condo SaaS + LINE LIFF Scalable Plan v6.1

## Summary

สร้าง Condo Management SaaS แบบ multi-tenant ด้วย `React Vite + Supabase + Edge Functions + LINE LIFF` สำหรับ `/admin`, `/staff`, `/liff`. v1 โฟกัส onboarding, CSV import, resident binding, announcements, parcels, staff permissions, minimal platform owner controls, and quota-conscious LINE notifications. Maintenance เลื่อนไป v1.1 เพื่อให้ pilot แคบและ ship ได้เร็วขึ้น. Phase 1 ใช้ Shared Platform LINE OA เป็น default; Phase 2 เพิ่ม Custom LINE OA เป็น premium upsell โดย v1 เตรียม schema ให้รองรับ แต่ไม่มี custom OA admin configuration UI, migration UI, หรือ re-bind flow เต็ม.

Implementation details that must not be re-decided during build live in
[v1 Implementation Contract](v1-implementation-contract.md). This plan remains
the product and architecture overview; the contract is the source of truth for
schema, RLS, Edge Function contracts, auth, notification queue, storage, import,
audit, and acceptance tests.

## Key Decisions

- Tenant model: `Organization` = customer/account, `Condo` = โครงการ, `Building -> floor -> Unit` เป็น layout ที่คอนโดกำหนดเอง.
- Resident model: `Resident` เป็น organization-scoped identity; คนเดียวกันในคนละ organization เป็นคนละ record. `Unit Resident` คือความสัมพันธ์กับห้อง เช่น owner/tenant/family. Tenant lease history อยู่ใน `lease_agreements` และผูกกับ tenant `unit_residents` ไม่ใช่แค่ resident identity.
- Staff model: v1 preset roles คือ `Condo Admin`, `Condo Manager`, `Juristic Staff`, `Security Staff`; role builder เต็ม defer หลัง v1. v1 ใช้ preset roles plus permission toggles สำหรับสิทธิ์สำคัญ. v1.1 เพิ่ม `Technician` และ `Housekeeping Staff` สำหรับงานซ่อม/ทำความสะอาดที่ถูก assign.
- Authorization: route เป็น UX convention เท่านั้น; enforce จริงผ่าน permission checks, Edge Functions/RPC, และ Supabase RLS. Critical writes derive scope server-side เสมอ.
- LINE strategy: Shared Platform LINE OA ใน v1 พร้อม condo-specific QR/LIFF deep link. Custom LINE OA เป็น phase 2; v1 เก็บ `line_channel_id`/mode เพื่อไม่ block อนาคต.
- LINE inbound: v1 มี `line_webhook_ingest` สำหรับ verify signature, dedupe LINE events, resolve channel, capture usable reply token, and update reachable state.
- Notifications: prefer reply when possible, individual push สำหรับ targeted events, multicast สำหรับ announcement ใหญ่, per-condo quota/usage guardrail สำหรับ Shared OA. Parcel recipients and critical fallback behavior come from Condo notification settings.
- Platform control plane: v1 includes a minimal owner-only area at `/admin/platform` for tenant subscription state, access suspension/reactivation, and delayed global usage metrics. It is not resident rent, water, electricity, payment, or accounting billing.
- Frontend architecture: v1 uses separate React Vite apps for `admin`, `staff`, and `liff` inside one workspace. Default deploy entrypoints are path routes `/admin`, `/staff`, and `/liff`; `/admin/platform` stays inside the admin app in v1 instead of adding a fourth deployment entrypoint. The apps share intentional packages for UI primitives, design tokens, domain types, permissions, and Edge Function/RPC clients, but keep routing, auth assumptions, bundles, and UX flows separate. Feature-specific UI such as admin import diff tables, staff camera flows, platform owner views, and LIFF context switchers stays inside the owning app until reuse is proven.
- Staff/admin auth: staff and admin users sign in with custom username + password. Usernames are globally unique across the platform and map to an internal email surrogate for Supabase Auth. v1 password recovery is Condo Admin reset; no self-service password reset in v1.

## Core Data And Flows

- RLS ทุก customer-data table scope ด้วย `organization_id`, `condo_id`, `unit_id` ตาม actor membership.
- Platform access status is a runtime flag on Organization/Condo and is separate
  from subscription history. Operational writes and LINE enqueue paths check this
  flag through shared Edge Function guards and RLS helpers; they do not join
  subscription history tables on every write.
- The v1 validation slice is: create condo -> import unit/resident -> bind LINE -> receive parcel -> enqueue LINE notification -> resident opens LIFF to read parcel status.
- Bootstrap/onboarding is platform-controlled in v1: seed Shared LINE channel and protected presets, create first Condo Admin, create default notification/quota settings, and activate Condo only after required checks pass.
- Canonical v1 permissions:
  - Residents: `residents:read`, `residents:write`, `residents:deactivate`
  - LINE binding: `line_bindings:read`, `line_bindings:approve`, `line_bindings:reject`
  - Announcements: `announcements:draft`, `announcements:publish`, `announcements:notify_line`, `critical_notifications:send`
  - Parcels: `parcels:receive`, `parcels:pickup`, `parcels:read`
  - Imports/settings/staff: `imports:run`, `imports:apply_deactivation`, `staff:manage`, `line_settings:manage`, `condo_settings:manage`
  - Leases/move-out: `leases:read`, `leases:write`, `leases:end`, `move_out_checklists:review`, `move_out_checklists:confirm`, `tenant_access:revoke`
- Preset defaults:
  - `Condo Admin`: setup/settings/staff/LINE plus all condo operations.
  - `Condo Manager`: residents, leases, move-out close, binding approval, announcements, parcels, and imports.
  - `Juristic Staff`: announcements, parcels, lease read, and move-out checklist review by preset; no tenant revoke, lease close, residents/settings/staff/imports by default.
  - `Security Staff`: parcels by preset and broad-scope critical notifications only when granted.
  - Permission toggles may grant or remove sensitive capabilities such as `critical_notifications:send`, `line_bindings:approve`, `staff:manage`, or `line_settings:manage`. A deny override wins over grant and preset defaults. v1 does not include a full custom role builder.
- LINE Binding:
  - No SMS OTP in v1.
  - Auto-bind only when `unit + normalized phone` matches exactly one active Unit Resident.
  - Name-only, phone mismatch, or ambiguous phone goes to review.
  - Manual review uses staff-assisted verification code shown in web app.
  - States: `pending_review`, `code_shown`, `verified`, `bound`, `revoked`, `expired`, `rejected`, `superseded`.
  - “Unbound” means no active binding row.
  - Duplicate LINE account within same `line_channel_id + condo_id` for different resident is hard blocked.
  - Same LINE account may bind to multiple condos through separate condo contexts.
  - Tenant binding is blocked when the Unit still has an active tenant lease or an active previous tenant relationship that has not been closed through move-out.
  - Phase 2 Custom OA re-bind creates a new binding and marks old shared-channel binding `superseded`.

## Product Modules

- Onboarding: create organization/condo, create buildings/floors/units, import CSV, review diff, create preset roles, configure Shared LINE OA mode, generate condo QR/LIFF links, activate condo after required checks.
- Admin operations: unit/resident management, preset role assignment, permission toggles, binding review, announcements, parcels, imports.
- Platform owner operations: view tenants, suspend/reactivate access, inspect
  subscription state, review delayed usage metrics, and audit platform actions
  from `/admin/platform`.
- Billing settings: v1 includes configuration-only rent, utility, and late-fee
  settings for future billing use. It does not create invoices, meter readings,
  persisted charges, payment records, accounting entries, or billing LINE jobs.
- Staff mobile web: optimized `/staff` workflows for parcel receive/pickup and broad-scope critical announcements.
- Tenant & Move-In / Move-Out:
  - `unit_residents` remains the access relationship; `lease_agreements` records
    the tenant contract history for that relationship.
  - v1 allows one active tenant lease per Unit. Multi-tenant lease participants
    are post-v1.
  - Staff creates/activates a lease after the tenant Unit Resident exists. The
    resident CSV import does not create lease rows.
  - Existing pilot/imported tenants keep the old active binding + active Unit
    Resident LIFF behavior until lease rows are backfilled and lease-gated access
    is enabled for that Condo.
  - `ends_at` never revokes access by itself in v1. Staff must confirm move-out
    through a backend function.
  - Move-out checklist records status-only operational evidence for utility
    clearance, deposit, room damage, review note, confirmer, timestamp, and files.
    It is not a billing ledger or the full Documents module.
  - Closing move-out ends the lease, ends the tenant Unit Resident, revokes the
    tenant LINE Binding, and writes audit in one transaction. Owner/family access
    remains unaffected.
- Resident LIFF:
  - condo-specific QR binding.
  - context switcher when resident has multiple condos/units.
  - announcements/read tracking.
  - parcel status.
  - tenant contexts in Condos with lease-gated access enabled require active
    binding, active tenant Unit Resident, active lease, and no confirmed move-out
    revoke.
- Maintenance (v1.1):
  - linked to reporting unit even for common-area category.
  - supports `request_type = maintenance | cleaning`.
  - repeated reports create separate requests.
  - statuses: submitted, acknowledged, assigned, accepted, in progress, resolved, closed, rejected, cancelled.
  - Admin/Manager/authorized Juristic Staff assign work first; Technician and
    Housekeeping Staff see only assigned work in the first release.
  - `resolved` means the worker finished the job; `closed` means authorized
    staff reviewed and closed it.
  - Condo Admin can turn required resolution evidence photos on/off per Condo;
    default is enabled.
  - category-based self-claim queues for workers are a future enhancement.
  - same-unit active residents can see resident-visible request/status history.
  - routine status changes do not push LINE; staff may explicitly notify LINE for important updates.
- Parcels:
  - unit-level record; optional recipient details and optional photo.
  - pickup confirmed by staff; pickup person need not have account/binding.
  - all active unit residents can see parcel in app.
  - LINE notification setting default `owner_and_tenant`; options `all_active_residents`, `owner_only`, `tenant_only`, `owner_and_tenant`, `none`.
- Announcements:
  - targets: all condo, building, floor, unit, unit range, resident role, specific residents.
  - recipient snapshot created on every publish, even without LINE notify.
  - read tracking per resident recipient; staff UI may aggregate by unit.
  - v1 content = text + optional one image.
  - publish modes: publish only, publish + LINE notify, critical notify.
  - full incident case management is post-v1; v1 urgent events are Critical Announcements/Notifications with reason and audit.

## Imports

- v1 supports CSV UTF-8 only; provide downloadable templates.
- Split templates:
  - `unit_layout.csv`: building, floor, unit.
  - `residents.csv`: resident identity and unit relationship.
- Resident import cannot apply rows referencing unknown building/floor/unit. Admin must import/create layout first or resolve missing units in web flow.
- Resident import creates residents and unit relationships only. Tenant lease rows
  are created by staff after import when the Condo uses the lease workflow.
- Import flow: upload -> validate -> generate diff -> admin review -> apply added/changed/deactivation separately.
- `import_scope` declares full condo, building/floor scope, or partial update; missing rows are compared only inside scope.
- Missing rows are warnings/proposed changes, never automatic deactivation.
- Deactivation is soft delete with explicit confirm, audit, and LINE binding impact summary.
- Web app supports direct add/edit for units/residents. Direct edits have audit; import diff detects stale CSV conflicts when uploaded rows may overwrite newer web edits.
- Stale CSV conflicts are detected through import baseline fields and are hard-blocked by default at apply time. Admins may explicitly override affected rows or batches after reviewing the conflict, and every override is audited.

## Notification Delivery

- Priorities: `critical`, `operational`, `broadcast`.
- Shared OA messages must include condo name/context.
- Critical may bypass condo soft limits, but never bypass provider limits, platform hard caps, confirmation, audit, or role/condo rate limiting.
- Security-sent critical messages are limited to broad scopes: all condo, building, floor.
- Every critical notification requires a reason, explicit target scope confirmation, quota-bypass confirmation when applicable, and an audit snapshot of sender, role/permission, target scope, message, and notification id before enqueue.
- Delivery methods:
  - `reply`: active interaction with valid reply token.
  - `individual push`: parcels, explicit maintenance notifications once maintenance ships, small announcements `< 20`.
  - `multicast`: normal/broadcast announcements `>= 20`, batch max 500 recipients.
  - no LINE broadcast in SaaS v1 because recipients must come from platform snapshots.
- Delivery statuses:
  - `queued`, `sending`
  - `sent_confirmed` for individual provider acceptance
  - `sent_assumed` for multicast batch accepted
  - `failed_batch`, `failed_individual`
  - `outcome_unknown` when a provider request left the platform but LINE acceptance/failure is not known
  - `dead_letter`
  - `skipped_no_line_binding`, `skipped_blocked_or_unknown`
- Failure handling:
  - permanent blocked/invalid recipient = terminal, mark unreachable and skip future sends.
  - rate limits follow provider retry timing.
  - confirmed transient timeout/5xx before provider dispatch retries with bounded backoff, default 3 attempts: 1m, 5m, 15m, then dead letter.
  - `outcome_unknown` does not auto-retry for non-critical multicast/broadcast notifications; staff may manually resend from the notification record.
  - `outcome_unknown` for critical notifications may retry or fallback to individual push when configured because missed emergency delivery is worse than rare duplicates.
- notification jobs/deliveries use stable idempotency keys.
- v1 notification jobs and deliveries are persisted in Postgres queue tables and processed by a scheduled Edge Function worker.
- Platform usage metrics are delayed hourly/daily aggregates from source tables
  such as notifications, residents, LINE bindings, and quota counters. They are
  not maintained with per-row realtime triggers or hot-row increments during bulk
  notification sends.
- LINE inbound webhook events are signature-verified and deduped before any reply-token workflow or reachable-state update.
- critical multicast may fallback to individual push after failed retry if configured; rare duplicate critical messages are acceptable compared with missed emergency delivery and must remain traceable by notification id.

## Test Plan

- Authorization/RLS: cross-condo and cross-organization denial for admin/staff/resident; frontend route bypass still blocked backend-side.
- Resident LIFF auth boundary: `/liff` resident workflows call Edge Functions/RPC only; revoked or inactive LINE Binding cannot read resident data; multi-condo/multi-unit residents must select condo/unit context before unit-scoped workflows; `/liff` must not query customer-data tables directly through the Supabase client in v1.
- Tenant lease boundary: lease-gated tenant LIFF contexts require active binding,
  active tenant Unit Resident, active lease for that Unit Resident, and no
  confirmed move-out revoke; imported tenants are not locked out before lease
  backfill/enablement.
- Staff/admin auth: global username uniqueness, username/password login for `/admin` and `/staff`, Condo Admin password reset, and no staff self-service password reset in v1.
- Platform owner auth: `/admin/platform` and platform functions require
  `app_metadata.role = "platform_super_admin"` and reject stale tokens after
  platform access revocation.
- Tenant suspension: suspended/cancelled Organization or Condo blocks operational
  writes and LINE enqueue paths but still allows authorized reads and platform
  recovery actions.
- Bootstrap/onboarding: Shared LINE channel seeded, protected preset matrix seeded, first Condo Admin created, default notification/quota settings created, and Condo activation blocked until checks pass.
- Permission matrix: exact preset role defaults, deny-over-grant override precedence, hidden nav behavior, critical permission alone cannot send without reason/scope confirmation/audit/rate limiting.
- Lease/move-out permissions: Admin/Manager can manage and close tenant move-out;
  Juristic Staff can read/review only by default; Security Staff has no default
  lease or revoke permissions.
- Tenant turnover: one active tenant lease per Unit, no automatic revoke from
  `ends_at`, transactional move-out close, new tenant binding blocked until the
  old tenant is closed, new tenant bind allowed after close, owner/family access
  preserved.
- LINE binding: normalized phone formats, exactly-one active candidate auto-bind, name-only review, ambiguous review, inactive reject, duplicate same-condo hard block, multi-condo same LINE allow, phase-2 supersede semantics.
- LINE webhook: signature verification, event dedupe, same-channel/user reply token handling, and reachable-state update for blocked/unreachable users.
- Import: CSV validation, unknown unit references block apply, layout dependency before resident apply, scoped diff, partial import warning, stale CSV conflict block using baseline fields, explicit stale override apply + audit, idempotent retry, explicit deactivation confirm and audit.
- Notifications: reply preferred, parcel recipient settings from Condo notification settings, publish-only snapshots recipients, multicast accepted records `sent_assumed`, individual push accepted records `sent_confirmed`, unknown provider outcome records `outcome_unknown`, non-critical multicast `outcome_unknown` no auto-retry, critical `outcome_unknown` may retry/fallback with traceable notification id, retry classification, blocked skip, per-condo quota, critical fallback idempotency.
- Parcels: create without photo, optional photo capture, staff pickup with pickup name/note/photo, resident visibility by unit.
- Frontend boundaries: `admin`, `staff`, and `liff` build separately; LIFF bundle does not include admin-only import/table flows; app-specific feature components stay inside the owning app.

## Assumptions

- v1 uses Supabase Auth/session for staff/admin with custom username + password, globally unique usernames, and internal email surrogate mapping. Resident LIFF uses verified LIFF identity, but resident customer-data access goes through Edge Functions/RPC only; Edge Functions resolve actor and scope server-side from active LINE Binding and selected condo/unit context.
- Shared Platform LINE OA is default for phase 1.
- Custom LINE OA is schema-ready in v1 but productized in phase 2. v1 has no custom OA admin configuration UI, migration UI, or re-bind flow.
- CSV is the only bulk import format in v1.
- SaaS owner subscription management is minimal v1 control-plane behavior. It
  controls tenant access and metrics only; invoice lifecycle, payment collection,
  accounting rules, meter-reading persistence, persisted charges, and resident
  utility/rent billing remain outside v1.
- Maintenance and cleaning requests with Technician and Housekeeping Staff roles
  are v1.1 scoped capabilities, not v1 pilot behavior. Resident billing,
  facility booking, visitor QR, documents, full incident case management,
  contacts, full custom role builder, and staff self-service password reset
  remain post-v1 unless pilot demands them.
- Billing / Utility Bills LINE notifications for rent charges, water charges, electricity charges, due-date reminders, and overdue reminders are post-v1. They should reuse the v1 LINE notification queue, LINE Binding, and LIFF access patterns, but require a separate billing runtime contract before implementation.
- Lease contract files and move-out checklist evidence are limited operational
  evidence files in v1, not the full Documents module.

## Related Docs

- [Domain glossary](../CONTEXT.md)
- [v1 Implementation Contract](v1-implementation-contract.md)
- [Deployment Flow](deployment-flow.md)
- [ADR 0001: Support shared and custom LINE Official Accounts](adr/0001-support-shared-and-custom-line-oa.md)
- [ADR 0002: Use LINE multicast for large announcement notifications](adr/0002-use-line-multicast-for-large-announcements.md)
- [ADR 0003: Resident LIFF access goes through Edge Functions](adr/0003-resident-liff-access-through-edge-functions.md)
