# `wat`

> Book WAT meeting rooms from Claude Code. Check availability, book a room, list and cancel your bookings — all conversationally.

Agent plugin conforming to the [Open Plugin Specification v1.0](https://github.com/vercel-labs/open-plugin-spec). Installs into any Open-Plugin-compatible host (Claude Code, Cursor, …) with a single command. The skills are thin wrappers over the [`wat` CLI](https://github.com/WATbeta/wat-cli), which drives the same public API the WAT Rooms web app uses.

## Install

```bash
npx plugins add WATbeta/wat-plugin
```

Claude Code native alternative:

```
/plugin marketplace add WATbeta/wat-plugin
/plugin install wat@wat-plugin
```

## Prerequisite — the `wat` CLI

The skills shell out to the `wat` CLI (`@wat-toolbox/wat`). Install it once and log in:

```bash
npm install -g @wat-toolbox/wat
wat login
```

`wat login` runs an email-OTP flow (and, on first enrollment, asks for the WAT signup code), then mints and stores an API key in `~/.config/wat/config.json` (mode 0600). **The plugin inherits this stored key** — you authenticate once via the CLI, and every skill reuses it. No secrets live in the plugin.

> **No shell access?** Hosts without a terminal (claude.ai, Claude Desktop) can use the [WAT Rooms MCP server](https://wat-mcp.vercel.app) instead — same tools, zero installation, OAuth sign-in.

## Environment targeting (`--env`)

The CLI — and therefore the skills — default to the **production** deployment. To target the **staging/dev** deployment (which reads the development data branch), the skills pass `--env dev` to the CLI. The skills only do this when you explicitly ask for the dev/staging environment, so a casual "book Room A" never touches dev data and a casual booking on dev never lands on production.

## What's inside

| Component | Count | Namespace |
|---|---|---|
| Skills | 4 | `/wat:<skill>` |

### Skills

| Skill | What it does | Wraps |
|---|---|---|
| `wat:list-rooms` | List the meeting rooms with capacity + description. | `wat rooms list` |
| `wat:check-availability` | Resolve a room + time window and summarize the free slots. | `wat availability --room --from --to [--tz]` |
| `wat:book-room` | Create a booking; surfaces the slot-overlap, daily-budget, and opening-hours errors cleanly. | `wat bookings create --room --start --end [--title] [--tz]` |
| `wat:my-bookings` | List the caller's own bookings, reschedule one, or cancel one by id. | `wat bookings list --mine`, `wat bookings edit <id>`, `wat bookings cancel <id>` |

Every skill calls the CLI with `--json` and parses the `{ success, data | error }` envelope, so results and errors are handled deterministically.

## Time interpretation

Bare datetimes (e.g. `2026-06-09T10:00`) are read as **Europe/Brussels** wall-clock by the CLI, matching the web UI. Pass `--tz <IANA>` to read bare times in another zone, or include an explicit offset (`2026-06-09T10:00+02:00`) to pin the instant.

Rooms have opening hours of **06:00–22:00 Europe/Brussels**: availability results are clipped to that window, and bookings outside it are rejected (`outside_hours`).

## License

MIT — see `LICENSE`.
