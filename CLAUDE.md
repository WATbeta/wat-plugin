# `wat-plugin`

Single-plugin [Open Plugin v1](https://github.com/vercel-labs/open-plugin-spec) repo for the WAT Rooms skills. The repo root IS the plugin root — no `plugins/` wrapper. Plugin name (namespace): `wat`. Skills surface as `/wat:<skill>`.

## What this is

Four Claude Code skills that let an agent book WAT meeting rooms conversationally. They are **thin wrappers** over the [`wat` CLI](https://github.com/WATbeta/wat-cli) (package `@wat-toolbox/wat`), which itself drives the same public API the WAT Rooms web app uses. The plugin holds **no auth, no secrets, no URLs** — the CLI owns auth (a stored API key from `wat login`) and `--env dev|prod` targeting.

## Skills

| Skill | Wraps |
|---|---|
| `skills/list-rooms` | `wat rooms list` |
| `skills/check-availability` | `wat availability --room --from --to [--tz]` |
| `skills/book-room` | `wat bookings create --room --start --end [--title] [--tz]` |
| `skills/my-bookings` | `wat bookings list --mine`, `wat bookings edit <id>`, `wat bookings cancel <id>` |

## Conventions (carry these when editing skills)

- **No secrets in skills.** They must never contain API keys or internal URLs. Auth and base-URL resolution live entirely in the `wat` CLI.
- **Always `--json`.** Every skill invokes the CLI with `--json` and parses the `{ success, data | error }` envelope. Never scrape the human-readable output.
- **`--env dev` only on request.** The CLI defaults to production. A skill adds `--env dev` only when the user explicitly asks for the staging/dev environment, so casual use can't cross prod/dev data.
- **Brussels time by default.** Bare datetimes are Europe/Brussels wall-clock; `--tz <IANA>` overrides; an explicit offset pins the instant. Mirror the CLI's documented behavior.
- **Side effects need confirmation.** `book-room` and `my-bookings` (cancel) cause real mutations — confirm ambiguous inputs with the user before acting.
- Keep each `SKILL.md` self-contained: it should work without other skills loaded (cross-references like "run list-rooms first" are suggestions, not dependencies).

## Manifests

Two manifests are kept intentionally in sync — update **both** in lockstep when changing metadata:

- `.plugin/plugin.json` — canonical Open Plugin v1 manifest (read by `npx plugins`, Cursor, …).
- `.claude-plugin/plugin.json` — Claude Code vendor-prefixed manifest (preferred by Claude Code when both are present).

## Adding a skill

1. Create `skills/<name>/SKILL.md` with YAML frontmatter (`name`, `description`). The loader auto-discovers it — no manifest edit needed.
2. Document it in `README.md`.

## Versioning

Semver, both manifests in lockstep. Adding a skill = minor; renaming/removing one = major.

## Local testing

```bash
npx plugins add .
```

Skills then surface as `/wat:<skill>`. End-to-end validation needs the `wat` CLI installed and `wat login` run (see README).
