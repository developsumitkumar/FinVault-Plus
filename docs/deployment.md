# Deployment Guide

FinVault can run locally with Docker Compose or be deployed to managed platforms.
The data pipeline (PySpark / Databricks) stays a separate local or cloud workload.

## Architecture

| Component | Local (Docker) | Production |
|---|---|---|
| Frontend | `http://localhost:3000` | Vercel (static SPA) |
| Backend API | `http://localhost:8000` | Render / Railway / Fly.io |
| PostgreSQL | Docker Compose | Render Postgres / Neon / Supabase |
| KYC uploads | Docker volume | Persistent disk / object storage |
| Pipeline | Local PySpark | Azure Databricks (see `docs/databricks.md`) |

---

## Docker Compose (recommended demo)

```bash
cp .env.example .env
docker compose up --build
```

| URL | Service |
|---|---|
| http://localhost:3000 | React app |
| http://localhost:8000/docs | FastAPI Swagger |
| http://localhost:8000/api/health | Health check |

**Demo login** (seeded by Alembic migration `003_user_auth_fields`):

| Email | Password | Role |
|---|---|---|
| `sumit@test.com` | `123456` | ADMIN (KYC approved) |

Register a new user to exercise the full onboarding flow (wallet + $10k opening balance).

Stop the stack:

```bash
docker compose down
```

---

## Backend — Render

1. Create a **PostgreSQL** database on Render (or use Neon/Supabase).
2. Create a **Web Service** from this repo:
   - **Root directory:** `backend`
   - **Runtime:** Docker (uses `backend/Dockerfile`)
   - **Health check path:** `/api/health`
3. Set environment variables:

| Variable | Example |
|---|---|
| `APP_ENV` | `production` |
| `LOG_LEVEL` | `INFO` |
| `DATABASE_URL` | `postgresql+pg8000://user:pass@host/db` |
| `JWT_SECRET_KEY` | long random string |
| `CORS_ORIGINS` | `https://your-app.vercel.app` |
| `KYC_UPLOAD_DIR` | `uploads/kyc` |

4. Add a **persistent disk** (or mounted volume) for `uploads/kyc` if you need KYC document storage across deploys.

Structured JSON logs are enabled automatically when `APP_ENV=production`.

---

## Frontend — Vercel

1. Import the repo in Vercel.
2. Set **Root Directory** to `frontend`.
3. Framework preset: **Vite**.
4. Build settings (defaults usually work):
   - Build command: `npm run build`
   - Output directory: `dist`
5. Environment variable:

| Variable | Value |
|---|---|
| `VITE_API_BASE_URL` | `https://your-api.onrender.com` |

6. Deploy. Vercel serves the SPA; client-side routing is handled by the static build.

Update the backend `CORS_ORIGINS` to include your Vercel URL after the first deploy.

---

## Environment reference

See [`.env.example`](../.env.example) for the full list. Key production settings:

```env
APP_ENV=production
LOG_LEVEL=INFO
DATABASE_URL=postgresql+pg8000://...
JWT_SECRET_KEY=<generate-a-strong-secret>
CORS_ORIGINS=https://your-frontend.vercel.app
VITE_API_BASE_URL=https://your-api.onrender.com
```

---

## Health checks

- **Liveness:** `GET /api/health` returns `{"status":"ok","database":"connected"}` when PostgreSQL is reachable.
- **Docker:** backend and frontend containers include `HEALTHCHECK` instructions.
- **Request tracing:** responses include `X-Request-ID`; logs include `request_id`, `method`, `path`, `status_code`, `duration_ms`.

---

## Data pipeline (optional cloud)

The app and pipeline share PostgreSQL. After wallet activity:

```bash
cd data-pipeline
source .venv/bin/activate
python -m local.run_pipeline --export
```

For Azure ADLS + Databricks, follow [`docs/azure-storage.md`](./azure-storage.md) and [`docs/databricks.md`](./databricks.md).

---

## Troubleshooting

| Issue | Fix |
|---|---|
| CORS errors in browser | Add frontend origin to backend `CORS_ORIGINS` |
| `database: unavailable` on health | Check `DATABASE_URL` host/credentials |
| Frontend calls wrong API | Rebuild with correct `VITE_API_BASE_URL` |
| KYC uploads lost on redeploy | Mount persistent storage for `uploads/kyc` |
