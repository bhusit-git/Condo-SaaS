# Condo SaaS v1 Implementation Contract

## Purpose

This document is the implementation source of truth for Condo SaaS v1.
The product plan and ADRs explain why the system exists; this contract defines
the minimum data model, authorization boundary, Edge Function surface, storage
rules, notification worker behavior, import semantics, audit requirements, and
tests needed to build it consistently.

The v1 validation slice is:

1. Create Organization and Condo with required Condo profile fields.
2. Import Building, Unit, Resident, and Unit Resident data.
3. Bind a Resident to LINE through the Shared Platform LINE OA.
4. Receive a Parcel for the Resident's Unit.
5. Enqueue and deliver the Parcel LINE notification.
6. Let the Resident open LIFF and read the Parcel status.

## Core Defaults

- Stack: React Vite apps for `/admin`, `/staff`, and `/liff`; Supabase Postgres,
  Auth, Storage, and Edge Functions; LINE LIFF and Messaging API.
- v1 LINE mode: Shared Platform LINE OA by default. Custom LINE OA fields are
  schema-ready only; no Custom OA admin setup, migration UI, or full re-bind flow
  ships in v1.
- Staff/admin auth: Supabase Auth session with a globally unique username mapped
  to an internal email surrogate.
- Resident LIFF auth: LIFF identity is verified server-side. LIFF workflows do
  not query customer-data tables directly from the frontend in v1.
- Notification queue: Postgres-backed queue tables processed by a scheduled Edge
  Function worker.
- Platform control plane: v1 includes a minimal owner-only area under
  `/admin/platform` for tenant access status, subscription state, and delayed
  usage metrics. It does not add a separate frontend entrypoint in v1.
- Maintenance and cleaning requests with Technician and Housekeeping Staff roles
  are v1.1 scoped capabilities, not v1 pilot behavior. Resident billing,
  facility booking, visitor QR, documents, contacts, full incident case
  management, full custom role builder, and staff self-service password reset are
  out of v1 unless a pilot explicitly reopens scope.
- v1 scope is reopened only for billing settings configuration. v1 may store
  Condo-scoped rent, utility, and late-fee settings for future billing use, but
  it does not define invoice creation, bill due-date calculation, partial
  payment, late-fee accrual, payment allocation, resident billing Edge
  Functions, accounting rules, payment collection, payment gateway behavior, or
  rent/water/electricity LINE notifications.
- Platform subscription management is only for the SaaS owner controlling whether
  an Organization or Condo may keep using the product. It must not be reused for
  resident rent, water, electricity, payment collection, or accounting workflows.
- Lease contract files and move-out checklist evidence in v1 are limited
  operational evidence files for tenant turnover. They are not the full Documents
  module and do not create a resident billing ledger.

## Data Model Contract

All customer-owned data tables include `id uuid primary key`, `created_at`,
`updated_at`, and the tenant keys needed by their scope. Soft-deleted records use
`status` or `ended_at` rather than hard delete unless explicitly noted.

### Tenant And Setup Tables

- `organizations`: `id`, `name`, `status`, `platform_access_status`.
  - Unique: normalized active organization name only if business requires it.
  - `platform_access_status` values: `allowed`, `suspended`, `cancelled`.
  - `platform_access_status` is the runtime access flag used by RLS helpers and
    Edge Functions. It is not the subscription history.
  - In accommodation-owner language, an Organization may represent the owner
    company or HQ that operates multiple Condos/properties/branches.
- `condos`: `id`, `organization_id`, `name`, `address_line`, `province`,
  `postal_code`, `status`, `line_mode`, `shared_line_channel_id`, `active_at`,
  `platform_access_status`.
  - In dorm, apartment, or small-accommodation language, a Condo may be presented
    as one property, accommodation site, or branch. Do not add a separate `branch`
    table in v1 unless scope is explicitly reopened.
  - Required setup fields: `name`, `address_line`, `province`, `postal_code`.
    Condo activation is blocked until these fields are present.
  - Building, floor, and unit baseline data belongs to the room-layout flow, not
    the Condo profile form.
  - `status` values: `setup`, `active`, `inactive`.
  - `status` is onboarding/lifecycle state only; do not add subscription states
    such as `suspended` to it.
  - `platform_access_status` values: `allowed`, `suspended`, `cancelled`.
  - Effective Condo access is allowed only when both the Organization and Condo
    platform access statuses are `allowed`.
  - `line_mode` values: `shared`, `custom_pending`, `custom`.
  - v1 creates `shared`; `custom_*` exists for future compatibility only.
- `tenant_subscriptions`: `id`, `organization_id`, `condo_id`, `status`,
  `plan_code`, `billing_period_start`, `billing_period_end`, `trial_ends_at`,
  `suspend_reason`, `cancelled_at`.
  - `condo_id` is nullable only when the subscription applies to the whole
    Organization.
  - `status` values: `trialing`, `active`, `past_due`, `suspended`, `cancelled`.
  - This table records business state and history for the SaaS owner. RLS
    policies and high-frequency operational writes must not join this table.
  - Suspend/reactivate/cancel operations update `tenant_subscriptions` and the
    relevant `platform_access_status` flag in the same transaction and write audit.
- `condo_notification_settings`: `id`, `organization_id`, `condo_id`,
  `parcel_line_recipient_policy`, `critical_fallback_enabled`,
  `critical_soft_limit_bypass_enabled`.
  - `parcel_line_recipient_policy` values: `owner_and_tenant`,
    `all_active_residents`, `owner_only`, `tenant_only`, `none`.
  - Default Parcel LINE policy is `owner_and_tenant`.
  - Critical fallback is enabled by default for Shared Platform LINE OA v1.
- `condo_quota_settings`: `id`, `organization_id`, `condo_id`,
  `line_channel_id`, `priority`, `period`, `soft_limit`, `hard_limit`.
  - `period` values: `day`, `month`.
  - Critical notifications may bypass soft limits only when the actor confirms the
    bypass reason; hard limits still apply.
