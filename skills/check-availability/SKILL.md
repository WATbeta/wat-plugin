---
name: check-availability
description: Check when a WAT meeting room is free over a time window and summarize the open slots. Use when the user asks "is Room A free tomorrow afternoon", "when can I book the boardroom", or wants free time ranges for a room before booking. Wraps the `wat availability` CLI command.
---

# Check Availability

Resolve a room plus a time window and report the room's free time ranges. Use this when the user wants to know when a room is open before booking it.

## Prerequisite

The `wat` CLI (`@wat/wat`) must be installed and the user logged in:

```bash
npm install -g @wat/wat   # if not already installed
wat login                 # one-time; mints + stores an API key
```

The skill inherits the CLI's stored API key — never handle secrets here. If `wat` is not found or no key is stored, tell the user to run the two commands above and stop.

## Targeting (dev vs prod)

The CLI defaults to production. Add `--env dev` only when the user explicitly asks for the dev/staging deployment (development data).

## Inputs to resolve first

- **Room** — a room name or id. If the user gave a fuzzy name and you are unsure it matches, run the `list-rooms` skill first to get exact names/ids.
- **Window** — a `--from` and a `--to`. Bare datetimes like `2026-06-09T10:00` (or a bare date `2026-06-09`) are interpreted as **Europe/Brussels** wall-clock by the CLI. Pass `--tz <IANA>` to interpret bare times in another zone, or include an explicit offset (`2026-06-09T10:00+02:00`) to pin the instant. Convert vague phrasing ("tomorrow afternoon") into concrete `--from`/`--to` values, confirming the date if ambiguous.

## Steps

1. Run, always with `--json`:

   ```bash
   wat availability --room "<room name or id>" --from "<when>" --to "<when>" --json
   ```

   Optional: `--tz "<IANA>"` to change the zone bare times are read in; `--env dev` to target staging.

2. Parse the JSON envelope:
   - Success: `{ "success": true, "data": { "room": { "id", "name" }, "tz", "free": [ { "start", "end" } ], "busy": ... } }` — `free`/`busy` carry ISO instants.
   - Failure: `{ "success": false, "error": { "code", "message", "nextAction"? } }`. Common codes: `ROOM_NOT_FOUND`, `AMBIGUOUS_ROOM` (pass the room id instead), `INVALID_TIME`, `unauthenticated`.

3. On `AMBIGUOUS_ROOM`, run `list-rooms` to show the candidate ids, then re-run with the id.

## Output format

Plain language. State the room and the window's timezone, then list the free ranges as readable local times (e.g. "Mon 09 Jun, 10:00 → 12:00"). If `free` is empty, say the room is fully booked over that window. Offer to book one of the free ranges (hand off to the `book-room` skill).
