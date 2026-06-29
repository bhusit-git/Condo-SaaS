# ADR 0002: Use LINE multicast for large announcement notifications

Date: 2026-05-28

## Status

Accepted

## Context

The platform sends condo announcements to recipient sets selected from its own database. Some announcements may target hundreds of residents in one condo.

Individual push gives clearer per-recipient delivery results, but it uses one provider call per recipient and can consume capacity quickly. Multicast is more efficient for large recipient sets, but successful batch acceptance does not prove that every recipient received the message.

## Decision

Use LINE reply messages whenever an active resident interaction can be answered directly.

Use individual push for parcels, explicit maintenance notifications once maintenance ships, and small announcement recipient sets.

Use LINE multicast for large normal or broadcast announcement recipient sets
starting in v1.1. Do not use LINE broadcast for SaaS v1.1 because condo targeting
must come from platform recipient snapshots.

For critical announcements, multicast may be attempted first for speed. If the batch fails and the error is retryable, retry according to delivery policy; if still unsuccessful, fall back to individual push when configured for critical delivery. When the platform cannot tell whether LINE accepted a provider request, the delivery is marked `outcome_unknown`; non-critical multicast is not retried automatically, while critical delivery may retry or fallback because missed emergency delivery is worse than rare duplicates.

v1.1 persists LINE notification jobs and deliveries in Postgres-backed queue
tables and processes them with a scheduled Edge Function worker. The
implementation contract for job states, delivery states, idempotency keys, quota
counters, retry classification, and manual resend is defined in
[v1 Implementation Contract](../v1-implementation-contract.md).

## Consequences

Large announcements use far fewer provider calls.

Per-recipient delivery logs must distinguish multicast batch acceptance from individual delivery confirmation.

Notification workers must group recipients by LINE channel, condo context, payload, and locale before dispatch.

Notification operators must be able to see and manually resend non-critical multicast deliveries whose provider outcome is unknown.
