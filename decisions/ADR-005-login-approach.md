---
id: ADR-005
title: "Choose how members log in"
status: draft
owner: "Åsa Wegelius (project owner)"
related: []
---

# ADR-005: Choose how members log in

## Context

Only invited members may use Tulka. The owner has not yet chosen a login method. This record is a proposal; the decision is open.

## Decision

**Open.** Proposed: Spring Security with a session cookie (HttpOnly, SameSite=Lax) and CSRF protection, using Angular's built-in XSRF support. Social login (Google or Apple) can be added later through OAuth2 login on the backend without frontend changes.

## Alternatives considered

- **Username and password with session cookie**: simplest; members need one more password.
- **Google or Apple sign-in**: no password to remember; depends on every member having such an account.
- **JWT bearer tokens in the browser**: rejected for now; token storage in the browser is harder to secure than an HttpOnly cookie.

## Consequences

- Benefits and costs depend on the chosen option.

## Related artifacts

None yet.

## Follow-up work

- Owner chooses the login method. Then update this record and add identity requirements.

## Replacement history

None.
