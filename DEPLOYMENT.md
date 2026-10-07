# PROXY-HUB Deployment Guide

Definitive runbook for the full stack (New API + DiscordGate sidecar + bot + Caddy) on a VPS with Docker. Verified against the current repo.

---

## 0. Prerequisites

- Docker Engine + Docker Compose v2 (`docker compose version`)
- A domain (e.g. `ai.example.com`) with an **A record → your VPS IP** (Caddy needs it for automatic HTTPS)
- A Discord application + bot (see step 1)
- SSH access to the VPS

---

## 1. Discord Developer Portal setup

Create an application at https://discord.com/developers/applications, then:

| What | Where | Env var |
|------|-------|---------|
| Application ID | General Information | `DISCORD_CLIENT_ID` |
| Application Secret | OAuth2 → General | `DISCORD_CLIENT_SECRET` |
| Public Key (optional) | General Information | `DISCORD_PUBLIC_KEY` |
| Bot token | Bot → Reset Token | `DISCORD_BOT_TOKEN` |
| Redirect URI | OAuth2 → Redirects → **add** `https://your-domain.com/auth/discord/callback` | `DISCORD_REDIRECT_URI` |
| Server (Guild) ID | right-click server → Copy Server ID | `TARGET_GUILD_ID` |

**Critical:** enable the privileged intent **Bot → Privileged Gateway Intents → Server Members Intent**. Without it the bot never receives `guildMemberRemove` / `guildBanAdd` events and revocation silently doesn't happen.

Invite the bot to your server (OAuth2 → URL Generator, scopes: `bot` + `identify`, permissions: `Manage Roles`/`Administrator` as you see fit).

---

## 2. Create `.env`

```bash
cd /path/to/PROXY-HUB/new-api
cp ../.env.example .env
sh scripts/generate-secrets.sh >> .env    # appends 6 strong secrets
```

`generate-secrets.sh` appends: `SESSION_SECRET`, `NEW_API_JWT_SECRET`, `INITIAL_ADMIN_PASSWORD`, `DB_PASSWORD`, `REDIS_PASSWORD`, `ROUTER_PASSWORD`.

Then **edit `.env`** and fill in the non-generated values:

```ini
# Discord (from step 1)
DISCORD_BOT_TOKEN=
DISCORD_CLIENT_ID=
DISCORD_CLIENT_SECRET=
DISCORD_REDIRECT_URI=https://your-domain.com/auth/discord/callback
TARGET_GUILD_ID=
DISCORD_INVITE_LINK=https://discord.gg/your-invite
DISCORD_OWNER_ID=            # your Discord ID (full mod access)
MOD_ROLE_ID=                 # optional mod role ID

# Site
SITE_URL=https://your-domain.com
SITE_DOMAIN=your-domain.com
DOMAIN=your-domain.com
CADDY_TLS_EMAIL=you@your-domain.com

# Admin panel access (comma-separated IPs allowed to hit /admin)
CADDY_ADMIN_ALLOWLIST=YOUR_VPS_OR_VPN_IP

# New API admin token — created in step 4, not now
NEW_API_ADMIN_TOKEN=
```

---

## 3. Two known blockers (fix before `up`)

The compose file is at `new-api/docker-compose.yml`. Two things will fail as committed:

**a) Build context for `bot` / `discord-gate`** — compose sets `context: .` (the `new-api/` dir) but their Dockerfiles `COPY packages/...` which lives at the **repo root**. Change both services to:

```yaml
build:
  context: ..            # repo root
  dockerfile: packages/discord-gate/Dockerfile
```

(same for `bot`: `packages/bot/Dockerfile`).

**b) `bridge` service** — it builds `../bridge`, which is **not in the repo**. Either:
- add the bridge source at `PROXY-HUB/bridge/`, or
- delete the `bridge` service block **and** the two `handle /connect*` / `handle /api/bridge/*` blocks in `new-api/Caddyfile` (those paths only matter when bridge exists).

---

## 4. First boot (New API only)

