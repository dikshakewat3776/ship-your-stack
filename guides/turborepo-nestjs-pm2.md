# Turborepo + NestJS (TypeScript) on a VPS with pnpm, Prisma, and PM2

Run a **pnpm + Turborepo** monorepo on your own server: install dependencies, generate and migrate **Prisma**, seed the database, **build** all packages with Turbo, and start or reload the NestJS API with **PM2**.

This pattern fits a repo layout like `services/<api-app>/` plus a shared package that owns the schema (often `packages/db` or `packages/common`).

---

## Architecture

| Piece | Platform | Role |
|--------|----------|------|
| Monorepo | Git on server (or deploy artifact) | pnpm workspaces + Turborepo task graph |
| API | PM2 | Runs compiled NestJS (`dist/main.js`) |
| Database | Your Postgres (managed or self-hosted) | Prisma migrations + optional tracked seeds |
| Config | Root `.env` on server | `DATABASE_URL` and app secrets (not committed) |

Typical flow on the production host: **pull code → `.env` present → one deploy script** (install → Prisma → build → PM2 reload).

---

## Prerequisites

On the **production server**:

- **Node.js** LTS (match your repo’s `engines` if set).
- **pnpm** (version aligned with `packageManager` in root `package.json`, if present).
- **PM2** installed globally (`npm i -g pm2`) and **startup** configured if you want survive reboot (`pm2 startup`, then `pm2 save`).
- Network access to **Postgres** from the server.
- Repository clone at a fixed path (e.g. `/var/www/my-api`).

In the **repository** (adjust names to your monorepo):

| Concept | Example placeholder |
|---------|---------------------|
| Prisma / shared DB package | pnpm filter name, e.g. `@my-org/db` or path `packages/db` |
| NestJS service folder | `services/api` or `apps/api` |
| Built entrypoint | `services/api/dist/main.js` (after `nest build` / `turbo build`) |
| PM2 app name | `my-api` (must match `ecosystem.config.js`) |
| Root env file | `.env` at monorepo root (or sourced by PM2) |

Scripts in the DB package commonly look like:

- `prisma:generate` — `prisma generate`
- `prisma:seed` — custom seed (often idempotent, e.g. tracked in a `*_seed_history` table)
- Migrations via `prisma migrate deploy` (production-safe; no `migrate dev`)

Ensure **`pnpm turbo build`** builds the NestJS app and any workspace dependencies Turbo declares in `turbo.json`.

---

## One-time server setup

1. Clone the repo and check out the branch you deploy (e.g. `main`).
2. Create **root `.env`** with at least `DATABASE_URL` and any variables NestJS reads in production (JWT secrets, feature flags, etc.). Restrict file permissions (`chmod 600 .env`).
3. Add **`ecosystem.config.js`** at the repo root (or under `deploy/`) pointing at your compiled `main.js` and `env_production`:

   ```js
   module.exports = {
     apps: [
       {
         name: 'my-api',
         script: 'services/api/dist/main.js',
         instances: 1,
         exec_mode: 'fork',
         env_production: {
           NODE_ENV: 'production',
         },
       },
     ],
   };
   ```

   Use `dotenv` in the ecosystem file or ensure the process loads `.env` if your app expects it at runtime.

4. Optionally install **log rotation** (`pm2 install pm2-logrotate`).

---

## End-to-end deploy order

On each release:

1. **Install** — `pnpm install --frozen-lockfile` (uses lockfile exactly; fails if `package.json` and lockfile disagree).
2. **Prisma generate** — client in the shared package.
3. **Migrations** — `prisma migrate deploy` against production DB.
4. **Seed** (if you use it) — only run seeds designed for production (idempotent / history-tracked).
5. **Build** — `pnpm turbo build`.
6. **PM2** — `start` on first deploy, `reload` on subsequent deploys with `--update-env`.

---

## Deploy script (reference)

Save as e.g. `scripts/prod.deploy.sh`, make executable (`chmod +x`), and run from the monorepo root after `.env` exists.