- `billing_rent_rules`: `id`, `organization_id`, `condo_id`, `scope_type`,
  `scope_json`, `cadence`, `amount`, `monthly_due_day`, `status`.
  - This is billing settings configuration only. It must not create invoices,
    invoice line items, ledger entries, receivables, or payment expectations.
  - `scope_type` values: `all_units`, `floor`, `unit_list`.
  - Effective rule precedence is most specific wins: Unit list, then Floor, then
    All Units.
  - `cadence` values: `monthly`, `daily`. Each Unit has only one effective rent
    cadence.
  - `monthly_due_day` is required only for monthly rent rules.
  - Daily rent cadence is stored for future billing use only. v1 does not define
    how daily rent is converted into invoice line items.
- `billing_utility_settings`: `id`, `organization_id`, `condo_id`,
  `utility_type`, `formula`, `unit_rate`, `flat_amount`, `minimum_amount`,
  `minimum_units`, `included_minimum_amount`, `included_minimum_units`,
  `rounding_policy`, `status`.
  - This is Condo default configuration only; v1 does not support per-floor or
    per-unit utility overrides.
  - `utility_type` values: `water`, `electricity`.
  - `formula` values:
    `flat_fee`, `actual_usage`, `actual_with_minimum_amount`,
    `actual_with_minimum_units`, `included_minimum_then_overage`.
  - `actual_usage` preview math is `usage_units * unit_rate`.
  - `actual_with_minimum_amount` preview math is
    `max(usage_units * unit_rate, minimum_amount)`.
  - `actual_with_minimum_units` preview math uses `minimum_amount` when
    `usage_units <= minimum_units`; otherwise it uses `usage_units * unit_rate`.
  - `included_minimum_then_overage` preview math uses
    `included_minimum_amount + max(usage_units - included_minimum_units, 0) *
    unit_rate`.
  - `rounding_policy` values: `whole_baht`, `two_decimals`.
  - Preview/example math in the settings UI is allowed. Persisted meter readings,
    charges, invoices, and utility bill line items are not v1 behavior.
- `billing_late_fee_rules`: `id`, `organization_id`, `condo_id`, `scope_type`,
  `scope_json`, `mode`, `amount`, `grace_days`, `max_fee_amount`, `applies_to`,
  `status`.
  - This is billing settings metadata only. It must not create late-fee charges,
    invoice adjustments, receivables, or payment expectations in v1.
  - `scope_type` values: `all_units`, `floor`, `unit_list`.
  - Effective rule precedence is most specific wins: Unit list, then Floor, then
    All Units.
  - `mode` values: `none`, `fixed_once`, `fixed_per_day`.
  - `grace_days` defaults to `0`.
  - `applies_to` is fixed to `future_invoice` in v1 and is metadata only.
  - `max_fee_amount` is nullable metadata reserved for a future invoice-level cap.
    v1 stores it but does not enforce it.
  - v1 does not support percentage fee, compounding fee, multi-step late-fee
    formulas, late-fee accrual, partial payment, or payment allocation.
- `buildings`: `id`, `organization_id`, `condo_id`, `name`, `status`.
  - Unique active building name per Condo.
  - Building is the canonical entity for a physical building/tower inside one
    Condo/property/branch.
- `units`: `id`, `organization_id`, `condo_id`, `building_id`, `floor`, `unit_no`,
  `status`.
  - Unique active unit identity per `condo_id + building_id + unit_no`.
  - `floor` is explicit layout data, not inferred from `unit_no`.

### Resident And Staff Tables

- `residents`: `id`, `organization_id`, `display_name`, `normalized_phone`,
  `raw_phone`, `status`.
  - Resident identity is organization-scoped.
  - Do not duplicate a Resident per Unit inside the same Organization when the
    same person is known.
- `unit_residents`: `id`, `organization_id`, `condo_id`, `unit_id`,
  `resident_id`, `resident_role`, `status`, `starts_at`, `ends_at`.
  - Active means `status = active` and `ends_at is null`.
  - `resident_role` values: `owner`, `tenant`, `family`.
- `lease_agreements`: `id`, `organization_id`, `condo_id`, `unit_id`,
  `tenant_unit_resident_id`, `owner_unit_resident_id`, `status`, `starts_at`,
  `ends_at`, `signed_contract_object_path`.
  - Status values: `draft`, `active`, `ended`, `terminated`, `cancelled`.
  - `tenant_unit_resident_id` must reference a `unit_residents` row with
    `resident_role = tenant` in the same `organization_id`, `condo_id`, and
    `unit_id`.
  - `owner_unit_resident_id` is nullable. When present, it must reference an
    owner `unit_residents` row in the same `organization_id`, `condo_id`, and
    `unit_id`.
  - A Unit may have at most one active tenant lease in v1.
  - `ends_at` is contract information for staff review only. v1 does not revoke
    tenant access from `ends_at` alone.
- `move_out_checklists`: `id`, `organization_id`, `condo_id`, `unit_id`,
  `lease_agreement_id`, `tenant_unit_resident_id`, `utilities_status`,
  `deposit_status`, `damage_status`, `move_out_status`, `review_note`,
  `confirmed_by_staff_id`, `confirmed_at`, `evidence_object_paths`.
  - `utilities_status` values: `pending`, `cleared`, `waived`, `blocked`.
  - `deposit_status` values: `pending`, `resolved`, `waived`, `disputed`.
  - `damage_status` values: `pending`, `no_damage`, `charged`, `disputed`.
  - `move_out_status` values: `draft`, `inspecting`, `ready_to_close`,
    `confirmed`, `cancelled`.
  - Checklist status is operational evidence only. It records whether staff can
    close the tenant lifecycle; it is not an invoice, payment, or accounting
    record.
- `staff_users`: `id`, `auth_user_id`, `username`, `internal_email`,
  `display_name`, `status`.
  - `username` is globally unique after normalization.
  - `internal_email` is deterministic and not user-facing, for example
    `<normalized_username>@staff.internal.local`.
