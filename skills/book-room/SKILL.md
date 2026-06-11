---
name: book-room
description: Book a WAT meeting room for a time range. Use when the user wants to reserve, book, or schedule a room (e.g. "book Room A tomorrow 10–11 for a sync"). Wraps `wat bookings create`; surfaces the slot-overlap, daily-budget, and opening-hours errors cleanly.
---

# Book Room

Create a booking for a WAT meeting room over a start/end time range. Use when the user wants to reserve a room.

## Prerequisite

The `wat` CLI (`@wat-toolbox/wat`) must be installed and the user logged in:

```bash
npm install -g @wat-toolbox/wat   # if not already installed
wat login                 # one-time; mints + stores an API key
```

The skill inherits the CLI's stored API key — never handle secrets here. If `wat` is not on the PATH, fall back to `npx -y @wat-toolbox/wat <command>` (same tool, same stored login) and suggest the global install once the task is done. If no key is stored (`unauthenticated` / "Not logged in"), run the login handoff yourself instead of stopping:

1. Ask the user for their WAT account email.
2. `wat login --email <email> --request-code --json` — sends a 6-digit code to their inbox.
3. Ask the user for the code, then `wat login --email <email> --code <code> --json`. A `NOT_AUTHORIZED` error means the email may not enroll: it is not on WAT's resident list. Tell the user to ask a WAT admin to add their email or company domain (web app, Admin → Access), then retry from step 2 — never retry blindly.

## Targeting (dev vs prod)

The CLI defaults to production. Add `--env dev` only when the user explicitly asks for the dev/staging deployment (development data). Do not book on production "to test" — use `--env dev` for that.

## Inputs to resolve first

- **Room** — name or id. If the name is fuzzy or might be ambiguous, run `list-rooms` first.
- **Start / end** — bare datetimes like `2026-06-09T10:00` are read as **Europe/Brussels** wall-clock. Use `--tz <IANA>` to read them in another zone, or include an explicit offset to pin the instant. Booking rules: minimum length 15 minutes; must be in the future; within the booking horizon (30 days); AND entirely within opening hours **06:00–22:00 Europe/Brussels** (a booking ending exactly at 22:00 is fine).
- **Title** (optional) — only the booker and admins ever see the title; other members see only the booker's name.
- **Guests** (optional) — up to 10 invitee email addresses, ANY address (guests need no WAT account). Guests receive an email invitation with a calendar invite, the pre-start reminder, and update/cancellation emails. Only the booker and admins see the guest list.

If anything is ambiguous (which room, which day, how long, who to invite), confirm with the user before booking — a booking is a real side effect, and inviting guests sends them real emails.

## Steps

1. Run, always with `--json`:

   ```bash
   wat bookings create --room "<room name or id>" --start "<when>" --end "<when>" --title "<title>" --guests "<a@x.com,b@y.be>" --json
   ```

   `--title` and `--guests` are optional (`--guests` is comma-separated, max 10; requires CLI v0.5.0+). Add `--tz "<IANA>"` to change the zone bare times are read in; `--env dev` to target staging.

2. Parse the JSON envelope:
   - Success: `{ "success": true, "data": { "booking": { "id" }, "room": { "id", "name" }, "tz", "startsAt", "endsAt", "guests"? } }`.
   - Failure: `{ "success": false, "status"?, "error": { "code", "message", "nextAction"? } }`.

3. Handle the expected failures clearly, do not retry blindly:
   - **Slot taken (409, `overlap`)** — the room was just taken for an overlapping range. Tell the user the slot is no longer free and offer to run `check-availability` for nearby open ranges. Do not silently retry.
   - **Daily budget (409, `budget`)** — each member has a daily booking budget (default 2 hours per Brussels calendar day). If the booking would exceed it, relay the message and suggest a shorter range or a different day.
   - **Outside opening hours (400, `outside_hours`)** — the range falls outside 06:00–22:00 Europe/Brussels. Propose the nearest in-hours slot instead.
   - **Bad guest list (422, `invalid_guests` / `too_many_guests`)** — a guest address is malformed (the message names it) or the list exceeds 10. Fix the flagged address / trim the list with the user and retry.
   - `INVALID_TIME` — the start/end couldn't be parsed; re-confirm the datetimes.
   - `ROOM_NOT_FOUND` / `AMBIGUOUS_ROOM` — run `list-rooms` and re-run with an exact name or id.
   - `unauthenticated` — the user needs to run `wat login`.

## Changing an existing booking

To reschedule a booking, do not cancel and rebook — the `wat` CLI (v0.2.0+) has `wat bookings edit <id> [--room --start --end --title]` (PATCH-backed, same validation including opening hours). See the `my-bookings` skill for rescheduling guidance.

## Output format

On success: a one-line confirmation with the room name, the local start → end, and the booking `id` (the user needs the id to cancel or edit). On failure: relay `error.message` plainly and offer the most useful next step (check availability, pick a shorter slot, log in).
