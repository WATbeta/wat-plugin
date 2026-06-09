---
name: book-room
description: Book a WAT meeting room for a time range. Use when the user wants to reserve, book, or schedule a room (e.g. "book Room A tomorrow 10–11 for a sync"). Wraps `wat bookings create`; surfaces the 409 slot-conflict and daily-budget errors cleanly.
---

# Book Room

Create a booking for a WAT meeting room over a start/end time range. Use when the user wants to reserve a room.

## Prerequisite

The `wat` CLI (`@wat-toolbox/wat`) must be installed and the user logged in:

```bash
npm install -g @wat-toolbox/wat   # if not already installed
wat login                 # one-time; mints + stores an API key
```

The skill inherits the CLI's stored API key — never handle secrets here. If `wat` is not found or no key is stored, tell the user to run the two commands above and stop.

## Targeting (dev vs prod)

The CLI defaults to production. Add `--env dev` only when the user explicitly asks for the dev/staging deployment (development data). Do not book on production "to test" — use `--env dev` for that.

## Inputs to resolve first

- **Room** — name or id. If the name is fuzzy or might be ambiguous, run `list-rooms` first.
- **Start / end** — bare datetimes like `2026-06-09T10:00` are read as **Europe/Brussels** wall-clock. Use `--tz <IANA>` to read them in another zone, or include an explicit offset to pin the instant. Minimum booking length is 15 minutes; bookings must be in the future and within the booking horizon (30 days).
- **Title** (optional) — only the booker and admins ever see the title; other members see only the booker's name.

If anything is ambiguous (which room, which day, how long), confirm with the user before booking — a booking is a real side effect.

## Steps

1. Run, always with `--json`:

   ```bash
   wat bookings create --room "<room name or id>" --start "<when>" --end "<when>" --title "<title>" --json
   ```

   `--title` is optional. Add `--tz "<IANA>"` to change the zone bare times are read in; `--env dev` to target staging.

2. Parse the JSON envelope:
   - Success: `{ "success": true, "data": { "booking": { "id" }, "room": { "id", "name" }, "tz", "startsAt", "endsAt" } }`.
   - Failure: `{ "success": false, "status"?, "error": { "code", "message", "nextAction"? } }`.

3. Handle the expected failures clearly, do not retry blindly:
   - **Slot conflict (409, `conflict`)** — the room was just taken for an overlapping range. Tell the user the slot is no longer free and offer to run `check-availability` for nearby open ranges. Do not silently retry.
   - **Daily budget** — each member has a daily booking budget (default 2 hours per Brussels day). If the booking would exceed it, the API returns a budget message; relay it and suggest a shorter range or a different day.
   - `INVALID_TIME` — the start/end couldn't be parsed; re-confirm the datetimes.
   - `ROOM_NOT_FOUND` / `AMBIGUOUS_ROOM` — run `list-rooms` and re-run with an exact name or id.
   - `unauthenticated` — the user needs to run `wat login`.

## Output format

On success: a one-line confirmation with the room name, the local start → end, and the booking `id` (the user needs the id to cancel). On failure: relay `error.message` plainly and offer the most useful next step (check availability, pick a shorter slot, log in).
