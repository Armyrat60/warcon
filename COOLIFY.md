# Running this fork on Coolify

This fork exists to put updates under our control. It **adds** two files
(`docker-compose.coolify.yml` and this one) and **modifies none**, so every upstream merge is a
fast-forward and can never conflict.

Deployment pulls a **pinned image** that this fork builds for itself, rather than building from
source on the server. Nothing on the panel changes unless we tag a new version and deliberately
point Coolify at it — a redeploy, a restart, or a host reboot all pull the same bytes.

## One-time setup

### 1. Enable Actions on the fork

GitHub does not register a fork's workflows until you ask it to. Open the repository's **Actions**
tab and click *"I understand my workflows, go ahead and enable them"*. Without this, tagging a
release silently builds nothing.

### 2. Cut the first release

```sh
git tag v0.2.0
git push origin v0.2.0
```

`release.yml` builds linux/amd64 + linux/arm64 and publishes:

- `ghcr.io/armyrat60/warcon:0.2.0` ← pin this one
- `ghcr.io/armyrat60/warcon:0.2`
- `ghcr.io/armyrat60/warcon:latest` ← do not use

Note the image tag drops the `v`: git tag `v0.2.0` produces image tag `0.2.0`.

### 3. Make the package pullable

After the first publish, open the package under the repository's **Packages** and either set its
visibility to **Public**, or leave it private and add a GHCR registry credential in Coolify (a
GitHub PAT with `read:packages`). Public is simpler; the image contains no secrets.

### 4. Coolify resource

| Setting | Value |
| --- | --- |
| Build Pack | Docker Compose |
| Repository | `Armyrat60/warcon`, branch `main` |
| Docker Compose Location | `/docker-compose.coolify.yml` |
| Automatic Deployment | **off** |
| Domain | on the `warcon` service only, port 3000 |

Everything else lives in the Environment Variables tab (below), never in a committed `.env`.

## Updating

Four steps, none of them automatic.

1. **Sync.** The fork modifies no upstream file, so GitHub's **Sync fork** button on the repository
   page does the whole merge in one click. It will always offer a fast-forward. This deploys
   nothing.
2. **Review** what you just took (see below).
3. **Tag.** `git tag v0.2.1 && git push origin v0.2.1`. Wait for the Actions run to go green.
4. **Deploy.** Change `WARCON_VERSION` to `0.2.1` in Coolify and hit Redeploy.

Step 4 is the only one that touches production, it is one environment variable, and it is
reversible — which is the whole point of pinning.

### What to review before tagging

```sh
git fetch upstream
git log --oneline main..upstream/main
git diff main..upstream/main -- .env.example drizzle/ docker-compose.yml Dockerfile
```

Four paths can actually break a deploy:

- **`.env.example`** — a newly required variable. `env.ts` refuses to start without `ORIGIN`,
  `BETTER_AUTH_SECRET`, or a database target, so add it in Coolify *before* redeploying.
- **`drizzle/`** — new migrations. The `migrate` service applies them automatically, but they are
  **one-way**. See rollback below.
- **`docker-compose.yml`** — upstream changing the service layout (new service, renamed role,
  changed port) is the one case needing hand work, since `docker-compose.coolify.yml` is a
  parallel copy rather than an override. Port the change across.
- **`Dockerfile`** — build or runtime changes.

### Rolling back

Set `WARCON_VERSION` to the previous version and redeploy. This is safe **only if that update
carried no migration** — a newer schema against an older image is not supported. If `drizzle/`
changed, roll back by restoring the database backup taken before the deploy, then changing the
version.

So: **back up the database before any deploy whose diff touched `drizzle/`.**

## Environment variables

Set in Coolify, not in the repository.

| Variable | Notes |
| --- | --- |
| `WARCON_VERSION` | The image tag to run, e.g. `0.2.0`. Never `latest`. This is the update switch. |
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
