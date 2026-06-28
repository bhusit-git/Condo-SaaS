# Context Glossary

## Organization

The customer or account that owns the SaaS relationship.

An Organization may manage one or more Condos.

For accommodation, dorm, or small-property customers, an Organization can
represent the owner company or HQ that operates many locations. Those locations
are modeled as Condos in v1 rather than separate deployments.

## Condo

A condo project managed within an Organization.

A Condo has its own buildings, floors, units, residents, staff scope, and operational configuration.

In Thai product language for dorms or small accommodation businesses, a Condo may
also be called a property, accommodation site, or branch. The canonical system
entity remains Condo unless the implementation contract explicitly introduces a
separate Branch entity.

## Accommodation Owner / HQ

The customer-side business owner or head office that oversees multiple Condos,
properties, accommodation sites, or branches inside one Organization.

Accommodation Owner / HQ is distinct from Platform Super Admin, which belongs to
the SaaS operator, and distinct from the resident role `owner`, which describes a
Resident's relationship to a Unit.

The expected product flow for multi-site customers is an Organization-level
overview first, then choosing the specific Condo/property/branch context before
entering site-scoped operations.

## Platform Super Admin

The SaaS owner operator who can manage tenant access, subscription state, and
global usage metrics across Organizations and Condos.

Platform Super Admin is not a Condo Admin role. It is authorized through a
platform-level Supabase Auth claim and service-role controlled grant/revocation
flow.

In v1, the Platform Super Admin surface lives at `/admin/platform` inside the
admin app. Route visibility is only UX; Edge Functions and RLS helpers enforce
the boundary.

## Platform Subscription

The SaaS relationship between the platform owner and an Organization or Condo.

Platform Subscription controls whether a tenant may continue using the product.
It is separate from resident rent, water, electricity, invoices, payment
collection, and accounting workflows.

Subscription history is recorded separately from the runtime access flag used for
authorization. Operational writes check Organization/Condo platform access flags,
not subscription history joins.

## Platform Usage Metric

A delayed owner-facing snapshot of usage such as active condos, active residents,
LINE bindings, notification jobs, deliveries, and message counts.

Platform Usage Metrics are produced by scheduled aggregate jobs from source
tables. They are not updated by per-row realtime triggers during parcel,
announcement, or LINE delivery workflows.

## Shared Platform LINE OA

The platform-owned LINE Official Account used by multiple Condos.

In phase 1, Condos can use the Shared Platform LINE OA so residents can start quickly without each Condo creating its own LINE Official Account.

Residents enter a Condo context through a condo-specific LIFF deep link or QR code. A Resident may bind the same LINE account to multiple Condos through separate Condo contexts.

Notifications sent through the Shared Platform LINE OA must identify the Condo in the message text so residents with multiple Condo relationships can understand the context.

Shared Platform LINE OA message usage is tracked per Condo so one Condo cannot silently consume capacity needed by other Condos.

When a resident interaction can be answered with a LINE Reply Message, the platform prefers reply over push. Push messages are reserved for events that need proactive delivery outside an active interaction.

LINE notifications have priority levels: critical, operational, and broadcast.

Critical notifications may bypass normal condo soft limits for urgent situations, but require an explicit reason, target-scope confirmation, and audit record. Operational notifications support routine condo workflows. Broadcast notifications are general announcements and should be limited most strictly.

Critical notifications may be sent by roles trusted for urgent condo operations, including Security when granted that permission.

Security-sent Critical Announcements are limited to pre-approved broad scopes such as all condo, building, or floor.

Blocked or invalid LINE recipients are terminal delivery outcomes and are not retried. Retry behavior depends on whether provider failure is confirmed or the provider outcome is unknown.

LINE delivery failures are classified before retry. Permanent failures are dead-lettered immediately, rate limits follow provider retry timing, and unknown provider outcomes are treated separately from confirmed failures.

When a Resident blocks the LINE account or otherwise becomes unreachable through LINE, future LINE notifications to that recipient are skipped until the reachable state changes.

Delivery status distinguishes assumed batch delivery from confirmed individual delivery.

LINE delivery statuses use explicit names such as sent confirmed for individual delivery acceptance and sent assumed for multicast batch acceptance. The domain avoids a generic sent status when delivery certainty differs by method.

