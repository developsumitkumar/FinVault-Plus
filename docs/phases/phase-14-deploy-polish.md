# Phase 14 — Deploy & Polish

Final phase: production-ready local stack, structured logging, CI, and deployment docs.

## Docker Compose full stack

```yaml
postgres  →  backend (FastAPI + Alembic migrate)  →  frontend (nginx SPA)
```

| Service | Port | Image |
|---|---|---|
| `postgres` | 5432 | `postgres:15` |
| `backend` | 8000 | `backend/Dockerfile` |
| `frontend` | 3000 | `frontend/Dockerfile` (nginx) |

```bash
docker compose up --build
```

- Backend entrypoint runs `alembic upgrade head` before `uvicorn`
- KYC uploads persisted in `finvault_kyc_uploads` volume
- Health checks on backend (`/api/health`) and frontend

## Structured logging

- `app/core/logging.py` — JSON logs when `APP_ENV=production|staging`, human-readable locally
- `app/middleware/request_logging.py` — request ID, method, path, status, duration
- Responses include `X-Request-ID` header

## Configuration

- `CORS_ORIGINS` — comma-separated allowed frontend origins (Docker + Vercel)
- See [`.env.example`](../../.env.example)

## CI

GitHub Actions workflow (`.github/workflows/ci.yml`):

- **backend** — PostgreSQL service + `pytest` (55 tests)
- **frontend** — `npm ci` + `npm run build`

## Local developer commands

```bash
make up              # docker compose up --build
make test-backend    # pytest with PYTHONPATH
make test-frontend   # production build
make test-pipeline   # PySpark tests (requires Java)
make verify-ledger   # fixture cycle check
```

## Deployment docs

[`docs/deployment.md`](../deployment.md) covers:

- Docker Compose demo walkthrough
- Render backend deployment
- Vercel frontend deployment
- Environment variables and troubleshooting

## Demo credentials

| Email | Password | Notes |
|---|---|---|
| `sumit@test.com` | `123456` | Admin, KYC approved (Alembic seed) |

## Deliverables

- [x] `backend/Dockerfile` + `frontend/Dockerfile` + full `docker-compose.yml`
- [x] Structured logging + request middleware
- [x] `CORS_ORIGINS` env config
- [x] `pytest.ini`, `Makefile`, GitHub Actions CI
- [x] `docs/deployment.md` (Render + Vercel)
- [x] README demo instructions

---

**All 14 phases complete.** FinVault is portfolio-ready.
