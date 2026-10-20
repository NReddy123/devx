# Slot — build prompt

Status: sendable. Implement one phase per session.

Familiarity anchor (not a spec, not a brand to copy): a public booking page for one host. Keep "offer times that stay true in both timezones and do not double-book." Do not copy names, icons, or page layouts from any shipping scheduler.

## What this prompt is teaching

The hard part is the clock, not the page. If you design screens first, you will ship a calendar that lies at DST and races two guests onto one time. Freeze the rules, then build the smallest page that exercises them.

## Product

Slot is a local web app where one host defines event types and weekly hours, and a guest books a free start time. Times are stored as UTC instants. Each side sees them in a named timezone.

v1 ships when a guest can book a slot, the host sees that interval as busy, a second guest cannot book an overlapping start, and a daylight-saving day does not offer a time that does not exist.

v1 is a failure if it can take payment or sync a calendar provider, but can double-book or display the wrong instant after reload.

## v1 user and job

- Primary user for the core loop: a guest booking one meeting.
- Second user: the host, who must trust the agenda and edit hours. The guest loop is the one that defines done.
- Situation: host and guest may be in different timezones. They will not both be looking at the host's wall clock.
- Done looks like: "I picked a time, I got a confirmation with both zone labels, and nobody else got that time."
- Not the user: a team scheduling across pooled calendars, or a clinic with resources and rooms.

## Core loop

1. Guest opens an event type link (`/e/{slug}`).
2. Slot shows the next bookable dates in the guest's selected timezone.
3. Guest picks a start, enters name and email, and confirms.
4. The confirmation page shows the same instant in the guest zone and the host zone.
5. The host agenda shows the booking as busy. A second confirm on that start fails.

Time box: under two minutes for a guest who already knows the day.

## Non-goals

- No payments, no coupons, no packages. Reason: different product.
- No Google, Outlook, or CalDAV sync. Reason: different product. The schema may include `external_event_id` as a nullable unused column so a later sync does not rewrite bookings. Do not call any external API.
- No accounts for guests, no passwords, no email delivery. Reason: quality trap. Email is a string the host can read. Confirmation is an on-screen page plus a link `/b/{token}`. Do not send mail. Do not pretend you sent mail.
- No seats, party size, or resources. Reason: different product. A booking occupies the host, fully, for the interval plus buffers.
- No recurring guest bookings. Reason: different product. Weekly rules are host availability, not a guest recurrence.
- No waitlist. Reason: different product. If it is taken, it is taken.
- No multi-host routing. Reason: different user.

## Rejected alternatives

- Storing local datetimes without a zone and a UTC instant. Killed. It is how DST bugs become unfixable. Store `start_utc` and both IANA zone names.
- Generating slots in the browser from a weekly rule the client interprets alone. Killed. The server generates slots and is the only writer of bookings. The client displays what the server returns.
- Moment.js. Killed. Use `luxon` for zone math. Do not hand-roll offset tables.
- A SaaS deploy with auth. Killed for v1. Local server, host is whoever can open `/host`. There is no login. Binding is localhost only, so "no login" is acceptable. Do not "fix" that by binding all interfaces.
- Optimistic booking without a transaction. Killed. Two confirms must serialize. The unique constraint is part of the product.

## Frozen decisions

- Time storage: `start_utc` is an ISO 8601 UTC string. Also store `guest_zone` and `host_zone` as IANA names (`America/New_York`, not `EST`).
- Display: always render with the zone abbreviation Luxon gives for that instant, plus the offset, plus the IANA name in a secondary line. Example shape: `Tue 3 Mar 2026, 09:00 EST (UTC−5)` and under it `America/New_York`. Do not invent abbreviations.
- Slot grid: starts are every `event.slot_minutes`, aligned to the host zone's local clock, inside available intervals, after applying buffers. Guest view converts those instants for display. Do not align the grid to the guest's local hour.
- A booking occupies `[start, start + duration + buffer_after)`. The previous booking's `buffer_after` and this event's `buffer_before` both block. Buffers from different event types still block the host. The host is one calendar.
- Weekly rule: for each weekday, zero or more ranges `HH:MM–HH:MM` in the host zone, half-open, 15-minute precision. Default for a new host: Mon–Fri 09:00–17:00, host zone `UTC` until they change it.
- Date override: a specific local date in the host zone replaces that weekday's ranges. Empty ranges means unavailable. Overrides do not repeat yearly.
- Booking horizon: guests may book from 2 hours after the server's current time through 21 days ahead. Starts inside the 2-hour lead time are not offered. Do not make this configurable in v1.
- Event type fields are frozen: `slug`, `title`, `duration_minutes` (15, 30, or 60), `slot_minutes` (must equal duration in v1; do not offer a different grid), `buffer_before_minutes`, `buffer_after_minutes` (0, 5, 10, or 15).
- Cancellation: guest or host can cancel with the token. Status becomes `cancelled`. The interval is free immediately. Cancelled rows are kept. A new booking may use that start. There is no reschedule flow; cancel and book again.
- Guest email is required, max 120, must contain one `@` and one dot after it. This is a format check, not verification. Name required, 1 to 80 characters.
- Token: 128-bit random, hex, unguessable. It is the only auth for viewing or cancelling that booking.

