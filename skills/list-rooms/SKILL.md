---
name: list-rooms
description: List the WAT meeting rooms with their capacity and description. Use when the user asks what rooms exist, which rooms are available to book, or wants a room's id/name/capacity. Thin wrapper over the `wat rooms list` CLI command.
---

# List Rooms

List the active WAT meeting rooms, showing each room's name, capacity, and description. Use this when the user asks "what rooms are there", "which rooms can I book", or needs a room name or id before checking availability or booking.

## Prerequisite

The `wat` CLI (`@wat-toolbox/wat`) must be installed and the user must be logged in:

```bash
npm install -g @wat-toolbox/wat   # if not already installed
wat login                 # one-time; mints + stores an API key
```

The skill inherits the CLI's stored API key — you never handle secrets here. If `wat` is not on the PATH, fall back to `npx -y @wat-toolbox/wat <command>` (same tool, same stored login) and suggest the global install once the task is done. If no key is stored (`unauthenticated` / "Not logged in"), run the login handoff yourself instead of stopping:

1. Ask the user for their WAT account email.
2. `wat login --email <email> --request-code --json` — sends a 6-digit code to their inbox.
3. Ask the user for the code, then `wat login --email <email> --code <code> --json`. A `NOT_AUTHORIZED` error means the email may not enroll: it is not on WAT's resident list. An access request was automatically filed with the WAT admins — tell the user to watch for the "you've been accepted" email, then retry from step 2 once it arrives. Never retry blindly: retrying does not speed up the review.

## Targeting (dev vs prod)

The CLI defaults to the production deployment. To target the staging/dev deployment (development data), pass `--env dev`. Only add `--env dev` when the user explicitly asks for the dev/staging environment.

## Steps

1. Run the command, always using `--json` so you can parse the result reliably:

   ```bash
   wat rooms list --json
   ```

   For the dev environment:

   ```bash
   wat --env dev rooms list --json
   ```

2. Parse the JSON envelope:
   - Success: `{ "success": true, "data": { "rooms": [ { "id", "name", "capacity", "description", "color", "image_url" } ] } }`
   - Failure: `{ "success": false, "error": { "code", "message", "nextAction"? } }`

3. On failure, relay `error.message` (and `nextAction` if present) plainly. A `code` of `unauthenticated` means the user needs to run `wat login`.

## Output format

A short plain-language list, one room per line: name, capacity (seats) if present, and description if present. `image_url` is the room's photo — mention or link it only when the user asks what a room looks like. Mention the room `id` only if the user will need it for a follow-up command (e.g. to disambiguate two rooms with the same name). If there are no rooms, say so.