- `staff_memberships`: `id`, `organization_id`, `condo_id`, `staff_user_id`,
  `preset_role`, `status`.
  - `preset_role` values: `condo_admin`, `condo_manager`, `juristic_staff`,
    `security_staff`.
  - v1.1 adds `technician` and `housekeeping_staff`.
- `staff_permission_overrides`: `id`, `organization_id`, `condo_id`,
  `staff_membership_id`, `permission`, `effect`.
  - `effect` values: `grant`, `deny`.
  - Effective permissions are preset permissions plus overrides.
  - Precedence is `deny` override, then `grant` override, then preset default.

### LINE Binding Tables

- `line_channels`: `id`, `mode`, `line_channel_id`, `line_liff_id`, `status`.
  - `mode` values: `shared`, `custom`.
  - Shared Platform LINE OA is seeded once.
- `line_webhook_events`: `id`, `line_channel_id`, `line_event_id`,
  `line_user_id`, `event_type`, `reply_token`, `payload_json`, `received_at`,
  `processed_at`, `status`.
  - `line_event_id` is unique per LINE channel when present. If LINE does not
    provide a stable id for an event shape, use a deterministic hash of channel,
    event timestamp, source user, event type, and payload.
  - `reply_token` is stored only while it can be used by the immediate workflow.
  - Status values: `received`, `processed`, `ignored`, `failed`.
- `line_bindings`: `id`, `organization_id`, `condo_id`, `line_channel_id`,
  `line_user_id`, `resident_id`, `status`, `matched_unit_id`, `evidence_json`,
  `review_code_hash`, `review_code_expires_at`, `reachable_state`,
  `superseded_by_id`.
  - Status values: `pending_review`, `code_shown`, `verified`, `bound`,
    `revoked`, `expired`, `rejected`, `superseded`.
  - There is no `unbound` row. Unbound means no active `bound` row.
  - Unique active binding: one `line_user_id + line_channel_id + condo_id` may
    bind to only one active Resident.
  - Same LINE account may bind to different Condos through separate Condo contexts.
  - `reachable_state` values: `reachable`, `blocked_or_invalid`, `unknown`.

### Parcel And Announcement Tables

- `parcels`: `id`, `organization_id`, `condo_id`, `unit_id`, `status`,
  `recipient_name`, `recipient_phone`, `description`, `photo_object_path`,
  `received_by_staff_id`, `received_at`, `picked_up_by_name`, `pickup_note`,
  `pickup_photo_object_path`, `picked_up_by_staff_id`, `picked_up_at`.
  - Status values: `waiting_pickup`, `picked_up`, `cancelled`.
- `announcements`: `id`, `organization_id`, `condo_id`, `status`, `priority`,
  `title`, `body`, `image_object_path`, `target_scope_json`, `published_at`,
  `published_by_staff_id`.
  - Status values: `draft`, `published`, `cancelled`.
  - Priority values: `critical`, `operational`, `broadcast`.
- `announcement_recipients`: `id`, `organization_id`, `condo_id`,
  `announcement_id`, `resident_id`, `unit_id`, `read_at`.
  - Recipients are snapshotted at publish time even when LINE notification is not sent.

### v1.1 Maintenance And Cleaning Tables

- `maintenance_request_settings`: `id`, `organization_id`, `condo_id`,
  `completion_evidence_photo_required`.
  - Default `completion_evidence_photo_required = true`.
  - This setting is Condo-scoped. It must not affect other Condos in the same
    Organization.
- `maintenance_requests`: `id`, `organization_id`, `condo_id`, `unit_id`,
  `request_type`, `location_scope_json`, `reported_by_resident_id`,
  `reported_by_staff_id`, `status`, `priority`, `title`, `description`,
  `resident_visible_note`.
  - `request_type` values: `maintenance`, `cleaning`.
  - Status values: `submitted`, `acknowledged`, `assigned`, `accepted`,
    `in_progress`, `resolved`, `closed`, `rejected`, `cancelled`.
  - `unit_id` is required for unit-related requests and nullable for common-area
    requests.
- `maintenance_request_assignments`: `id`, `organization_id`, `condo_id`,
  `maintenance_request_id`, `assigned_staff_membership_id`,
  `assigned_by_staff_id`, `assigned_at`, `accepted_at`, `started_at`,
  `resolved_at`.
  - v1.1 requires an assignment before a Technician or Housekeeping Staff member
    can accept work.
  - Technician assignments are valid only for `request_type = maintenance`.
  - Housekeeping Staff assignments are valid only for `request_type = cleaning`.
- `maintenance_request_evidence`: `id`, `organization_id`, `condo_id`,
  `maintenance_request_id`, `uploaded_by_staff_id`, `object_path`, `note`,
  `created_at`.
  - Resolution evidence belongs to the request and is read through signed URLs
    after authorization.

### Import And Audit Tables

- `import_batches`: `id`, `organization_id`, `condo_id`, `template_type`,
  `scope_json`, `status`, `uploaded_by_staff_id`, `source_file_object_path`,
  `summary_json`.
  - `template_type` values: `unit_layout`, `residents`.
  - Status values: `uploaded`, `validated`, `ready_to_apply`, `partially_applied`,
    `applied`, `rejected`, `failed`.
- `import_rows`: `id`, `import_batch_id`, `row_number`, `raw_json`,
  `normalized_json`, `diff_status`, `error_json`, `stale_conflict`,
  `target_entity_type`, `target_entity_id`, `baseline_updated_at`.
  - `diff_status` values: `add`, `change`, `missing`, `unchanged`, `invalid`.
- Import-managed tables that can be edited in the web app include
  `last_import_batch_id`, `last_import_row_id`, `last_import_applied_at`, and
  `last_web_edit_at`.
  - Direct web edits update `last_web_edit_at`.
  - Import apply updates the import baseline fields.
  - `import_validate` sets `stale_conflict = true` when a target row has
    `last_web_edit_at > last_import_applied_at` for the baseline being overwritten.
