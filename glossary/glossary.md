# Glossary

| Term | Definition |
|------|------------|
| Allowed actions | List the API returns with each item naming what the current member may do with it, for example `book`, `edit`, `message-booker`. The frontend shows only these actions. |
| Bookable resource | Something members can book by date, currently the guest apartment and the pottery studio. New rooms such as a music studio are added as resources, not as new booking code. |
| Inner circle | Members with the role that allows booking. Proposed role name; to be confirmed. |
| Member | A person invited to Tulka and able to log in. |
| Group | A set of members created by a member, used to share items with only those people. |
| Platform module | Backend module shared by all sections: identity, groups, visibility, calendar, media, messaging, sections. |
| Section | One area of the app, such as booking or recipes, with its own key, frontend feature and backend module. See ADR-004. |
| Section key | Lowercase identifier of a section (`booking`), used in folder names, API paths and table prefixes. Uppercase form is the documentation domain (`BOOKING`). |
| Visibility | Who may see an item: Only me, selected groups, or Public (all members). Enforced by the backend. |
