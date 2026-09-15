# Running this fork on Coolify

This fork exists to make upstream updates deliberate, not to change Warcon. It **adds** two files
(`docker-compose.coolify.yml` and this one) and **modifies none**, so every upstream merge is a
fast-forward and never conflicts.

Coolify resource settings:

| Setting | Value |
| --- | --- |
| Build Pack | Docker Compose |
| Repository | `Armyrat60/warcon`, branch `main` |
| Docker Compose Location | `/docker-compose.coolify.yml` |
| Automatic Deployment | **off** |
| Domain | on the `warcon` service only, port 3000 |

Everything else lives in the Environment Variables tab (see the table below), never in a
committed `.env`.

## Keeping up with upstream

Because nothing upstream owns is modified here, GitHub's **Sync fork** button on the repository
page does the whole merge — no local clone needed. It will always offer a fast-forward.

Syncing the fork deploys nothing. Coolify only moves when you press Redeploy, so the safe order
is: review → sync → redeploy.

### Review first

```sh
git fetch upstream
git log --oneline main..upstream/main
git diff main..upstream/main -- .env.example drizzle/ docker-compose.yml Dockerfile
```

Four paths are the ones that can actually break a deploy:

- **`.env.example`** — a newly required variable. `env.ts` refuses to start without `ORIGIN`,
  `BETTER_AUTH_SECRET`, or a database target, so add it in Coolify *before* redeploying.
- **`drizzle/`** — new migrations. The `migrate` service applies them automatically on deploy, but
  they are one-way: a rollback to an older image against a migrated database is not supported.
  Take a database backup before a deploy that carries migrations.
- **`docker-compose.yml`** — upstream changing the service layout (new service, renamed role,
  changed port) is the one case where this fork needs hand-editing, since
  `docker-compose.coolify.yml` is a parallel copy rather than an override. Port the change across.
- **`Dockerfile`** — build or runtime changes.

### Then

```sh
git merge --ff-only upstream/main
git push
```

Then Redeploy in Coolify. `migrate` applies pending schema changes and exits before `warcon` and
`worker` start; environment variables and the `warcon-db` volume are untouched by a redeploy.

If `--ff-only` ever refuses, something in this fork has diverged — find it with
`git diff --stat upstream/main` and move it back out of an upstream-owned file.

## Environment variables

Set in Coolify, not in the repository.

| Variable | Notes |
| --- | --- |
| `ORIGIN` | The exact public URL, no trailing slash. Cookies are only marked Secure when it starts with `https://`. |
| `BETTER_AUTH_SECRET` | `openssl rand -base64 32`. Changing it signs everyone out. |
| `ENCRYPTION_KEY` | `openssl rand -base64 32`. **Changing or losing it makes every stored RCON password permanently undecryptable.** Back it up separately from the database. |
| `RELAY_SECRET` | `openssl rand -base64 32`. Shared by the web and worker roles. |
| `POSTGRES_PASSWORD` | Cannot be rotated by editing this alone — Postgres only reads it at initdb, so the existing volume keeps the old one. Change it inside Postgres first. |
| `SETUP_TOKEN` | Required before the first deploy: `/setup` creates the site owner unauthenticated while the panel has zero users. |
| `ADDRESS_HEADER` | `x-forwarded-for` behind Traefik alone, `cf-connecting-ip` behind a proxied Cloudflare record. Wrong value collapses every client onto one apparent IP, and the 40-failure per-IP lockout then locks out everyone at once. |
| `XFF_DEPTH` | `1` with `x-forwarded-for`; leave unset with `cf-connecting-ip`. |
| `APP_NAME` | Optional. Shown in the UI and as the TOTP issuer. |
| `STEAM_API_KEY` | Optional. Enables persona, account age and VAC/ban lookups. |
| `DISCORD_CLIENT_ID` / `DISCORD_CLIENT_SECRET` | Optional, both or neither. Enables Discord sign-in. |
| `TURNSTILE_SITE_KEY` / `TURNSTILE_SECRET_KEY` | Optional, both or neither. |

## What a redeploy does not touch

Environment variables and the `warcon-db` volume are properties of the Coolify resource, not of a
deployment. Redeploy, restart and rebuild all preserve them. **Deleting the Coolify resource
deletes the volume.**

Enable Coolify's scheduled backups on the `db` service: the panel holds the entire ban history and
audit trail, and there is no export-everything button.