## Announcement

A condo message published to a targeted set of Residents.

Announcement recipients are snapshotted at publish time. Read tracking is recorded per Resident recipient, while staff views may aggregate read status by Unit.

Announcement recipient snapshots are created whenever an Announcement is published, regardless of whether LINE notification is sent.

General Announcements target all active Resident roles by default. Staff may target specific resident roles, such as owners, tenants, or family members, when the message is role-specific.

In v1, Announcements contain text and may include one image. LINE notifications for Announcements use short text with a LIFF link to the full Announcement.

Incident case management is outside v1. Urgent situations in v1 are handled as Critical Announcements or Critical Notifications with reason and audit history.

Publishing an Announcement and notifying residents by LINE are separate choices. Staff may publish without LINE notification, publish with normal LINE notification, or use the critical notification path for urgent messages.

Large Announcement notifications may use LINE multicast delivery. Multicast success means LINE accepted the batch, not that every recipient definitely received the message.

## Billing Notification

A future LINE notification for rent charges, utility bills, due-date reminders, or overdue reminders.

Billing Notifications are post-v1. They are expected to reuse the platform notification queue, LINE Binding, and LIFF access pattern, but they are not committed v1 behavior.

## Billing Settings

A v1 configuration-only surface for future resident billing rules.

Billing Settings may store Condo-scoped rent rules, water/electricity formula
settings, rounding policy, and late-fee metadata. They do not create invoices,
persisted charges, meter readings, payments, accounting entries, or LINE billing
notifications in v1.

## Rent Charge

A future billing-domain charge for rent owed by a tenant or responsible Resident relationship.

Rent Charges are post-v1 and do not define v1 invoice lifecycle, payment flow, accounting rules, payment gateway behavior, or daily-rent invoice conversion. v1 Billing Settings may store rent-rule configuration only.

## Utility Bill

A future billing-domain bill for utilities such as water or electricity.

Utility Bills are post-v1 and may later notify residents through LINE when issued, near due date, or overdue. v1 Billing Settings may store water/electricity formula configuration and preview examples only.

## Lease Agreement

A tenant contract history record for a specific tenant Unit Resident.

Lease Agreements are tied to `tenant_unit_resident_id`, not only to Resident
identity, because the same Resident may be a tenant of multiple Units or may have
multiple historical tenant periods.

In v1, a Unit may have at most one active tenant Lease Agreement. Shared leases
with multiple tenant participants are deferred until a later version.

`ends_at` records the contract date for staff review. It does not automatically
revoke LINE or LIFF access in v1.

Staff must explicitly close the lease or confirm move-out through a backend
function before tenant access is revoked.

Lease contract files are limited operational evidence files in v1. They are not
the full Documents module.

## Move-In / Move-Out Checklist

A staff-reviewed operational checklist used when a tenant leaves a Unit and
another tenant may move in.

The checklist records status-only evidence for items such as utility clearance,
deposit resolution, room damage review, notes, confirming staff, timestamps, and
evidence files.

Move-out checklist statuses are not rent, utility bills, invoices, payment
collection, or accounting records.

Closing move-out ends the tenant Lease Agreement, ends the tenant Unit Resident,
revokes the tenant LINE Binding, and writes audit in one backend transaction.

Owner and family Unit Resident access is not revoked by tenant move-out.

## Custom LINE OA

A Condo-specific LINE Official Account connected to the platform.

Custom LINE OA is a premium phase 2 capability for Condos that want their own branding and dedicated LINE channel.

When a Condo moves from the Shared Platform LINE OA to a Custom LINE OA, Residents re-bind through the Custom LINE OA while keeping their existing Resident and Unit relationships.

## Condo Manager

A condo-scoped staff role responsible for day-to-day juristic office operations for a condo.

In Thai product language, this is the canonical meaning of "นิติ" unless explicitly discussing a legal entity or property management company.

## Juristic Staff

A condo-scoped staff role responsible for assigned juristic office workflows without full manager authority.

Juristic Staff may handle operational areas such as announcements, maintenance, or parcels when granted the relevant permissions, but does not automatically receive resident management, staff management, billing, or sensitive settings access.

Juristic Staff is a v1 staff role preset.

## Condo Admin

A condo-scoped staff role responsible for system administration for a condo.

