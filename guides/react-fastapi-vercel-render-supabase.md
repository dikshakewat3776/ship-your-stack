# React (Vite) + FastAPI + PostgreSQL on Vercel, Render, and Supabase

Deploy a **React (Vite) frontend on Vercel**, a **FastAPI backend on Render**, and **PostgreSQL on Supabase**, with database migrations run from your machine against Supabase.

---

## Architecture

| Piece | Platform | Role |
|--------|----------|------|
| Frontend | Vercel | Static Vite build; calls the API using a base URL env var (e.g. `VITE_API_BASE_URL`) |
| Backend | Render (Web Service, Python) | FastAPI + Gunicorn with Uvicorn workers |
| Database | Supabase (Postgres) | Managed Postgres; access via connection string |

Local development often uses a local Postgres (Docker optional) and a Vite dev-server proxy to the API on `localhost:8000`. Production relies on public URLs and environment variables only.

---

## End-to-end deployment order

Use **Supabase → Render → migrations (and optional seed) → Vercel**. You need the Render URL before setting the frontend API base URL, and a working `DATABASE_URL` on Render before API routes that use the database will succeed.

---

### Step 1 — Supabase: create Postgres and get `DATABASE_URL`

1. Go to [supabase.com](https://supabase.com), sign in, and click **New project**.
2. Choose organization, **name**, **region** (close to users or to Render if latency matters), and set a strong **database password**. Store it in a password manager.
3. Wait until provisioning finishes (dashboard shows the project as ready).
4. Open **Connect** (top) or **Project Settings → Database**.
5. Pick a connection mode that works from **Render** (recommended):
   - **Transaction pooler** or **Session pooler** — host looks like `*.pooler.supabase.com`, not `db.<ref>.supabase.co`.
   - **Direct** (`db.<ref>.supabase.co:5432`) often resolves to **IPv6 only**; Render may log `Network is unreachable` unless you have verified IPv6 connectivity.
6. Show a **URI** in the UI (not only `psql`). Copy the full **PostgreSQL URI**.
7. Ensure the URI includes **`sslmode=require`** (append `?sslmode=require` or `&sslmode=require` if missing).
8. Store this as **`DATABASE_URL`**. If the password contains special characters (`@`, `:`, `/`, `%`, `#`, etc.), URL-encode them in the password portion of the URI.

**Reference — connection modes**

| Mode | Typical use | Notes |
|------|-------------|--------|
| **Direct** | Long-lived servers | `db.<project-ref>.supabase.co:5432` — often **IPv6** in DNS. |
| **Transaction pooler** | Pooling, many short connections | `*.pooler.supabase.com:6543`, user often `postgres.<project-ref>`. |
| **Session pooler** | ORMs / longer sessions | Same pooler family; use **exact** host, port, and user from the UI. |

Example **shape** for the transaction pooler (always copy the real URI from Supabase):

`postgresql://postgres.<PROJECT_REF>:<PASSWORD>@<REGION-POOLER>.pooler.supabase.com:6543/postgres?sslmode=require`

---

### Step 2 — GitHub: push the application code

1. Push your application repository to GitHub on the branch you will deploy (commonly `main`).
2. For Vercel Hobby deploys, commit authors should use an email **verified** on the GitHub account linked to Vercel (see **Step 6** if deploys are blocked).

---

### Step 3 — Render: deploy the FastAPI backend

1. Go to [dashboard.render.com](https://dashboard.render.com) and sign in.
2. Click **New +** → **Web Service**.
3. **Connect** GitHub if prompted, then **select your repository** (and branch if not default).
4. Configure the service:
   - **Name:** any identifier (e.g. `my-api`).
   - **Region:** optional; match users or Supabase region if useful.
   - **Branch:** `main` or your deploy branch.
   - **Root Directory:** the folder that contains your FastAPI app (often `backend` in a monorepo).
   - **Runtime:** Python.
   - **Build Command:** `pip install -r requirements.txt` (adjust if you use Poetry or another tool).
   - **Start Command** (typical for FastAPI + Uvicorn under Gunicorn):  
     `gunicorn -k uvicorn.workers.UvicornWorker --bind 0.0.0.0:$PORT main:app`  
     Replace `main:app` with your module and ASGI app object if different.
5. **Instance type:** Free is fine for experiments; expect cold starts on free tiers.
6. **Environment variables** — add at least:

   | Key | Value |
   |-----|--------|
   | `DATABASE_URL` | The **Supabase pooler** URI from Step 1 (same value you use locally for migrations). |
   | `CORS_ORIGINS` | Your Vercel origin, e.g. `https://<project>.vercel.app` (can add or edit after Step 5). |

   Add any others your app requires (examples many stacks use: `JWT_SECRET_KEY`, `JWT_ALGORITHM`, `JWT_ACCESS_TOKEN_EXPIRE_MINUTES`, and optional seed credentials). Names and values depend on your codebase.

   Optional: **`PYTHON_VERSION=3.12.8`** (or another full `3.12.x`) if the platform does not pick the version you need from the repo.

7. Click **Create Web Service** and wait for the first deploy.
8. Copy the **public URL** (e.g. `https://<name>.onrender.com`) for Vercel in Step 5.
9. **Smoke test:** open `https://<your-render-host>/health` (or your app’s health path) — expect a simple OK response if you implemented one. If other routes return 500, check logs for database connection errors (wrong `DATABASE_URL`, IPv6/direct host issues, etc.).

**Python version on Render:** Pin **Python 3.12** (e.g. `.python-version` at the repo root or `PYTHON_VERSION` on the service) if newer defaults break wheels for packages like `pydantic-core` (Rust/maturin build failures).

**Render Shell:** Often unavailable on **free** web tiers. Run migrations from your laptop (Step 4), not on the server, unless your plan allows shell access.

---

### Step 4 — Your machine: run migrations and optional seed

Use the **same** `DATABASE_URL` as Render (same Supabase database).

1. Use a **Python version** compatible with your project (3.12 is common for current FastAPI stacks).
2. From your backend directory (paths are examples):

   ```bash
   cd backend   # or your API root
   python -m venv .venv
   source .venv/bin/activate    # Windows: .venv\Scripts\activate
   pip install -r requirements.txt
   export DATABASE_URL='postgresql://...'   # same as Render
   python -m alembic upgrade head           # if you use Alembic
   # Optional: python scripts/seed_data.py   # only if your repo provides seeding
   ```

3. Confirm migrations finish without errors. Alembic reads `DATABASE_URL` through ConfigParser; **`%` in passwords** often needs escaping in `alembic/env.py` or URL-encoding — see [Alembic database URLs](https://alembic.sqlalchemy.org/en/latest/tutorial.html#the-ini-file).
4. Re-test the API: a representative **GET** to your REST prefix (e.g. `https://<render>/api/v1/...`) should return **200** when the DB is reachable. **500** usually indicates connection or configuration issues on Render.

---

### Step 5 — Vercel: deploy the Vite frontend

1. Go to [vercel.com](https://vercel.com) and sign in (same GitHub account as the repo is simplest).
2. **Add New… → Project** → **Import** the repository.
3. **Configure Project:**
   - **Framework Preset:** Vite (auto-detected is usually fine).
   - **Root Directory:** the folder with `vite.config` (often `frontend` in a monorepo).
   - **Build Command:** `npm run build` or `pnpm build` / `yarn build` (defaults are usually correct).
   - **Output Directory:** `dist` (Vite default).
4. **Environment Variables** — add the variable your frontend uses for the API origin, for example:

   | Name | Value |
   |------|--------|
   | `VITE_API_BASE_URL` | `https://<your-service>.onrender.com` — **origin only**, no path |

   Rules: **no** trailing slash; **do not** append `/health` or `/api` to the base URL if your client code already prefixes paths.

5. **Deploy**, then open the **Production** URL.

6. **CORS:** If the UI loads but browser requests fail with CORS errors, set **`CORS_ORIGINS`** on Render to the exact Vercel URL (`https://<project>.vercel.app`), save, and redeploy or restart the Render service.

---

### Step 6 — Vercel + GitHub (Hobby): if deploys are blocked

1. **“Could not associate the committer with a GitHub user”**  
   In **this repository:**  
   `git config user.email "<verified-github-email>"`  
   `git config user.name "Your Name"`  
   Push a new commit. The email must be verified under GitHub **Settings → Emails**.

2. **“Deployment canceled / unverified commit”**  
   In Vercel: **Project → Settings → Git** — disable **Require verified commits**, or sign commits per GitHub’s docs.

---

### Step 7 — Optional: `vercel.json` at repo root

If you deploy **only** the frontend from a subdirectory and the API lives on Render, you may not need a root **`vercel.json`** with multi-service or experimental settings. Remove or trim it to avoid confusion if it is unused.

---

## Reference: Alembic and migrations

- Keep migration history in version control; run `alembic upgrade head` against the same database your production API uses.
- Fix broken migration revisions before re-running; repair rare partial states manually if a deploy was interrupted.

---

## Troubleshooting

### Backend

| Symptom | Likely cause |
|---------|----------------|
| `pydantic-core` / Rust / maturin build failure on Render | Pin **Python 3.12** (`.python-version` or `PYTHON_VERSION`). |
| `Network is unreachable` to `db.*.supabase.co` (IPv6) | Use a **pooler** `DATABASE_URL` on Render, not the direct `db` host. |
| `500` on API routes but health check OK | Database connection failing; check Render logs and `DATABASE_URL`. |
| `500` with Supabase **transaction pooler** (`:6543`) | **PgBouncer** transaction mode plus SQLAlchemy (or similar) **connection pooling** can conflict; use **`NullPool`** for that endpoint, switch pooler mode, or follow your ORM’s Supabase/PgBouncer guidance. |

### Frontend

| Symptom | Likely cause |
|---------|----------------|
| Wrong paths like `/health/api/v1/...` | API base URL incorrectly includes `/health` or another path segment. |
| CORS errors | Add your exact Vercel origin to **`CORS_ORIGINS`** on the backend. |
| `Failed to fetch` | Wrong API URL, mixed HTTP/HTTPS, or backend down / cold start. |

### Database

| Symptom | Likely cause |
|---------|----------------|
| Alembic `invalid interpolation` / `%` | Escape `%` for ConfigParser and/or URL-encode the password. |
| DNS fails locally for `db.*` | Direct host may be IPv6-only; try the pooler URI or another network. |

---

## Quick verification

1. **Backend:** Health or root check returns success (e.g. `GET https://<render>/health` → `{"status":"ok"}` if that is what you implemented).  
2. **Backend:** At least one real **GET** that hits the DB returns **200** with JSON.  
3. **Frontend:** Production URL loads; primary flows work without errors in the browser console or network tab.

---

## Security reminders

- Never commit `.env` files with production secrets.
- Rotate database passwords if they were exposed.
- Keep auth secrets (e.g. JWT signing keys) strong and unique per environment.