- `audit_events`: `id`, `organization_id`, `condo_id`, `actor_type`,
  `actor_id`, `action`, `entity_type`, `entity_id`, `before_json`,
  `after_json`, `evidence_json`, `created_at`.
  - Required for critical notification enqueue, binding auto-bind/review decision,
    staff permission change, import apply/override, resident deactivation, and
    parcel pickup.

### Notification Tables

- `notification_jobs`: `id`, `organization_id`, `condo_id`, `priority`,
  `source_type`, `source_id`, `line_channel_id`, `method`, `payload_json`,
  `target_scope_json`, `status`, `idempotency_key`, `scheduled_at`,
  `attempt_count`, `locked_by`, `locked_until`, `last_error_json`.
  - `method` values: `reply`, `individual_push`, `multicast`.
  - Status values: `queued`, `sending`, `sent_confirmed`, `sent_assumed`,
    `failed_batch`, `outcome_unknown`, `dead_letter`, `cancelled`.
  - `idempotency_key` is unique and stable for the source event plus delivery
    intent, for example `parcel:<parcel_id>:line:new`.
- `notification_deliveries`: `id`, `notification_job_id`, `organization_id`,
  `condo_id`, `resident_id`, `line_binding_id`, `line_user_id`, `status`,
  `provider_request_id`, `provider_response_json`, `attempt_count`,
  `last_error_json`.
  - Status values: `queued`, `sending`, `sent_confirmed`, `sent_assumed`,
    `failed_individual`, `outcome_unknown`, `dead_letter`,
    `skipped_no_line_binding`, `skipped_blocked_or_unknown`.
- `line_quota_counters`: `id`, `organization_id`, `condo_id`, `line_channel_id`,
  `period_start`, `period_end`, `priority`, `sent_count`, `soft_limit`,
  `hard_limit`.
  - These counters support quota enforcement for LINE sending. They are separate
    from platform analytics snapshots.
- `platform_usage_daily`: `id`, `usage_date`, `organization_id`, `condo_id`,
  `active_condo_count`, `resident_count`, `bound_line_user_count`,
  `notification_job_count`, `notification_delivery_count`, `line_message_count`.
  - The unique key is `usage_date + organization_id + condo_id`, with `condo_id`
    nullable only for Organization-level rollups.
  - Rows are written by an hourly or daily aggregate job. Do not update this table
    from per-row triggers or inside high-volume notification delivery loops.
  - Source tables remain the authority for live operational state; this table is
    delayed analytics for the platform owner.

## Bootstrap And Onboarding Contract

- Platform bootstrap is performed by a service-role seed script or platform admin
  function, not by an unauthenticated public route.
- Bootstrap seeds the Shared Platform LINE channel, protected preset permission
  matrix, default quota settings, storage buckets, and required scheduled worker
  configuration.
- Creating a pilot Condo requires a platform actor. v1 does not add
  Organization-scoped staff membership.
- The data model supports one Organization with multiple Condos/properties/branches
  and each Condo with multiple Buildings. Customer-side Owner/HQ overview and
  Organization-scoped staff membership are product directions for multi-site
  customers, but they are not part of the v1 validation slice unless scope is
  explicitly reopened.
- `platform_create_condo` creates the Organization when needed, creates the Condo
  in `setup` status, attaches the Shared Platform LINE channel, creates default
  `condo_notification_settings` and `condo_quota_settings`, creates the first
  Condo Admin staff user, assigns preset permissions, and writes audit events.
- Condo activation sets `condos.active_at` only after required checks pass:
  at least one active Condo Admin, Shared LINE channel attached, default
  notification settings present, at least one active Building and Unit, and LIFF
  deep-link/QR context generated.
- v1 does not expose public self-signup for Organizations or Condos.

## Authorization And RLS Contract

- Application routes are UX conventions only. Backend checks and RLS are the
  security boundary.
- Customer-side Owner/HQ access for multi-site organizations must be modeled as
  Organization-scoped authorization when introduced. It must not reuse Platform
  Super Admin authority and must not be confused with the resident `owner` role.
- Staff/admin requests use a Supabase session. Server code resolves the actor from
  `auth.uid()` through `staff_users` and active `staff_memberships`.
- Platform owner requests use Supabase Auth with
  `app_metadata.role = "platform_super_admin"` plus a short-lived session. This
  claim is set or removed only by a service-role/admin-only script or function.
- Sensitive platform actions must also reject tokens older than the latest
  platform admin grant/revocation version or revocation timestamp recorded by the
  service-role path. Removing platform access must not depend on waiting for an
  old JWT to expire naturally.
- RLS policies deny by default and allow only when an active membership or active
  resident relationship grants the requested scope.
- Customer-data tables must include enough tenant keys for RLS to avoid joins
  through nullable or ambiguous paths.
- Operational write policies and helpers check the Organization/Condo
  `platform_access_status` runtime flags. They must not join
  `tenant_subscriptions` on each operational row.
- Critical writes derive target scope server-side. The frontend may submit an
  intended scope, but the server validates role, permission, condo, and allowed
  scope before enqueue.
- Every staff Edge Function that mutates customer data or enqueues LINE
  notifications must call a shared `assert_condo_platform_active(condo_id)` guard
  before writing. RLS is a safety net, not the only suspend enforcement layer.
- Staff permission checks use effective permissions, not route or nav visibility.
- LIFF frontend never uses the anon Supabase client to query customer-data tables.
  It calls Edge Functions/RPC that verify LIFF identity and derive scope.

### Canonical v1 Permissions

- Residents: `residents:read`, `residents:write`, `residents:deactivate`.
- Leases: `leases:read`, `leases:write`, `leases:end`.
- Move-out: `move_out_checklists:review`, `move_out_checklists:confirm`,
  `tenant_access:revoke`.
- LINE binding: `line_bindings:read`, `line_bindings:approve`,
  `line_bindings:reject`.
- Announcements: `announcements:draft`, `announcements:publish`,
  `announcements:notify_line`, `critical_notifications:send`.
