---
id: REQ-BOOKING-002
title: "Taken dates show who booked them, with a way to message them"
type: functional
status: draft
owner: "Åsa Wegelius (project owner)"
source: "Project brief from Åsa Wegelius, Claude project chat, 2026-11-24 (summarised): taken dates show who has them, with a Message button. Drafted by Claude as a proposal."
decisions: []
---

# REQ-BOOKING-002: Taken dates show who booked them, with a way to message them

## Need and rationale

When a date is taken, a member who wants it can ask the booker to swap or share. Showing the booker and a direct way to message them makes that easy.

## Scope

- Applies to: availability views of bookable resources.
- Boundaries: the conversation itself belongs to the chat section.

## Requirement

For each taken date, the availability view shows the member who booked it and offers a Message action that opens a conversation with that member.

## Trigger and preconditions

- Trigger: a member views availability of a resource.
- Required starting conditions: the member is logged in.

## Acceptance criteria

- AC-1: Given a date booked by member A, when member B views availability, the date shows as taken by member A.
- AC-2: Given a date booked by member A, when member B chooses Message on that date, a conversation between B and A opens.
- AC-3: Given a date booked by the viewing member themself, the Message action is not offered.

## Verification plan

Backend integration test for AC-1 and the `allowedActions` behind AC-2 and AC-3; frontend test that the Message action follows `allowedActions`.

## Related decisions and constraints

None yet. API: [booking API contract](../api/booking-api-contract.md).

## Assumptions and open questions

| Assumption or question | Consequence if wrong or unresolved | Human owner | Resolve before |
|:-----------------------|:-----------------------------------|:------------|:---------------|
| Can members outside the inner circle see who booked? (Assumed yes, all members) | Privacy of the booker | Åsa Wegelius | Approval |
| Does Message need the chat section first, or is a simple notification enough for the first version? | Build order | Åsa Wegelius | Planning booking work |

## Review record

- Agent review: Drafted by Claude; not yet reviewed separately.
- Human disposition: Pending.
- Human approval: Not approved.