Condo Admins manage setup, staff access, and sensitive condo-level configuration. A person may be both a Condo Admin and a Condo Manager in small condos.

Condo Admin is site-scoped. For multi-site customers, Organization-level Owner/HQ
access must not be treated as the same role unless the contract explicitly grants
that broader scope.

## Security Staff

A condo-scoped staff role responsible for front desk or security operations.

In v1, Security Staff can handle parcel receiving, parcel pickup, and urgent incident notification when granted the relevant permission.

## Technician

A condo-scoped v1.1 staff role responsible for assigned maintenance work.

Technicians can see, accept, start, and resolve only Maintenance Requests assigned
to them and categorized as maintenance work. They do not receive resident
management, staff management, settings, LINE binding, import, parcel, or
announcement permissions by default.

## Housekeeping Staff

A condo-scoped v1.1 staff role responsible for assigned cleaning work.

Housekeeping Staff can see, accept, start, and resolve only Maintenance Requests
assigned to them and categorized as cleaning work. They do not receive resident
management, staff management, settings, LINE binding, import, parcel, or
announcement permissions by default.

## Staff Permission

A condo-scoped staff capability that may be granted in addition to a staff role.

Staff roles act as presets, while sensitive capabilities such as sending critical notifications, managing staff, managing LINE settings, or approving LINE bindings can be controlled as explicit permissions.

Permissions are action-level capabilities, not only module-level access. For example, drafting an announcement and publishing an announcement can be controlled separately.

Preset roles are created during condo onboarding and may be cloned into custom roles later, while protected system presets remain available as defaults.

Technician and Housekeeping Staff are v1.1 preset roles. Maintenance and cleaning
operations remain outside the v1 pilot validation slice.

Application routes are user experience conventions, not the security boundary. Authorization is based on Staff Permissions and resident relationships, and must also be enforced by backend policies and server-side functions.

## LINE Binding

The relationship between a Resident and a LINE account within a Condo context.

LINE Binding can be automatic only when the match is high confidence against an active Unit Resident. Active means the Unit Resident is active and has no ended date.

Duplicate LINE accounts are hard-blocked when they are already bound to another active Resident relationship in the same relevant context.

Every automatic LINE Binding records audit evidence explaining which fields matched and why the binding was accepted.

In v1, automatic LINE Binding requires a unit match and exactly one normalized phone match among active Unit Residents for that Unit. Name-only matches are sent to review.

Tenant LINE Binding is blocked when the Unit still has an active tenant lease or
an active previous tenant Unit Resident that has not been closed through the
move-out workflow.

Phone numbers are normalized before matching so local and international Thai phone formats compare consistently.

If no active Unit Resident exists for the unit, the binding is rejected. If multiple active candidates match the phone, the binding is sent to review as ambiguous data.

A LINE Binding belongs to a LINE channel and a Condo. The same LINE account may bind to different Condos through the same shared channel, but it must not bind to different Residents within the same Condo and LINE channel.

When a Condo moves to a Custom LINE OA, residents bind again through the new LINE channel rather than reusing the old shared-channel binding.

LINE Binding statuses describe actual binding attempts or historical bindings. "Unbound" means there is no active binding row.

Superseded LINE Bindings are historical bindings that were replaced by a newer binding, such as when a Condo moves from the Shared Platform LINE OA to a Custom LINE OA.

LINE Binding does not use SMS OTP in v1. When a code is needed, it is a staff-assisted verification code shown in the web app for condo staff to verify with the resident.

Staff-assisted verification codes are used for pending or uncertain binding requests, not for high-confidence automatic binding.

Pending LINE Binding approval and rejection require an explicit staff permission. Condo Manager has this permission by default.

## Resident

A person who can access condo services through one or more unit relationships.

A Resident is an identity-level concept. The same Resident may be related to multiple units and should not be duplicated per unit.

## Unit Resident

The relationship between a Resident and a Unit.

Unit Residents express the resident's role for that unit, such as owner, tenant, or family member, and determine which units the Resident can access.

In v1, active Unit Residents can use the core resident workflows for their unit
regardless of whether they are owners, tenants, or family members, except that
tenant access may also require an active Lease Agreement after lease-gated access
is enabled for that Condo.

