---
id: REQ-BOOKING-001
title: "Inner circle members book free dates directly"
type: functional
status: draft
owner: "Åsa Wegelius (project owner)"
source: "Project brief from Åsa Wegelius, Claude project chat, 2026-11-24 (summarised): booking of the guest apartment and pottery studio is for the inner circle only, needs no approval, and free dates are booked directly. Drafted by Claude as a proposal."
decisions: [ADR-001]
---

# REQ-BOOKING-001: Inner circle members book free dates directly

## Need and rationale

Family members in the inner circle want to reserve the guest apartment or pottery studio without waiting for anyone to approve. Without this, bookings happen by message and double bookings occur.

## Scope

- Applies to: bookable resources (guest apartment, pottery studio).
- Boundaries: cancelling and changing bookings are separate requirements (not yet written).

## Requirement

A member with the inner circle role can book one or more consecutive free dates of a resource. The booking is confirmed immediately. A booking can never overlap another booking of the same resource.

## Trigger and preconditions

- Trigger: the member submits a booking for a resource and a date range.
- Required starting conditions: the member is logged in.

## Acceptance criteria

- AC-1: Given an inner circle member and a resource with all requested dates free, when the member books them, the booking is confirmed immediately and the dates show as taken by that member.
- AC-2: Given a requested range in which at least one date is already booked, when any member books it, the request is rejected with a "date taken" outcome and no booking is created.
- AC-3: Given a member without the inner circle role, when they try to book, the request is rejected as not allowed and no booking is created.
- AC-4: Given two inner circle members booking overlapping dates of the same resource at the same time, exactly one booking is created.

## Verification plan

Backend integration tests against a real database for AC-1 to AC-4 (AC-4 with concurrent requests). One frontend test that the Book action appears only when the API lists it.

## Related decisions and constraints

[ADR-001](../decisions/ADR-001-one-modular-spring-boot-app.md). API: [booking API contract](../api/booking-api-contract.md).

## Assumptions and open questions

| Assumption or question | Consequence if wrong or unresolved | Human owner | Resolve before |
|:-----------------------|:-----------------------------------|:------------|:---------------|
| Is the guest apartment booked by night (check-in/check-out) and the studio by day? | Changes how overlap is computed and how ranges are entered | Åsa Wegelius | Approval |
| Is there a maximum booking length or limit on future dates? | Members could reserve long periods | Åsa Wegelius | Approval |
| Can past dates be booked? (Assumed no) | Affects validation | Åsa Wegelius | Approval |

## Review record

- Agent review: Drafted by Claude; not yet reviewed separately.
- Human disposition: Pending.
- Human approval: Not approved.
