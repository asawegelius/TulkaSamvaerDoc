# Booking API contract

Status: draft. Owner: Åsa Wegelius (project owner). Requirements: [REQ-BOOKING-001](../requirements/REQ-BOOKING-001-book-free-dates.md), [REQ-BOOKING-002](../requirements/REQ-BOOKING-002-show-who-has-taken-dates.md).

## Endpoint

`GET /api/v1/booking/resources/{resourceId}/availability?from=YYYY-MM-DD&to=YYYY-MM-DD`

### Purpose

Show each day in the range as free or taken, and who holds a taken day.

### Authentication and Authorization

Any logged-in member.

### Response

Success: 200

```json
{
  "resourceId": "6b0c2d4e-...",
  "days": [
    { "date": "2026-12-23", "status": "FREE", "allowedActions": ["book"] },
    { "date": "2026-12-24", "status": "TAKEN",
      "bookedBy": { "id": "91fa...", "name": "Example Member" },
      "bookingId": "c3e1...",
      "allowedActions": ["message-booker"] }
  ]
}
```

`book` appears only for members allowed to book. A member's own booking lists `cancel` instead of `message-booker`.

### Errors

| Status | Code | Meaning |
|:-------|:-----|:--------|
| 400 | booking.invalid-range | `from` after `to`, or range too long (limit to be agreed) |
| 401 | auth.not-logged-in | Not logged in |
| 404 | booking.resource-not-found | Unknown resource |

## Endpoint

`POST /api/v1/booking/bookings`

### Purpose

Book free dates directly, without approval.

### Authentication and Authorization

Members with the inner circle role.

### Request

```json
{ "resourceId": "6b0c2d4e-...", "from": "2026-12-27", "to": "2026-12-29", "note": "optional" }
```

Whether `to` is inclusive (nights vs. days) is an open question in REQ-BOOKING-001.

### Response

Success: 201 with the created booking and `allowedActions`.

### Errors

| Status | Code | Meaning |
|:-------|:-----|:--------|
| 400 | booking.invalid-range | Invalid or past dates |
| 403 | booking.not-allowed | Member lacks the inner circle role |
| 409 | booking.date-taken | One or more requested days are already booked; nothing is created |

### Side Effects

- Booking stored; `BookingCreated` published; booking appears in the shared calendar.

### Notes

- Idempotency: not idempotent; a retried request after success returns 409 `booking.date-taken`.

## Other endpoints (to be specified)

- `GET /api/v1/booking/resources`
- `DELETE /api/v1/booking/bookings/{id}` (cancel own booking)
