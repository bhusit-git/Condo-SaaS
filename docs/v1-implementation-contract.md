# Condo SaaS v1 Implementation Contract

## Purpose

This document is the implementation source of truth for Condo SaaS v1.
The product plan and ADRs explain why the system exists; this contract defines
the minimum data model, authorization boundary, Edge Function surface, storage
rules, notification worker behavior, import semantics, audit requirements, and
tests needed to build it consistently.

The v1 validation slice is:

1. Create Organization and Condo.
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
- Maintenance, billing, facility booking, visitor QR, documents, contacts, full
  incident case management, full custom role builder, staff self-service password
  reset, and Technician role are out of v1 unless a pilot explicitly reopens scope.

## Data Model Contract

All customer-owned data tables include `id uuid primary key`, `created_at`,
`updated_at`, and the tenant keys needed by their scope. Soft-deleted records use
`status` or `ended_at` rather than hard delete unless explicitly noted.

### Tenant And Setup Tables

- `organizations`: `id`, `name`, `status`.
  - Unique: normalized active organization name only if business requires it.
- `condos`: `id`, `organization_id`, `name`, `status`, `line_mode`,
  `shared_line_channel_id`, `active_at`.
  - `status` values: `setup`, `active`, `inactive`.
  - `line_mode` values: `shared`, `custom_pending`, `custom`.
  - v1 creates `shared`; `custom_*` exists for future compatibility only.
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
- `buildings`: `id`, `organization_id`, `condo_id`, `name`, `status`.
  - Unique active building name per Condo.
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
- `staff_users`: `id`, `auth_user_id`, `username`, `internal_email`,
  `display_name`, `status`.
  - `username` is globally unique after normalization.
  - `internal_email` is deterministic and not user-facing, for example
    `<normalized_username>@staff.internal.local`.
- `staff_memberships`: `id`, `organization_id`, `condo_id`, `staff_user_id`,
  `preset_role`, `status`.
  - `preset_role` values: `condo_admin`, `condo_manager`, `juristic_staff`,
    `security_staff`.
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

## Bootstrap And Onboarding Contract

- Platform bootstrap is performed by a service-role seed script or platform admin
  function, not by an unauthenticated public route.
- Bootstrap seeds the Shared Platform LINE channel, protected preset permission
  matrix, default quota settings, storage buckets, and required scheduled worker
  configuration.
- Creating a pilot Condo requires a platform actor. v1 does not add
  Organization-scoped staff membership.
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
- Staff/admin requests use a Supabase session. Server code resolves the actor from
  `auth.uid()` through `staff_users` and active `staff_memberships`.
- RLS policies deny by default and allow only when an active membership or active
  resident relationship grants the requested scope.
- Customer-data tables must include enough tenant keys for RLS to avoid joins
  through nullable or ambiguous paths.
- Critical writes derive target scope server-side. The frontend may submit an
  intended scope, but the server validates role, permission, condo, and allowed
  scope before enqueue.
- Staff permission checks use effective permissions, not route or nav visibility.
- LIFF frontend never uses the anon Supabase client to query customer-data tables.
  It calls Edge Functions/RPC that verify LIFF identity and derive scope.

### Canonical v1 Permissions

- Residents: `residents:read`, `residents:write`, `residents:deactivate`.
- LINE binding: `line_bindings:read`, `line_bindings:approve`,
  `line_bindings:reject`.
- Announcements: `announcements:draft`, `announcements:publish`,
  `announcements:notify_line`, `critical_notifications:send`.
- Parcels: `parcels:receive`, `parcels:pickup`, `parcels:read`.
- Imports/settings/staff: `imports:run`, `imports:apply_deactivation`,
  `staff:manage`, `line_settings:manage`, `condo_settings:manage`.

### Preset Permission Matrix

| Permission | Condo Admin | Condo Manager | Juristic Staff | Security Staff |
| --- | --- | --- | --- | --- |
| `residents:read` | yes | yes | no | no |
| `residents:write` | yes | yes | no | no |
| `residents:deactivate` | yes | yes | no | no |
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
- Client-supplied `condo_id` and `unit_id` are accepted only when they are in the
  derived allowed-context list.
- Auto-bind requires exactly one active Unit Resident match for
  `condo_id + building/unit + normalized_phone`.
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
- `staff_login_resolve_username`: input `username`; output `internal_email`.
- `staff_reset_password`: input `staff_user_id`, password reset mode; requires
  `staff:manage`; writes audit.
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
  LINE requests, classifies outcomes, updates counters, and schedules retries.
- `notification_resend`: manual resend for authorized operators when a delivery is
  retryable or `outcome_unknown`.

## Notification Delivery Contract

- Prefer LINE reply when the request has a valid reply token and the workflow can
  answer inside the active interaction.
- Reply messages may only be sent from a persisted, unexpired
  `line_webhook_events.reply_token` associated with the same LINE channel and user.
- Use individual push for parcels, explicit maintenance notifications once
  maintenance ships, and small announcements under 20 recipients.
- Use multicast for normal/broadcast announcements with 20 or more recipients,
  batch max 500 recipients.
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
- Object paths include tenant and entity scope:
  - `org/<organization_id>/condo/<condo_id>/parcels/<parcel_id>/<file_id>`.
  - `org/<organization_id>/condo/<condo_id>/announcements/<announcement_id>/<file_id>`.
  - `org/<organization_id>/condo/<condo_id>/imports/<import_batch_id>/<file_id>`.
- All reads use short-lived signed URLs generated server-side after authorization.
- Allowed image MIME types: `image/jpeg`, `image/png`, `image/webp`.
- Max image size: 10 MB for Parcel photos, 5 MB for Announcement images.
- Import uploads accept UTF-8 CSV only.
- Storage object paths are never trusted as authorization proof; table ownership
  and actor scope are checked first.

## Import Contract

### `unit_layout.csv`

Required columns: `building`, `floor`, `unit_no`.

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
- LINE webhook blocked/unreachable state changes when they change an active
  binding's `reachable_state`.
- LINE auto-bind and staff review decision, including match evidence and reviewer.
- Staff permission grant/deny and preset role assignment changes.
- Import apply, stale override, and deactivation confirmation.
- Resident or Unit Resident deactivation and LINE Binding impact summary.
- Parcel pickup, including pickup name/note/photo path when present.

## Acceptance Test Matrix

- RLS isolation denies cross-Organization, cross-Condo, and cross-Unit access for
  admin, staff, and resident paths.
- Frontend route bypass is blocked backend-side.
- `/liff` revoked binding, inactive Unit Resident, invalid selected context, and
  missing context cannot read customer data.
- Multi-Condo or multi-Unit Residents receive a context list and must select
  context before Unit-scoped workflows.
- Staff/admin username is globally unique; login resolves internal email; Condo
  Admin reset works; self-service reset is unavailable in v1.
- Bootstrap seeds Shared LINE, protected presets, first Condo Admin, default
  notification settings, and activation checks.
- Permission matrix matches the preset table; `deny` override wins over `grant`,
  and hidden nav is not treated as authorization.
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
- Storage covers unauthorized media read denial, signed URL generation after
  authorization, invalid MIME rejection, and oversized file rejection.
- Vertical validation slice passes end to end: create Condo, import data, bind
  LINE, receive Parcel, enqueue notification, deliver or record skip/failure, and
  read Parcel from LIFF under the selected Unit context.