```bash
#!/usr/bin/env bash
# Production deploy — Turborepo NestJS backend + PM2
#
# Usage (on the production server):
#   cd /path/to/your-monorepo
#   ./scripts/prod.deploy.sh
#
set -euo pipefail

ROOT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")/.." && pwd)"
cd "$ROOT_DIR"

# pnpm filter for the workspace package that owns prisma/schema
DB_PKG="@my-org/db"
PM2_APP_NAME="my-api"
MAIN_SCRIPT="services/api/dist/main.js"

[[ -f .env ]] || { echo "error: missing root .env"; exit 1; }

pnpm install --frozen-lockfile

pnpm -F "$DB_PKG" prisma:generate
pnpm -F "$DB_PKG" exec prisma migrate deploy
pnpm -F "$DB_PKG" prisma:seed

pnpm turbo build

[[ -f "$MAIN_SCRIPT" ]] || { echo "error: build output not found: $MAIN_SCRIPT"; exit 1; }

if pm2 describe "$PM2_APP_NAME" >/dev/null 2>&1; then
  pm2 reload ecosystem.config.js --env production --update-env
else
  pm2 start ecosystem.config.js --only "$PM2_APP_NAME" --env production --update-env
fi

pm2 save
pm2 status
```

**Customize** `DB_PKG`, `PM2_APP_NAME`, `MAIN_SCRIPT`, and script paths to match your workspace names and Turbo pipeline.

If Prisma lives at the repo root instead of a package, replace filtered commands with `pnpm exec prisma generate`, `pnpm exec prisma migrate deploy`, and your seed script.

---

## Environment variables

| Variable | Where | Notes |
|----------|--------|--------|
| `DATABASE_URL` | Root `.env` | Production Postgres URL; required for migrate deploy |
| App secrets | `.env` or PM2 `env_production` | JWT, API keys, etc. — same names as local/staging where possible |
| `NODE_ENV` | PM2 | Set to `production` for deploys |

Never commit `.env`. Rotate credentials if they appear in logs or shell history.

---

## Verification

1. **`pm2 status`** — app `online`, stable restarts.
2. **Logs** — `pm2 logs my-api --lines 50` — no Prisma or Nest bootstrap errors.
3. **HTTP** — hit health or a read-only route (e.g. `GET /health`).
4. **Database** — confirm `_prisma_migrations` matches expectations; spot-check critical tables after seed.

---

## Ongoing update flow

- Pull or sync new code (git pull, CI artifact, or rsync).
- Run `./scripts/prod.deploy.sh` (or your CI step that runs the same commands).
- Watch PM2 reload; roll back by checking out the previous commit and re-running the script if needed.

---

## Troubleshooting

| Symptom | Likely cause |
|---------|----------------|
| `missing root .env` | Create `.env` on server before deploy |
| `frozen-lockfile` install failure | Lockfile out of date with `package.json`; fix in dev, commit lockfile, redeploy |
| `prisma migrate deploy` fails | Wrong `DATABASE_URL`, DB unreachable, or migration history conflict |
| Seed runs twice / bad data | Non-idempotent seed; use history table or guard in seed script |
| `build output not found` | Wrong `MAIN_SCRIPT` path or Turbo did not build that service (check `turbo.json` `dependsOn`) |
| PM2 starts old code | Forgot `turbo build`; or reload without rebuild |
| App up but 500 on DB routes | `DATABASE_URL` not loaded in PM2 process; use `--update-env` or explicit env in ecosystem |
| High memory after deploy | Use `pm2 reload` (graceful) vs `restart`; tune Nest/Node heap if needed |

---

## Security reminders

- Run the app as a non-root Linux user where possible.
- Restrict who can read `.env` and who can SSH to run deploys.
- Use least-privilege DB users for the application (separate from migration superuser if your policy requires it).
- Keep `pnpm install --frozen-lockfile` in CI and production so dependency supply chain matches what you tested.

---

## Quick checklist

1. Server has Node, pnpm, PM2, and Postgres connectivity.  
2. Root `.env` exists with valid `DATABASE_URL`.  
3. `ecosystem.config.js` names and script path match your NestJS build output.  
4. Deploy script variables (`DB_PKG`, `MAIN_SCRIPT`, `PM2_APP_NAME`) match the repo.  
5. One successful `./scripts/prod.deploy.sh` → API healthy in PM2 and over HTTP.