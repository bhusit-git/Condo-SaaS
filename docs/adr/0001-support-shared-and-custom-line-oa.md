# ADR 0001: Support shared and custom LINE Official Accounts

Date: 2026-05-28

## Status

Accepted

## Context

The product needs v1.0 Core Backoffice to launch without depending on LINE, then
needs fast LINE onboarding in v1.1 and a premium branding path for Enterprise
later.

A shared platform LINE Official Account lets new condos start quickly without setting up their own LINE account. A custom condo-specific LINE Official Account gives premium condos their own branding and dedicated channel.

LINE identity and channel behavior make silent migration between LINE Official Accounts risky. Residents should explicitly bind through the LINE Official Account they will use.

## Decision

The platform supports both Shared Platform LINE OA and Custom LINE OA.

v1.1 uses the Shared Platform LINE OA by default for Business and Enterprise
Condos with LINE entitlement. Residents enter condo context through
condo-specific LIFF links or QR codes.

Enterprise/future phases add Custom LINE OA as a premium option. When a condo
upgrades, residents re-bind through the custom LINE account. Their existing
Resident and Unit relationships remain unchanged.

The v1.1 schema and routing contract for `line_channel_id`, Shared OA defaults,
and Custom OA schema-ready fields is defined in
[v1 Implementation Contract](../v1-implementation-contract.md).

## Consequences

Condos can launch v1.0 backoffice without LINE, then start quickly on the shared
account in v1.1 and later upsell to branded LINE.

The system must route LINE events and notifications by both condo context and LINE channel mode.

Custom LINE OA upgrade requires resident communication and a re-binding flow instead of silent migration.

v1.0 must not ship LINE runtime. v1.1 must not ship Custom OA admin setup,
migration UI, or full re-bind flow unless a later pilot decision explicitly
reopens scope.