- Parcels: `parcels:receive`, `parcels:pickup`, `parcels:read`.
- Imports/settings/staff: `imports:run`, `imports:apply_deactivation`,
  `staff:manage`, `line_settings:manage`, `condo_settings:manage`.
  - Billing settings are managed through `condo_settings:manage`; v1 does not add
    resident billing permissions.

### v1.1 Maintenance And Cleaning Permissions

- Maintenance requests: `maintenance_requests:read_all`,
  `maintenance_requests:read_assigned`, `maintenance_requests:accept_assigned`,
  `maintenance_requests:start_assigned`, `maintenance_requests:resolve_assigned`,
  `maintenance_requests:upload_resolution_evidence`,
  `maintenance_requests:assign`, `maintenance_requests:close`.
- Maintenance request settings: `maintenance_request_settings:manage`.

### v1.1 Maintenance And Cleaning Preset Defaults

| Permission | Condo Admin | Condo Manager | Juristic Staff | Security Staff | Technician | Housekeeping Staff |
| --- | --- | --- | --- | --- | --- | --- |
| `maintenance_requests:read_all` | yes | yes | grant-only | no | no | no |
| `maintenance_requests:read_assigned` | yes | yes | grant-only | no | yes | yes |
| `maintenance_requests:accept_assigned` | yes | yes | grant-only | no | yes | yes |
| `maintenance_requests:start_assigned` | yes | yes | grant-only | no | yes | yes |
| `maintenance_requests:resolve_assigned` | yes | yes | grant-only | no | yes | yes |
| `maintenance_requests:upload_resolution_evidence` | yes | yes | grant-only | no | yes | yes |
| `maintenance_requests:assign` | yes | yes | grant-only | no | no | no |
| `maintenance_requests:close` | yes | yes | grant-only | no | no | no |
| `maintenance_request_settings:manage` | yes | no | no | no | no | no |

- Technician defaults apply only to assigned `maintenance` requests.
- Housekeeping Staff defaults apply only to assigned `cleaning` requests.
- v1.1 does not include worker self-claim queues. Category or skill matching for
  workers to see and claim unassigned work is a future enhancement.

### Preset Permission Matrix

| Permission | Condo Admin | Condo Manager | Juristic Staff | Security Staff |
| --- | --- | --- | --- | --- |
| `residents:read` | yes | yes | no | no |
| `residents:write` | yes | yes | no | no |
| `residents:deactivate` | yes | yes | no | no |
| `leases:read` | yes | yes | yes | no |
| `leases:write` | yes | yes | no | no |
| `leases:end` | yes | yes | no | no |
| `move_out_checklists:review` | yes | yes | yes | no |
| `move_out_checklists:confirm` | yes | yes | no | no |
| `tenant_access:revoke` | yes | yes | no | no |
| `line_bindings:read` | yes | yes | no | no |
| `line_bindings:approve` | yes | yes | no | no |
| `line_bindings:reject` | yes | yes | no | no |
| `announcements:draft` | yes | yes | yes | no |
| `announcements:publish` | yes | yes | yes | no |
| `announcements:notify_line` | yes | yes | yes | no |
| `critical_notifications:send` | yes | yes | no | no |
| `parcels:receive` | yes | yes | yes | yes |
| `parcels:pickup` | yes | yes | yes | yes |
| `parcels:read` | yes | yes | yes | yes |
| `imports:run` | yes | yes | no | no |
| `imports:apply_deactivation` | yes | yes | no | no |
| `staff:manage` | yes | no | no | no |
| `line_settings:manage` | yes | no | no | no |
| `condo_settings:manage` | yes | no | no | no |

- Security Staff may receive a `grant` override for
  `critical_notifications:send`, but server-side scope validation still limits
  Security-sent critical messages to all Condo, Building, or Floor scopes.
- A `deny` override removes a permission even when the preset grants it.

## Tenant Lease And Move-Out Contract

- `unit_residents` is the resident-to-unit access relationship. `lease_agreements`
  is the tenant contract history for that relationship.
- v1 supports one active tenant lease per Unit. Multi-tenant lease participants
  and shared leases are post-v1.
- Activating a new tenant lease is blocked when the Unit already has an active
  tenant lease or an active tenant `unit_residents` row tied to a previous lease.
  Owner and family `unit_residents` rows do not block tenant lease activation.
- Resident import creates or updates `residents` and `unit_residents` only. It
  does not create lease rows in v1. Staff must create and activate a lease from
  the admin workflow before lease-gated tenant access is enforced for that Unit.
- For existing pilot or imported tenant data, a Condo may enable lease-gated LIFF
  access only after staff backfills active lease rows for active tenant
  `unit_residents`. Until that Condo-level lease workflow is enabled, imported
  tenants keep the existing active `line_bindings + unit_residents` LIFF behavior.
- v1 does not automatically end leases or revoke tenant access when `ends_at`
  passes. Staff must explicitly close the lease or confirm move-out through a
  backend function.
- `staff_close_tenant_move_out` closes the move-out lifecycle in one transaction:
  confirms the checklist, sets the lease to `ended` or `terminated`, ends the
  tenant `unit_residents` row, revokes active tenant `line_bindings`, and writes
  audit events.
- New tenant LINE binding for a Unit is blocked while that Unit still has an
  active tenant lease or active tenant `unit_residents` row tied to the previous
  lease. Owner and family access must not be revoked by tenant move-out.

## Staff/Admin Auth Contract

- Login form accepts `username` and `password`.
- The client normalizes username by trimming and lowercasing before submitting to
  a login helper.
- The login helper resolves `username -> internal_email`, then signs in with
  Supabase Auth email/password.
- The internal email is not displayed in the product UI.
- Username changes are post-v1 unless explicitly implemented with audit and
  internal email remapping.
- Condo Admin password reset is a staff-management action that sets or invites a
  new Supabase Auth password. There is no self-service staff password reset in v1.

## LIFF And LINE Binding Contract

- LIFF requests submit a LINE ID token or access token to an Edge Function.
- The Edge Function verifies the token with LINE, resolves `line_user_id`, resolves
  the `line_channel_id`, and derives allowed Condo/Unit contexts from active
  `line_bindings` plus active `unit_residents`.
