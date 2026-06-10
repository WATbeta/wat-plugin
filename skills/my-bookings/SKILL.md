---
name: my-bookings
description: List the caller's own WAT room bookings, reschedule one, or cancel one by id. Use when the user asks "what have I booked", "show my reservations", "move my booking", or "cancel my booking". Wraps `wat bookings list --mine`, `wat bookings edit <id>`, and `wat bookings cancel <id>`.
---

# My Bookings

List the caller's own bookings, reschedule one, or cancel one by id. Use when the user wants to see, change, or cancel their reservations.

## Prerequisite

The `wat` CLI (`@wat-toolbox/wat`) must be installed and the user logged in:

```bash
npm install -g @wat-toolbox/wat   # if not already installed
wat login                 # one-time; mints + stores an API key
```

The skill inherits the CLI's stored API key (which identifies the member) — never handle secrets here. If `wat` is not on the PATH, fall back to `npx -y @wat-toolbox/wat <command>` (same tool, same stored login) and suggest the global install once the task is done. If no key is stored (`unauthenticated` / "Not logged in"), run the login handoff yourself instead of stopping:

1. Ask the user for their WAT account email.
2. `wat login --email <email> --request-code --json` — sends a 6-digit code to their inbox.
3. Ask the user for the code, then `wat login --email <email> --code <code> --json`. A `NOT_AUTHORIZED` error means the email may not enroll: it is not on WAT's resident list. Tell the user to ask a WAT admin to add their email or company domain (web app, Admin → Access), then retry from step 2 — never retry blindly.

## Targeting (dev vs prod)

The CLI defaults to production. Add `--env dev` only when the user explicitly asks for the dev/staging deployment (development data).

## Listing the caller's bookings

Always use `--mine` so you only return the caller's own bookings, and `--json`:

```bash
wat bookings list --mine --json
```

Optional filters: `--room "<name|id>"`, `--from "<when>"`, `--to "<when>"`, `--tz "<IANA>"` (zone for bare times + display), `--env dev`.

Parse the envelope:
- Success: `{ "success": true, "data": { "tz", "bookings": [ { "id", "room_id", "title", "starts_at", "ends_at", "status", "is_mine", "is_maintenance", "booker_name" } ] } }`. Times are ISO instants; render them in the returned `tz` (Europe/Brussels by default).
- Failure: `{ "success": false, "error": { "code", "message", "nextAction"? } }`.

## Rescheduling a booking

To move a booking (or change its room/title), prefer editing over cancel + rebook — it keeps the same booking id and atomically swaps the slot. Requires `wat` CLI v0.2.0+:

```bash
wat bookings edit <id> [--room "<name|id>"] [--start "<when>"] [--end "<when>"] [--title "<title>"] --json
```

Pass only the fields to change; get the `id` from the list above (never guess). Bare datetimes are Europe/Brussels wall-clock (`--tz <IANA>` overrides). The new range goes through the same validation as creating: minimum 15 minutes, in the future, within the 30-day horizon, and within opening hours **06:00–22:00 Europe/Brussels**. Failures use the same codes as `book-room`: `overlap` (409, slot taken), `budget` (409, over the 2h/day Brussels cap), `outside_hours` (400). Confirm the target booking and new time with the user before editing — it is a real side effect.

## Cancelling a booking

You need the booking `id` (get it from the list above; never guess). Bookings can be cancelled up until they end.

```bash
wat bookings cancel <id> --json
```

Parse:
- Success: `{ "success": true, "data": { "cancelled": "<id>" } }`.
- Failure (`{ "success": false, ... }`): relay `error.message`. A `permission-denied` (403) means it isn't the caller's booking; a message about the booking already being over means it can no longer be cancelled.

Before cancelling, confirm with the user which booking (echo the room + time), since cancellation is a real side effect — unless the user already gave an unambiguous id.

## Output format

For a list: a short plain-language list, one booking per line — room (or title if the caller owns it), local start → end, status, and the `id` (needed to cancel). If empty, say there are no bookings. For a cancel: a one-line confirmation naming the cancelled booking.