## Data model

### Host settings

Single row.

| Field | Type | Notes |
| --- | --- | --- |
| host_zone | string | IANA. Default `UTC`. |
| weekly_json | string | Map of `1`–`7` (ISO weekday) to `[start, end)` pairs of `HH:MM`. |

### Date override

| Field | Type | Notes |
| --- | --- | --- |
| local_date | string | `YYYY-MM-DD` in the host zone. Primary key. |
| ranges_json | string | Same pair format. `[]` means day off. |

### Event type

| Field | Type | Notes |
| --- | --- | --- |
| id | integer | Internal. |
| slug | string | Unique, `[a-z0-9-]{1,40}`. User-visible in the URL. |
| title | string | |
| duration_minutes | integer | 15, 30, or 60. |
| buffer_before_minutes | integer | |
| buffer_after_minutes | integer | |
| active | bool | Inactive types 404 on the public page and do not accept bookings. Existing bookings remain. |

### Booking

| Field | Type | Notes |
| --- | --- | --- |
| id | integer | Internal. |
| event_type_id | integer | |
| start_utc | string | |
| guest_zone | string | IANA, chosen on the page. |
| host_zone | string | Copied from host settings at book time. Do not rewrite old rows if the host later changes zone. |
| guest_name | string | |
| guest_email | string | |
| status | `confirmed` or `cancelled` | |
| token | string | Unique. |
| external_event_id | string or null | Always null in v1. |
| created_at | string | ISO UTC. |

Unique constraint: one row with `status = confirmed` may exist for a given `start_utc` only if intervals cannot overlap. SQLite cannot express interval overlap as a simple unique key. Enforce overlap in a transaction: begin immediate, select confirmed bookings whose occupied interval overlaps the candidate occupied interval, insert only if none, commit. Also uniquely constrain `token`.

Derived, do not store: the list of offered starts. Recompute on each page load.

Occupied interval for a confirmed booking: from `start_utc - buffer_before of that booking's event type` to `start_utc + duration + buffer_after`. Use the event type's current buffers. Document this in `DECISIONS.md` as a known limitation: editing buffers later changes what existing bookings block. v1 accepts that. Do not snapshot buffers onto the booking unless you want a new prompt.

## Screens

1. **Guest event page** `/e/{slug}`. Title, duration, zone select (default: browser zone if it is a valid IANA zone, else host zone). A date list for days that have at least one start, then starts for the selected day. Form: name, email, Confirm. Empty: "No open times in the next 21 days." Error: "Could not load times." Inactive or unknown slug: 404 plain text "No such event."
2. **Confirmation** `/b/{token}`. Both zone renderings, event title, status, Cancel button if confirmed. Unknown token: 404 "No such booking."
3. **Host** `/host`. Zone select. Weekly ranges editor. Override editor (pick a date, set ranges or mark day off). Event type list and a form to create one. Agenda: confirmed bookings in the next 21 days, host zone, with guest name. Loading and error banners as on the guest page.

Not screens: login, payments, email templates, team, analytics, embed builder.

## Interaction spec

### Confirm booking

- Success: server rechecks the start is still offered (lead time, horizon, weekly rule or override, no overlap). Inserts `confirmed`, returns the token, browser navigates to `/b/{token}`. The page shows both zones for the same `start_utc`.
- Cancel: leaving the form before confirm writes nothing. Cancel on the confirmation page sets `cancelled` and shows "Cancelled. This time is free." A second cancel on an already cancelled booking shows that same sentence and does not error.
- Conflict: start no longer free, or failed the recheck. Do not insert. Response 409, UI message: "That time was just taken. Pick another." Refresh the offered starts.

### Edit weekly hours

- Success: save replaces `weekly_json`. Future offer calculations use it. Existing confirmed bookings are not moved or cancelled.
- Cancel: a Cancel control on the editor reverts the form to the last loaded JSON. No request.
- Conflict: a range where end ≤ start, or minutes not divisible by 15, or overlapping ranges on the same day. Refuse save. "Ranges must be 15-minute steps, end after start, and must not overlap."

## Edge cases

