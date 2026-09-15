# Running this fork on Coolify

Two containers (`warcon` + `db`), built by Coolify from this repository. No terminal, no
registry, no release tags.

**Nothing updates on its own.** Upstream cannot push to this fork, so `main` only moves when you
press *Sync fork* on GitHub. Redeploy as often as you like — you get the same code every time
until you deliberately sync. The fork is the update gate.

This fork **adds** two files (`docker-compose.coolify.yml` and this one) and **modifies none**,
so *Sync fork* is always a clean fast-forward and can never hit a conflict.

`WARCON_ROLE=all` serves the panel, runs the observation worker in-process, and applies
migrations on start — the two-container layout warcon.app documents. Upstream's own compose
splits that into `migrate`/`web`/`worker` so a web deploy never interrupts observation, but
Coolify redeploys the whole stack at once, so the split would buy nothing here while costing an
extra secret and an internal relay port.

## Setting it up

Create the resource: **Public Git Repository** → `https://github.com/Armyrat60/warcon`, then:

| Setting | Value |
| --- | --- |
| Build Pack | Docker Compose |
| Docker Compose Location | `/docker-compose.coolify.yml` |
| Branch | `main` |
| Automatic Deployment | **off** |
| Domain | on the `warcon` service only, port 3000 |

Then fill in the environment variables below, deploy, and immediately open `/setup` to create the
owner account using `SETUP_TOKEN`.

## Updating

Two buttons, both in a browser:

1. **Sync fork** on the GitHub repository page. This deploys nothing — it only moves `main`.
2. **Redeploy** in Coolify when you are ready.

Leave days or weeks between them if you want. Upstream's pace never touches the server.

Before syncing, it is worth glancing at what changed — GitHub's compare view shows it without a
terminal. Four paths actually matter:

- **`.env.example`** — a newly required variable. The app refuses to start without `ORIGIN`,
  `BETTER_AUTH_SECRET`, or a database target, so add it in Coolify *before* redeploying.
- **`drizzle/`** — new migrations, applied on start and **one-way**. Take a database backup
  before redeploying when this changed; you cannot roll back to older code against a newer schema.
- **`docker-compose.yml`** — if upstream changes the service layout, mirror it into
  `docker-compose.coolify.yml`, which is a parallel copy rather than an override.
- **`Dockerfile`** — build or runtime changes.

## Environment variables

Set in Coolify, not in the repository.

| Variable | Notes |
| --- | --- |
| `ORIGIN` | The exact public URL, no trailing slash. Cookies are only marked Secure when it starts with `https://`. |
| `BETTER_AUTH_SECRET` | 32+ random characters. Changing it signs everyone out. |
| `ENCRYPTION_KEY` | base64 of 32 random bytes. **Changing or losing it makes every stored RCON password permanently undecryptable.** Back it up separately from the database. |
| `POSTGRES_PASSWORD` | Cannot be rotated by editing this alone — Postgres only reads it at initdb, so the existing volume keeps the old one. Change it inside Postgres first. |
| `SETUP_TOKEN` | Required before the first deploy: `/setup` creates the site owner unauthenticated while the panel has zero users. |
| `ADDRESS_HEADER` | `x-forwarded-for` behind Traefik alone, `cf-connecting-ip` behind a proxied Cloudflare record. Wrong value collapses every client onto one apparent IP, and the 40-failure per-IP lockout then locks out everyone at once. |
| `XFF_DEPTH` | `1` with `x-forwarded-for`; leave blank with `cf-connecting-ip`. |
| `APP_NAME` | Optional. Shown in the UI and as the TOTP issuer. |
| `STEAM_API_KEY` | Optional. Enables persona, account age and VAC/ban lookups. |
| `DISCORD_CLIENT_ID` / `DISCORD_CLIENT_SECRET` | Optional, both or neither. Enables Discord sign-in. |
| `TURNSTILE_SITE_KEY` / `TURNSTILE_SECRET_KEY` | Optional, both or neither. |

Coolify can generate the random values for you: use the dice / generate button on the variable
rather than finding a terminal. `ENCRYPTION_KEY` must be base64 of exactly 32 bytes — if Coolify's
generator gives something else, the panel will say so clearly at startup.

## What a redeploy does not touch

Environment variables and the `warcon-db` volume are properties of the Coolify resource, not of a
deployment. Redeploy, restart and rebuild all preserve them. **Deleting the Coolify resource
deletes the volume.**

Enable Coolify's scheduled backups on the `db` service: the panel holds the entire ban history and
audit trail, and there is no export-everything button.

## Building needs memory

Coolify builds from source on the server (`bun install` + `vite build`). Give the host at least
2 GB of RAM or the build can be killed partway through.
