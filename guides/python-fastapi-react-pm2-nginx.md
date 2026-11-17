# Python (FastAPI) + React on a VPS with venv, Gunicorn, PM2, and Nginx

Deploy a **Python 3.11** API (FastAPI under **Gunicorn + Uvicorn workers**) with **PM2**, optional **React** static UI behind **Nginx** path prefixes. Suited to staging or single-server production where one host runs the backend and Nginx terminates TLS and routes `/app/`, `/app-api/`, and static assets.

---

## Architecture

| Piece | Platform | Role |
|--------|----------|------|
| API | PM2 → Gunicorn (`UvicornWorker`) | ASGI app bound to `127.0.0.1:<port>` |
| Python env | `venv/` at repo root | Isolated deps from `requirements-staging.txt` + editable install |
| Config | Root `.env` | Secrets and `PORT` / `BACKEND_PORT`; loaded before PM2 start |
| Reverse proxy | Nginx (typical) | Public URLs → upstream port; separate paths for UI vs API |
| Frontend | React build (Vite/CRA) | Often served at a subpath; built assets under `/static/` or similar |

Example **path layout** (customize names in Nginx and app settings):

| Public path | Purpose |
|-------------|---------|
| `/my-app/` | React UI |
| `/my-app-api/v1/` | REST API (proxied to backend) |
| `/my-app/static/` | Built JS/CSS (or same as UI mount) |

Default **upstream port** in many setups is `8083` so Nginx `proxy_pass` matches the app bind port.

---

## Prerequisites

On the server:

- **Python 3.11** on `PATH` (or set `PYTHON=python3.11` in your script).
- **PM2** (`npm install -g pm2`); optional `pm2 startup` + `pm2 save`.
- **curl** for post-deploy health checks.
- **Nginx** configured if users hit the app on 443/80, not the raw backend port.

In the repo:

| File | Purpose |
|------|---------|
| `requirements-staging.txt` (or `requirements.txt`) | Locked/staging deps including `gunicorn`, `uvicorn` |
| `pyproject.toml` / `setup.cfg` | Supports `pip install -e .` for your package |
| `.env.example` | Template copied on first deploy if `.env` is missing |
| `ecosystem.config.cjs` | PM2 app: Gunicorn command, workers, `env`, script args |
| `deploy/nginx/<app>.conf` (optional) | Reference vhost matching your path prefixes |
| `scripts/deploy-staging.sh` | One-shot install, validate, start, health check |

Build the **React** app before or inside deploy (e.g. `cd frontend && npm ci && npm run build`) and configure FastAPI or Nginx to serve `dist/` at your UI base path.

---

## `ecosystem.config.cjs` (pattern)

PM2 runs Gunicorn with Uvicorn workers; **port and worker count** often live in ecosystem `args` or env so the deploy script can export `PORT` and `WORKERS` before `pm2 start`:

```js
module.exports = {
  apps: [
    {
      name: process.env.PM2_APP_NAME || 'my-app-api',
      cwd: __dirname,
      script: 'venv/bin/gunicorn',
      args: [
        '-k', 'uvicorn.workers.UvicornWorker',
        '--bind', `127.0.0.1:${process.env.PORT || 8083}`,
        '--workers', String(process.env.WORKERS || 3),
        'my_package.api:app', // module:app — adjust to your import path
      ],
      env: {
        NODE_ENV: 'staging',
      },
    },
  ],
};
```

Use the same **`PM2_APP_NAME`** in the deploy script and ecosystem. If you change bind port or workers, **recreate** the PM2 process (delete + start) so new args apply; `reload` alone may not refresh Gunicorn CLI args.

---

## Environment variables

| Variable | Notes |
|----------|--------|
| `PORT` / `BACKEND_PORT` | Backend listen port; CLI arg can override |
| `WORKERS` | Gunicorn worker count (default e.g. `3`) |
| `PM2_APP_NAME` | PM2 process name |
| App-specific secrets | e.g. DB URLs, LLM gateway keys — validate before start |

**Pattern:** Load `.env` in the deploy shell with `set -a; source .env; set +a` so the current process (and PM2 with `--update-env`) sees values. OS-level env vars override values from Pydantic `env_file` inside the app — document that for operators.

On first deploy, if `.env` is missing, copy `.env.example` and **exit** until secrets are filled in.

Add a small **pre-flight check** (Python or shell) for required keys your API cannot run without.

---

## End-to-end deploy order

1. Ensure **Python 3.11** and **PM2** exist.
2. Create **`venv`** if missing; activate and `pip install -U pip`.
3. Install **`requirements-staging.txt`** and **`pip install -e .`**.
4. **Import check** — e.g. `gunicorn`, `uvicorn`, and `from my_package.api import app`.
5. Ensure **`.env`** exists and required variables are non-empty.
6. Create **runtime dirs** (e.g. `logs/`).
7. Export **`PORT`**, **`WORKERS`**, **`PM2_APP_NAME`** (and any overrides).
8. **PM2:** delete existing app if rebinding port/workers; `pm2 start ecosystem.config.cjs --only <name> --update-env`; `pm2 save`.
9. **Health loop:** `curl -sf` against your health URL (e.g. `http://127.0.0.1:<port>/api/v1/health`) for up to ~30s.

Deploy script usage pattern:

```bash
./scripts/deploy-staging.sh [port] [workers]
# Env overrides: PORT, BACKEND_PORT, WORKERS, PM2_APP_NAME
```

---

## Deploy script (reference)