Resident imports create Unit Residents but do not create Lease Agreements in v1.
Existing imported tenants keep the active Unit Resident access behavior until
staff backfills leases and enables the lease workflow for that Condo.

When a Resident has access to multiple Condos or Units, the resident app asks them to choose the Condo and Unit context before starting unit-scoped workflows.

## Unit

A private condo unit that can have one or more active Residents.

Units belong to a Building and have an explicit floor value. Floors are configured as building layout data, not inferred from unit numbers.

Condo setup captures the Condo profile first: name, address, province, and postal code. Floor and Unit baseline data belongs to the room-layout screen, not the Condo profile itself.

## Building

A physical building inside a Condo.

Each Condo defines its own Buildings, floors per Building, and Units per floor.

For dorms or accommodation businesses, Building is the canonical entity for a
physical tower/building inside one property or branch.

## Parcel

A delivery item received for a Unit.

Parcels are primarily unit-level records. Recipient names or phone numbers may be captured as descriptive details, but active Residents of the Unit can see the Parcel workflow.

Parcel photos are optional supporting evidence captured from the web app. A Parcel can be recorded without a photo.

Parcel pickup is confirmed by staff. The person picking up does not need to have a resident account or LINE binding, and staff may record a pickup name, note, or optional photo as supporting details.

Parcel visibility and Parcel notification recipients may differ. Active Residents of a Unit can see the Unit's Parcels, while LINE notifications for new Parcels follow the Condo's configured resident-role notification setting.

The default Parcel LINE notification setting is owner and tenant. Other supported settings are all active residents, owner only, tenant only, and none.

## Maintenance Request

A v1.1 request reported by an active Unit Resident or staff member and linked to
the reporting Unit when the work is unit-related.

Maintenance Requests may be categorized as unit-related or common-area-related,
but unit-related requests keep the reporting Unit so staff know which room
reported the issue.

Maintenance Requests use `request_type` to distinguish maintenance work from
cleaning work. v1.1 supports `maintenance` and `cleaning`.

Repeated reports of the same issue are recorded as separate Maintenance Requests.

Maintenance Request statuses are submitted, acknowledged, assigned, accepted, in
progress, resolved, closed, rejected, and cancelled.

In v1.1, an Admin, Manager, or authorized Juristic Staff member assigns the work
before a Technician or Housekeeping Staff member can accept it. Category-based
self-claim queues are a future enhancement.

`resolved` means the assigned worker has finished the job. `closed` means an
authorized staff member has reviewed and closed the request.

Condo Admins can configure whether resolution evidence photos are required for
that Condo. When enabled, resolving a Maintenance Request requires at least one
photo. When disabled, evidence photos are optional.

Active Residents of the same Unit can see the Unit's Maintenance Requests and resident-visible status history.

Routine Maintenance Request status changes are shown in the resident app and are not pushed to LINE by default.

Staff may explicitly choose to notify residents by LINE for important Maintenance Request updates, such as appointment coordination, requests for more information, or completion that needs resident review.

## Import Batch

A reviewed upload of condo setup or resident data.

Import Batches produce a diff before changes are applied. Added and changed rows can be applied separately from missing rows.

## Import Scope

The declared coverage of an Import Batch.

Import Scope explains whether an uploaded file represents the whole Condo, a specific Building or floor range, or a partial update. Missing rows are interpreted only within the declared Import Scope.

## Missing Import Row

An existing active record that is not present in an Import Batch's declared scope.

Missing Import Rows are warnings or proposed changes, not automatic deactivations. Deactivation requires explicit confirmation and is recorded as a reversible inactive state rather than a hard delete.

In v1, imports use CSV files. The system provides CSV templates that admins can edit externally and upload back into the system.

Unit layout and resident imports use separate CSV templates in v1.

The unit layout template is the import baseline for the room-layout screen only. Condo name, address, province, and postal code are required Condo setup fields outside the layout template.

Admins can also add or edit units and residents directly in the web app. CSV import is a bulk workflow, not the only way to maintain data.

Single-record edits in the web app validate and save directly with audit history. Actions that deactivate access require explicit confirmation and show the resident access impact.

When an import proposes changing data that was edited directly in the web app after the relevant previous import, the import has a stale CSV conflict. Stale CSV conflicts are blocked unless an admin explicitly overrides them.
