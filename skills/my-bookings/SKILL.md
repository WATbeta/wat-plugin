---
name: my-bookings
description: List the caller's own WAT room bookings and cancel one by id. Use when the user asks "what have I booked", "show my reservations", or "cancel my booking". Wraps `wat bookings list --mine` and `wat bookings cancel <id>`.
---

# My Bookings

List the caller's own bookings and cancel a booking by id. Use when the user wants to see or cancel their reservations.

## Prerequisite

The `wat` CLI (`@wat-toolbox/wat`) must be installed and the user logged in:

```bash
npm install -g @wat-toolbox/wat   # if not already installed
wat login                 # one-time; mints + stores an API key
```

The skill inherits the CLI's stored API key (which identifies the member) — never handle secrets here. If `wat` is not found or no key is stored, tell the user to run the two commands above and stop.

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