| Case | Required behavior |
| --- | --- |
| DST spring-forward gap in the host zone | Do not offer a start whose local time does not exist. Luxon invalid local times are skipped, not shifted. |
| DST fall-back repeated hour | Offer a start only if you can name the instant unambiguously. Use the offset Luxon returns for that zone at the earlier occurrence only if you also store UTC. Do not offer two starts that map to the same `start_utc`. Do not offer a local label you cannot round-trip to the same UTC. Test with `America/New_York` on the 2026-03-08 and 2026-11-01 transitions (verify the actual transition dates with Luxon rather than hardcoding a wrong date; if the date is wrong in this sentence, the library wins, and record the real dates in `DECISIONS.md`). |
| Guest zone differs from host zone | The start list labels use the guest zone. The stored `start_utc` is identical to what the host agenda shows. |
| Two confirms at once for the same start | One 200, one 409. Exactly one confirmed row. |
| Booking that overlaps via buffer only | Not offered, and confirm returns 409 if forced. |
| Lead time | A start 90 minutes from now is not in the list, even if it sits in weekly hours. |
| Host changes zone after a booking | Old booking keeps its stored `host_zone` display. New offers use the new zone. |
| Inactive event | Public page 404. Confirm endpoint rejects. Host still sees old bookings. |

## Visual constraints

- 16px system UI font on the guest page, 14px on the host page. 8px scale. One accent for the selected start and the confirm button.
- The guest page is a single column, max 480px, because it is a form. The host page is a dense stack, not a marketing layout. No hero image, no testimonials, no logo wall.
- Icons: Lucide only, optional, not required.
- Show the zone on every time you render. A time without a zone is a bug.
- No gradients, no illustrations.

## Technical constraints

- Stack: one npm repo. Hono, Node, TypeScript, `better-sqlite3`, React, Vite. Package manager: npm. Time library: `luxon` only. Do not add moment, dayjs, or date-fns.
- Bind `127.0.0.1` port 4188. No auth. No public bind.
- All offer and confirm logic on the server. The client must not decide that a slot is free.
- Tests: node test runner is allowed and expected for the DST and overlap rules. UI checks can be manual. Do not add a browser test framework.
- Seed one event type on first boot if none exist: slug `intro`, title `Intro`, 30 minutes, buffers 0 and 0, active. Seed weekly hours Mon–Fri 09:00–17:00. Do not seed bookings.
- Record exact versions in the project README. Do not upgrade mid-phase.

## Phases and acceptance checks

### Phase 1 — Rules, no UI polish required

In scope: schema, seed, server functions for "list starts" and "confirm", unit tests for overlap, buffer, lead time, and DST. A minimal JSON endpoint is enough if the page is not ready, but phase 1 is not done on tests alone if there is no way to call confirm. Ship `POST /api/bookings` and `GET /api/events/intro/starts?guest_zone=`.

Checks:

1. Automated: two overlapping confirms, one success, one rejection, one row.
2. Automated: buffer overlap is a rejection even when durations do not overlap.
3. Automated: a start in the DST gap is absent from the list. A duplicated local hour does not produce two identical `start_utc` values.
4. Automated: a start inside the 2-hour lead is absent.
5. `external_event_id` exists and is null. No outbound network calls exist in the server code.

### Phase 2 — Guest page and confirmation

In scope: `/e/intro`, zone select, confirm form, `/b/{token}`, cancel.

Checks:

1. Book a start in `America/Los_Angeles` while host zone is `UTC`. Confirmation shows both zone lines and the same instant (the UTC clock time matches the Los Angeles line converted).
2. Second guest (a private window is enough) does not see that start. Forcing the POST returns the taken message.
3. Cancel, then the start is offered again.
4. Unknown slug shows "No such event."

### Phase 3 — Host editor and agenda

In scope: `/host` zone, weekly editor, one date override, create event type, agenda.

Checks:

1. Set a date override to day off. That date disappears from the guest list. An existing booking on that date remains on the agenda.
2. Invalid range is refused with the message in the interaction spec, and the stored JSON is unchanged.
3. New event type `deep-dive` at 60 minutes has its own URL and does not change `intro` bookings.
4. Changing host zone does not rewrite `host_zone` on old bookings.

## Agent operating rules

You are implementing a frozen spec.

- Read this file before writing code. If a chat message conflicts with this file, stop and ask which wins. Do not silently follow the chat.
- Implement the named phase only.
- Do not add features, screens, settings, dependencies, or abstractions this file does not require.
- If you want to add something, write it in `OUT-OF-SCOPE.md` and do not build it.
- If the spec is silent, choose the simplest behavior that preserves the data model and record it in `DECISIONS.md`. Do not invent a product decision.
- If the spec is contradictory, stop on that point only.
- A phase is done only when its checks pass, including empty, error, and cancel paths.
- No unwired placeholder UI. No fake "email sent" copy.
- Do not copy branding, assets, or text from any commercial scheduler.
- Do not add payments, email sending, OAuth, or a public bind.

## Open questions

None. If you find one, stop and report it. Do not guess a product answer.