- For tenant `unit_residents` in a Condo where lease-gated access is enabled, an
  allowed LIFF context also requires an active `lease_agreements` row for the same
  `unit_id` and `tenant_unit_resident_id`, and no confirmed move-out checklist
  that has revoked access.
- Client-supplied `condo_id` and `unit_id` are accepted only when they are in the
  derived allowed-context list.
- Auto-bind requires exactly one active Unit Resident match for
  `condo_id + building/unit + normalized_phone`.
- Tenant auto-bind is blocked when the target Unit has an active tenant lease or
  active tenant `unit_residents` row for a different tenant that has not been
  closed through the move-out flow.
- Name-only, phone mismatch, and multiple phone matches create `pending_review`.
- Automatic binding records `evidence_json` describing unit match, phone match,
  normalization, candidate count, and actor.
- Staff-assisted review code is used only for pending or uncertain binding. It is
  never required for high-confidence auto-bind.
- Duplicate active binding for the same `line_user_id + line_channel_id + condo_id`
  and a different Resident is hard-blocked.

## LINE Webhook Contract

- `line_webhook_ingest` is the only public LINE webhook endpoint in v1.
- The webhook verifies the LINE signature before reading or persisting event data.
- The webhook resolves the receiving `line_channel_id` from the configured route or
  channel secret, then records an idempotent `line_webhook_events` row.
- Webhook events are deduped by `line_channel_id + line_event_id` or the fallback
  deterministic event hash.
- Follow, message, postback, and LIFF-related events may provide active reply
  tokens for immediate workflows. Reply tokens are not used after their provider
  validity window.
- Unfollow, blocked, invalid-recipient, or provider feedback events update matching
  active `line_bindings.reachable_state` to `blocked_or_invalid` when the LINE user
  and channel are known.
- Webhook processing must not trust client-supplied Condo context. Condo context is
  derived from deep-link/postback state and then validated against active Condos and
  LINE channel mode.

## Edge Function And RPC Surface

All functions return JSON with this envelope:

```json
{
  "ok": true,
  "data": {},
  "error": null
}
```

Errors return:

```json
{
  "ok": false,
  "data": null,
  "error": {
    "code": "permission_denied",
    "message": "Human-readable message",
    "details": {}
  }
}
```

Required error codes: `unauthenticated`, `permission_denied`, `not_found`,
`validation_failed`, `conflict`, `rate_limited`, `provider_error`,
`outcome_unknown`.

### Required v1 Functions

- `platform_create_condo`: create or attach Organization, create Condo defaults,
  seed first Condo Admin, attach Shared LINE channel, and write audit; platform
  actor/service-role only.
- `platform_activate_condo`: validate onboarding checks and set `active_at`;
  requires platform actor or an active Condo Admin membership with
  `condo_settings:manage` for that Condo.
- `platform_list_tenants`: list Organizations/Condos, subscription state,
  platform access status, and latest usage snapshot; platform super admin only.
- `platform_update_subscription_status`: change tenant subscription state and
  runtime access flags transactionally; platform super admin only; requires reason
  and writes audit.
- `platform_get_usage_metrics`: read global and per-tenant delayed usage metrics;
  platform super admin only.
- `platform_aggregate_usage_tick`: scheduled aggregate job that reads source
  tables and upserts `platform_usage_daily`; service-role/scheduler only.
- `staff_login_resolve_username`: input `username`; output `internal_email`.
- `staff_reset_password`: input `staff_user_id`, password reset mode; requires
  `staff:manage`; writes audit.
- `staff_create_lease`: create or update draft lease evidence for a tenant Unit
  Resident; requires `leases:write`; writes audit.
- `staff_activate_lease`: activate a lease only when no active tenant lease or
  prior active tenant row blocks the Unit; requires `leases:write`; writes audit.
- `staff_review_move_out_checklist`: update move-out checklist statuses, note, and
  evidence; requires `move_out_checklists:review`; writes audit.
- `staff_close_tenant_move_out`: confirm checklist, end lease, end tenant Unit
  Resident, revoke tenant LINE Binding, and write audit in one transaction;
  requires `move_out_checklists:confirm`, `leases:end`, and
  `tenant_access:revoke`.
- `liff_resolve_contexts`: input LINE token and optional selected context; output
  active Condo/Unit contexts.
- `liff_bind_line`: input LINE token, condo deep-link context, unit identifier,
  name, phone; output binding status and review instructions if needed.
- `staff_review_line_binding`: approve/reject pending binding; requires
  `line_bindings:approve` or `line_bindings:reject`; writes audit.
- `line_webhook_ingest`: verify LINE signature, persist/dedupe inbound event,
  route reply-capable workflows, and update reachable state when applicable.
- `staff_receive_parcel`: create parcel; requires `parcels:receive`; optionally
  stores photo and enqueues notification according to Condo parcel settings.
- `staff_pickup_parcel`: mark pickup; requires `parcels:pickup`; writes audit.
- `liff_list_parcels`: list Parcels for selected active Unit context.
- `staff_publish_announcement`: snapshot recipients, publish, and optionally
  enqueue LINE notification; requires publish/notify permissions.
- `import_validate`: validate uploaded CSV and generate diff rows.
- `import_apply`: apply selected add/change/deactivation rows; requires
  `imports:run` and `imports:apply_deactivation` for deactivation rows.
- `notification_worker_tick`: scheduled worker that locks queued jobs, dispatches
  LINE requests, classifies outcomes, updates quota-enforcement counters when
  needed, and schedules retries. It must not update `platform_usage_daily`.
- `notification_resend`: manual resend for authorized operators when a delivery is
  retryable or `outcome_unknown`.

### v1.1 Maintenance And Cleaning Functions

- `staff_assign_maintenance_request`: assign a submitted or acknowledged
  Maintenance Request to a Technician or Housekeeping Staff membership; requires
  `maintenance_requests:assign`; writes audit.