`discord-gate` refuses to start without a real `NEW_API_ADMIN_TOKEN`, so boot the core first:

```bash
cd /path/to/PROXY-HUB/new-api
docker compose up -d postgres redis new-api caddy
docker compose logs -f new-api        # watch it come up
```

Then:

1. Open `https://your-domain.com` → complete the setup wizard → create the root admin (the password field accepts anything; the generated `INITIAL_ADMIN_PASSWORD` is for the startup guard, not the wizard).
2. In the New API admin panel: **System → Personal Access Tokens** (root user) → generate an admin access token.
3. Put it in `.env` as `NEW_API_ADMIN_TOKEN`, then:

```bash
docker compose up -d discord-gate bot
docker compose logs -f discord-gate bot
```

If you skip step 2, `discord-gate` crash-loops with a zod env-validation error — that's expected.

---

## 5. Full stack up

```bash
cd /path/to/PROXY-HUB/new-api
docker compose up -d --build
docker compose ps            # all healthy
```

Expected services: `postgres`, `redis`, `bot`, `discord-gate`, `new-api`, `caddy` (+ `9router`, `bridge` if present).

Verify the gate is live:
- `https://your-domain.com` → redirected to Discord OAuth
- non-member → `https://your-domain.com/auth/denied?reason=not_member`
- member login → lands back on the site, `/auth/verify` returns `X-Discord-User-ID`
- slash commands (registered automatically on bot ready): `/checkaccess`, `/revokeaccess`, `/apikeys`, `/botstats`, `/syncmembers`

---

## 6. Local dev (without deploying)

Use the dev override (hot reload + localhost ports 3000–3002, 5432, 6379, 20128):

```bash
cd /path/to/PROXY-HUB/new-api
docker compose -f docker-compose.yml -f docker-compose.override.yml up -d --build
```

For the sidecar/bot packages standalone:

```bash
cd packages/discord-gate && npm install --legacy-peer-deps && npx tsc && npm run dev
cd packages/bot         && npm install && npm run dev     # needs discord-gate/dist first (npx tsc in sidecar)
```

Note: the sidecar reads env from `process.env` — export the same vars from your `.env` or run it through the compose override.

---

## 7. VPS hardening (production)

```bash
cd /path/to/PROXY-HUB/new-api
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```

- `docker-compose.prod.yml` pins images, sets resource limits, JSON log rotation, and removes all host ports except Caddy 80/443/443-udp.
- Firewall: only `80`/`443` (and `22` for SSH) public. Everything else is `127.0.0.1`-bound or on the internal-only Docker network (`internal: true`) — postgres/redis/bot are unreachable from outside.
- Keep `CADDY_ADMIN_ALLOWLIST` = your IP, or the admin panel (`/admin`, `/dashboard`, `/api/admin`) returns 403 to everyone.
- Caddy gets Let's Encrypt certs automatically using `CADDY_TLS_EMAIL`.

## 8. Operations

```bash
docker compose logs -f discord-gate bot caddy   # follow
docker compose restart discord-gate             # restart one service
docker compose down                             # stop (keep volumes)
docker compose up -d --build                    # rebuild after git pull
```

- **Audit trail:** Caddy writes JSON access logs to `/var/log/caddy/access.log` (volume `caddy_logs`), including `X-Discord-User-ID`. Query per user: `sh new-api/scripts/audit-log-query.sh <discord_id>`.
- **Backups:** volumes `postgres_data` and `redis_data` hold all state. `docker run --rm -v PROXY-HUB_postgres_data:/data -v $PWD:/backup alpine tar czf /backup/pg.tgz -C /data .`
- **Security guard:** `new-api` container entrypoint runs `scripts/check-defaults.sh`; if `INITIAL_ADMIN_PASSWORD` is empty/default or `SESSION_SECRET`/`NEW_API_JWT_SECRET` are < 32 chars, it exits 1 and the container won't start.
- **Never commit `.env`** (gitignored). Rotate secrets by re-running `generate-secrets.sh` and editing `.env`.