Generalized from a FastAPI + LiteLLM-style staging flow. Replace import paths, health URLs, and validation with your app’s names.

```bash
#!/usr/bin/env bash
# Deploy FastAPI API with PM2 (gunicorn + uvicorn workers)
# Usage: ./scripts/deploy-staging.sh [port] [workers]
#
set -euo pipefail

ROOT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")/.." && pwd)"
cd "$ROOT_DIR"

CLI_PORT="${1:-}"
WORKERS="${2:-${WORKERS:-3}}"
PM2_APP_NAME="${PM2_APP_NAME:-my-app-api}"

VENV_DIR="$ROOT_DIR/venv"
ENV_FILE="$ROOT_DIR/.env"
ENV_EXAMPLE_FILE="$ROOT_DIR/.env.example"
ECOSYSTEM_FILE="$ROOT_DIR/ecosystem.config.cjs"
PYTHON="${PYTHON:-python3.11}"

command -v "$PYTHON" >/dev/null || { echo "Error: $PYTHON not found"; exit 1; }
command -v pm2 >/dev/null || { echo "Error: pm2 not found"; exit 1; }
[[ -f "$ECOSYSTEM_FILE" ]] || { echo "Error: missing $ECOSYSTEM_FILE"; exit 1; }

[[ -d "$VENV_DIR" ]] || "$PYTHON" -m venv "$VENV_DIR"
# shellcheck source=/dev/null
source "$VENV_DIR/bin/activate"

python -m pip install -U pip
python -m pip install -r requirements-staging.txt
python -m pip install -e .

python -c "import gunicorn, uvicorn; from my_package.api import app"

if [[ ! -f "$ENV_FILE" ]]; then
  cp "$ENV_EXAMPLE_FILE" "$ENV_FILE"
  echo "Update $ENV_FILE then re-run."
  exit 1
fi

mkdir -p "$ROOT_DIR/logs"

# Optional: validate required .env keys with python-dotenv or a small inline script

set -a
# shellcheck source=/dev/null
source "$ENV_FILE"
set +a

if [[ -n "$CLI_PORT" ]]; then
  PORT="$CLI_PORT"
else
  PORT="${BACKEND_PORT:-${PORT:-8083}}"
fi

export PORT WORKERS PM2_APP_NAME BACKEND_PORT="$PORT"
HEALTH_URL="http://127.0.0.1:${PORT}/api/v1/health"

if pm2 describe "$PM2_APP_NAME" >/dev/null 2>&1; then
  pm2 delete "$PM2_APP_NAME" >/dev/null
fi

pm2 start "$ECOSYSTEM_FILE" --only "$PM2_APP_NAME" --update-env
pm2 save >/dev/null 2>&1 || true

for _ in $(seq 1 30); do
  if curl -sf "$HEALTH_URL" >/dev/null; then
    echo "Deployment successful: $HEALTH_URL"
    exit 0
  fi
  sleep 1
done

echo "Deployment failed — backend not healthy."
pm2 logs "$PM2_APP_NAME" --lines 30 --nostream || true
exit 1
```

**Nginx:** Keep a checked-in sample vhost under `deploy/nginx/` whose `proxy_pass` port and `location` blocks match `PORT` and your public path prefixes.

---

## React + subpath hosting

1. Set the frontend **base path** in Vite (`base: '/my-app/'`) or equivalent so asset URLs work behind Nginx.
2. Build: `npm run build` → output in `frontend/dist` (or your layout).
3. Either:
   - **FastAPI** mounts static files and optional SPA fallback for the UI, or  
   - **Nginx** serves `dist/` for `/my-app/` and proxies only API routes to Gunicorn.

Align **API client base URL** in React with the public API prefix (`/my-app-api/v1`), not only `localhost`.

---

## Verification

1. `pm2 status` — app online.  
2. `curl -sf http://127.0.0.1:<port>/api/v1/health` on the server.  
3. Through Nginx: UI loads at `/my-app/`, API at `/my-app-api/v1/...`, no mixed-content or 404 on JS bundles.  
4. `pm2 logs <app>` — no import or env errors after worker fork.

---

## Troubleshooting

| Symptom | Likely cause |
|---------|----------------|
| `python3.11 not found` | Install 3.11 or set `PYTHON` to your binary |
| Import check fails | Wrong editable package name or missing dep in requirements |
| Exits after copying `.env.example` | Secrets not configured — fill `.env` |
| Health check timeout | Wrong health path, app crash on boot, or port conflict |
| PM2 up but Nginx 502 | Upstream port mismatch vs `ecosystem` / Nginx `proxy_pass` |
| Static assets 404 | Wrong Vite `base` or Nginx `alias` path |
| Stale workers after port change | Old PM2 process; use delete + start, not only reload |
| Env ignored in workers | Shell did not `source .env` before start; use `--update-env` |

---

## Security reminders

- Do not commit `.env`; restrict `chmod 600 .env` on the server.  
- Bind Gunicorn to **`127.0.0.1`** when Nginx is the only public entrypoint.  
- Terminate TLS at Nginx; do not expose the raw backend port on a public interface unless required.  
- Rotate API keys if they were logged during failed deploys.

---

## Quick checklist

1. Python 3.11, PM2, and (if used) Nginx installed.  
2. `ecosystem.config.cjs` points at `my_package.api:app` and correct bind port.  
3. `.env` complete; deploy script validates critical keys.  
4. React built with correct public base path.  
5. `./scripts/deploy-staging.sh` exits 0 and health URL responds; public paths work via Nginx.