- `staff_accept_maintenance_request`: assigned worker accepts the request;
  requires `maintenance_requests:accept_assigned` and active assignment.
- `staff_start_maintenance_request`: assigned worker marks work in progress;
  requires `maintenance_requests:start_assigned` and accepted assignment.
- `staff_resolve_maintenance_request`: assigned worker resolves the request with
  optional note and evidence photos; requires
  `maintenance_requests:resolve_assigned` and
  `maintenance_requests:upload_resolution_evidence` when uploading evidence.
  When `completion_evidence_photo_required` is enabled for the Condo, at least
  one evidence photo is required.
- `staff_close_maintenance_request`: authorized staff reviews and closes a
  resolved request; requires `maintenance_requests:close`; writes audit.
- `staff_update_maintenance_request_settings`: update the Condo-scoped evidence
  photo requirement; requires `maintenance_request_settings:manage`; writes
  audit.
- `liff_create_maintenance_request`: create a Maintenance Request for the
  selected active Unit context or allowed common-area context.
- `liff_list_maintenance_requests`: list resident-visible Maintenance Requests
  and status history for the selected active Unit context.

## Notification Delivery Contract

- Prefer LINE reply when the request has a valid reply token and the workflow can
  answer inside the active interaction.
- Reply messages may only be sent from a persisted, unexpired
  `line_webhook_events.reply_token` associated with the same LINE channel and user.
- Use individual push for parcels, explicit maintenance notifications once
  maintenance ships, and small announcements under 20 recipients.
- Use multicast for normal/broadcast announcements with 20 or more recipients,
  batch max 500 recipients.
- In v1, the notification worker supports Parcel and Announcement notification
  jobs. Billing notification jobs for rent, water, electricity, due-date
  reminders, and overdue reminders are outside v1.
- Do not use LINE broadcast in SaaS v1; all recipients come from platform snapshots.
- Shared OA messages must include Condo context in the message text.
- Critical notifications may bypass Condo soft limits but never provider limits,
  platform hard caps, audit, confirmation, or role/Condo rate limits.
- Parcel LINE notification recipients are selected from active Unit Residents using
  `condo_notification_settings.parcel_line_recipient_policy`.
- Security-sent critical messages are limited to all Condo, Building, or Floor scopes.
- The worker groups recipients by LINE channel, Condo context, payload, locale,
  and delivery method before dispatch.
- Permanent blocked/invalid recipients are terminal: update `reachable_state` and
  skip future LINE sends until state changes.
- Rate limits follow provider retry timing.
- Confirmed transient timeout/5xx before provider dispatch retries with bounded
  backoff: default 1 minute, 5 minutes, 15 minutes, then dead letter.
- `outcome_unknown` for non-critical multicast/broadcast does not auto-retry.
  Staff may manually resend from the notification record.
- `outcome_unknown` for critical notifications may retry or fallback to individual
  push when configured. Duplicate critical delivery is acceptable only when it is
  traceable by notification id.

## Storage Contract

- Buckets:
  - `parcel-photos`: private.
  - `announcement-media`: private.
  - `import-files`: private.
  - `lease-documents`: private.
  - `move-out-evidence`: private.
  - `maintenance-request-evidence`: private.
- Object paths include tenant and entity scope:
  - `org/<organization_id>/condo/<condo_id>/parcels/<parcel_id>/<file_id>`.
  - `org/<organization_id>/condo/<condo_id>/announcements/<announcement_id>/<file_id>`.
  - `org/<organization_id>/condo/<condo_id>/imports/<import_batch_id>/<file_id>`.
  - `org/<organization_id>/condo/<condo_id>/units/<unit_id>/leases/<lease_id>/contract/<file_id>`.
  - `org/<organization_id>/condo/<condo_id>/units/<unit_id>/move-outs/<checklist_id>/<file_id>`.
  - `org/<organization_id>/condo/<condo_id>/maintenance-requests/<request_id>/<file_id>`.
- All reads use short-lived signed URLs generated server-side after authorization.
- Allowed image MIME types: `image/jpeg`, `image/png`, `image/webp`.
- Lease documents and move-out evidence additionally allow `application/pdf`.
- Max image size: 10 MB for Parcel photos, 5 MB for Announcement images, 10 MB
  for move-out evidence images, 10 MB for Maintenance Request evidence images,
  and 15 MB for lease PDFs.
- Import uploads accept UTF-8 CSV only.
- Storage object paths are never trusted as authorization proof; table ownership
  and actor scope are checked first.

## Import Contract

### `unit_layout.csv`

Required columns: `building`, `floor`, `unit_no`.

- This template is the baseline for the room-layout screen. It does not carry
  Condo profile fields such as Condo name, address, province, or postal code.
- `building` and `unit_no` are trimmed and required.
- `floor` is required and stored as configured layout text or number; do not infer
  it from `unit_no`.
- Identity key: `condo_id + normalized(building) + normalized(unit_no)`.

### `residents.csv`

Required columns: `building`, `unit_no`, `resident_name`, `phone`,
`resident_role`.

- `resident_role` must be `owner`, `tenant`, or `family`.
- Phone numbers are normalized before matching. Thai local and international
  formats must compare consistently.
- Resident import cannot apply a row that references an unknown Building/Unit.
- Resident matching key in v1 is `organization_id + normalized_phone` when phone
  is present; ambiguous or missing phone requires manual review instead of merge.
- Resident import does not create lease agreements in v1. Tenant lease creation is
  a staff workflow after the tenant Unit Resident exists.

### Diff And Apply Rules

- Import flow: upload, validate, generate diff, admin review, apply selected rows.
- `import_scope` declares full Condo, Building/Floor range, or partial update.
- Missing rows are compared only inside scope and are warnings/proposed
  deactivations, never automatic deactivations.
- Added and changed rows can be applied separately from missing/deactivation rows.
- Deactivation requires explicit confirmation, impact summary, and audit.
- Stale CSV conflict occurs when a row would overwrite data edited directly in the
  web app after the previous relevant import. Stale conflicts are blocked unless
  an admin explicitly overrides affected rows or the batch; every override is audited.
