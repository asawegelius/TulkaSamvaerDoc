# Booking service overview

Status: draft. Owner: Åsa Wegelius (project owner).

## Service Name

Booking (`sections/booking`, API prefix `/api/v1/booking`)

## Purpose

Let members see when the guest apartment and the pottery studio are free, and let the inner circle book free dates directly.

## Responsibilities

- Bookable resources (guest apartment, pottery studio; later possibly the music studio or robot workshop).
- Availability per resource and date range.
- Creating and cancelling bookings without approval, preventing overlaps.
- Showing who holds a taken date.
- Contributing bookings to the shared calendar.

## Non-Responsibilities

- Sending messages: the Message button on a taken date opens a conversation in the messaging module.
- Member and role management: owned by identity.

## Public Interfaces

- REST: see [booking API contract](../api/booking-api-contract.md).
- Event: `BookingCreated`, `BookingCancelled` (for notifications).
- Extension: implements `CalendarContributor`.

## Data Ownership

- `booking_resource`, `booking_booking` (proposed names).

## Dependencies

- identity (current member and role), messaging (conversation with the booker), calendar (contributor registration).

## Security Notes

- Only members with the inner circle role may book (see [REQ-BOOKING-001](../requirements/REQ-BOOKING-001-book-free-dates.md)).
- All members may see availability and who has booked (see [REQ-BOOKING-002](../requirements/REQ-BOOKING-002-show-who-has-taken-dates.md)). Open question in that record.

## Operational Notes

- Overlap prevention must hold under concurrent requests (database constraint or lock), not only in application code.

## Related Documents

- Requirements: REQ-BOOKING-001, REQ-BOOKING-002
- ADRs: ADR-001, ADR-003