- Stale conflict detection compares the imported target row against the target
  entity's baseline fields. A conflict exists when `last_web_edit_at` is later
  than `last_import_applied_at` for that entity and the incoming row changes a
  field that the web edit may have changed.
- Manual stale override records the overridden entity id, previous baseline,
  incoming row, reviewer, and reason in `audit_events.evidence_json`.
- Apply operations must be idempotent by `import_batch_id + row_id + action`.

## Audit Contract

Audit records are append-only. They must include actor, scope, entity, action,
before/after state when relevant, and evidence for automated decisions.

Required audited actions:

- Critical notification enqueue, including reason, sender role/permission, target
  scope, message snapshot, quota-bypass confirmation, and notification id.
- Platform bootstrap, Condo creation, first Condo Admin creation, and Condo
  activation.
- Platform super admin grant/revocation, subscription state changes, and
  Organization/Condo platform access status changes, including reason, before and
  after values, and actor.
- LINE webhook blocked/unreachable state changes when they change an active
  binding's `reachable_state`.
- LINE auto-bind and staff review decision, including match evidence and reviewer.
- Lease create/activate/end, move-out checklist review/confirm, and tenant access
  revoke, including before/after access state and uploaded evidence paths.
- Staff permission grant/deny and preset role assignment changes.
- Import apply, stale override, and deactivation confirmation.
- Resident or Unit Resident deactivation and LINE Binding impact summary.
- Parcel pickup, including pickup name/note/photo path when present.
- Maintenance Request setting changes, assignment, accept/start/resolve, uploaded
  evidence, and close.

## Acceptance Test Matrix

- RLS isolation denies cross-Organization, cross-Condo, and cross-Unit access for
  admin, staff, and resident paths.
- Platform routes and platform functions deny users without the
  `platform_super_admin` claim, and deny stale tokens after platform access
  revocation.
- Suspended or cancelled Organization/Condo access blocks staff operational writes
  and LINE enqueue paths while preserving authorized reads and platform recovery
  actions.
- Operational write enforcement uses runtime platform access flags, not joins to
  `tenant_subscriptions`.
- `platform_aggregate_usage_tick` upserts idempotent usage snapshots and can run
  repeatedly without double counting or row-locking high-volume delivery loops.
- Frontend route bypass is blocked backend-side.
- `/liff` revoked binding, inactive Unit Resident, invalid selected context, and
  missing context cannot read customer data.
- Tenant LIFF access in a Condo with lease-gated access enabled requires active
  binding, active tenant Unit Resident, active lease for the same Unit Resident,
  and no confirmed move-out revoke.
- Imported tenant data keeps existing LIFF behavior until lease rows are
  backfilled and lease-gated access is enabled for that Condo.
- Multi-Condo or multi-Unit Residents receive a context list and must select
  context before Unit-scoped workflows.
- Multi-site owner/HQ behavior is covered at the model boundary by one
  Organization containing multiple Condos/properties/branches and each Condo
  containing multiple Buildings. If the Owner/HQ dashboard is implemented in v1,
  tests must verify Organization-level overview first, explicit Condo/property/
  branch selection before site-scoped operations, and no privilege leakage into
  Platform Super Admin or resident `owner` behavior.
- Staff/admin username is globally unique; login resolves internal email; Condo
  Admin reset works; self-service reset is unavailable in v1.
- Bootstrap seeds Shared LINE, protected presets, first Condo Admin, default
  notification settings, and activation checks.
- Permission matrix matches the preset table; `deny` override wins over `grant`,
  and hidden nav is not treated as authorization.
- Lease permission tests cover Admin/Manager full access, Juristic read/review
  only, and Security denied by default.
- Tenant move-out close is transactional: lease ended, tenant Unit Resident ended,
  tenant LINE Binding revoked, and audit written together. `ends_at` alone never
  revokes access.
- Active tenant lease uniqueness blocks overlapping active leases for one Unit,
  while owner and family access remains unaffected.
- LINE binding covers exact unit+phone auto-bind, name-only review, ambiguous
  phone review, inactive reject, duplicate same-Condo hard block, and multi-Condo
  same-LINE allow.
- LINE webhook verifies signature, dedupes events, accepts unexpired reply tokens
  only for the same channel/user, and updates blocked/unreachable bindings.
- Import covers unknown Unit block, scoped diff, partial import warning, stale CSV
  conflict block using import baseline fields, explicit stale override with audit,
  idempotent retry, and deactivation impact confirmation.
- Notification covers reply preference, Parcel recipient settings, multicast
  `sent_assumed`, individual push `sent_confirmed`, blocked skip, provider
  retry classification, non-critical `outcome_unknown` manual resend, critical
  fallback/idempotency, and per-Condo quota.
- Billing settings validation covers rent-rule precedence, monthly due-day
  requirement, daily rent stored without invoice conversion, utility formula
  configuration, settings-screen preview math, rounding policy, late-fee rule
  precedence, `grace_days = 0`, nullable `max_fee_amount`, and the absence of
  invoice creation, meter-reading persistence, persisted charges, late-fee
  accrual, partial payment, payment allocation, and billing LINE jobs.
- v1.1 Maintenance Request tests cover Technician and Housekeeping Staff preset
  defaults, assigned-only visibility, request-type restrictions, and Security
  denied by default.
- v1.1 Maintenance Request evidence tests cover required-photo rejection when
  enabled, optional-photo success when disabled, per-Condo setting isolation, and
  `maintenance_request_settings:manage` enforcement.
- v1.1 Maintenance Request lifecycle tests cover assign, accept, start, resolve,
  close, audit events, resident-visible status history, and internal staff notes
  hidden from residents.
- Storage covers unauthorized media read denial, signed URL generation after
  authorization, invalid MIME rejection, and oversized file rejection.
- Vertical validation slice passes end to end: create Condo, import data, bind
  LINE, receive Parcel, enqueue notification, deliver or record skip/failure, and
  read Parcel from LIFF under the selected Unit context.
