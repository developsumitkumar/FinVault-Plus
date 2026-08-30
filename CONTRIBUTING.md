# Contributing to FinVault

Thank you for your interest in FinVault! This project is primarily a **portfolio and learning codebase**, but contributions that improve correctness, documentation, tests, or developer experience are welcome.

---

## Before you start

1. Read [`PROJECT_PLAN.md`](PROJECT_PLAN.md) to understand phase boundaries and scope.
2. Skim [`docs/architecture.md`](docs/architecture.md) for layering rules (routes → services → repositories).
3. For UI work, follow [`design.md`](design.md) — do not introduce one-off colors or patterns.
4. For Java parity questions, see [`docs/java-port-reference.md`](docs/java-port-reference.md).

---

## How to set up locally

### Option A — Docker (recommended)

```bash
cp .env.example .env
docker compose up --build
```

- App: http://localhost:3000  
- API docs: http://localhost:8000/docs  

### Option B — Native dev

**Backend**

```bash
cd backend
python3 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
alembic upgrade head
uvicorn app.main:app --reload --port 8000
```

**Frontend**

```bash
cd frontend
npm install --legacy-peer-deps
npm run dev
```

**Database**

PostgreSQL 15+ with database `finvault` (see `.env.example` for connection string).

**Optional demo data**

```bash
make seed-demo-small   # quick dataset
make seed-demo         # ~1000 users, 5 years of ledger history
```

---

## Development workflow

### Branching

- Branch from `main` (or `master`).
- Use descriptive names: `fix/kyc-resubmit-history`, `feat/passbook-export`, `docs/readme-screenshots`.

### Scope

- **One concern per PR** — avoid mixing unrelated features.
- Match existing code style and folder conventions.
- Do not commit secrets (`.env`, credentials, API keys).
- Do not run destructive git commands on shared branches without agreement.

### Backend changes

| Rule | Detail |
|---|---|
| Layering | Routes validate input and call services; services own business logic; repositories own SQL |
| Ledger | Financial writes must be atomic; use `LedgerService` patterns |
| Migrations | Every schema change needs an Alembic revision in `backend/alembic/versions/` |
| Tests | Add or update pytest cases in `backend/tests/` |
| API | Keep `/api` prefix; document new endpoints via OpenAPI (Pydantic models) |

Run tests:

```bash
make test-backend
# or
cd backend && PYTHONPATH=. pytest
```

Tests expect PostgreSQL database `finvault_test` (CI uses the same).

### Frontend changes

| Rule | Detail |
|---|---|
| Design system | Use tokens from `design.md` and shared components under `components/` |
| Data fetching | Prefer TanStack Query hooks in `hooks/queries.ts` |
| Forms | React Hook Form + Zod |
| Charts | Use hex colors from `lib/chart-colors.ts` (Tremor + Tailwind v4) |

Run checks:

```bash
make test-frontend
# or
cd frontend && npm run build
```

### Data pipeline changes

- Shared transform logic lives in `data-pipeline/common/transforms/`.
- Local runner: `python -m local.run_pipeline`
- Verify: `make verify-ledger` or `python -m local.verify_ledger_cycle --fixture`

```bash
make test-pipeline
```

---

## Pull request checklist

- [ ] Change is scoped and explained in the PR description
- [ ] Backend tests pass (`make test-backend`)
- [ ] Frontend builds (`make test-frontend`)
- [ ] New API routes have tests where behavior is non-trivial
- [ ] Alembic migration included if the schema changed
- [ ] No secrets or generated artifacts committed (except intentional screenshots in `docs/screenshots/`)
- [ ] `PROJECT_PLAN.md` updated if you complete or start a named phase
- [ ] README / docs updated if setup steps or public behavior changed

---

## Screenshots

If your PR changes visible UI, consider refreshing README screenshots:

```bash
# With backend + frontend running locally
npm install playwright --no-save
npx playwright install chromium
node scripts/capture-readme-screenshots.mjs
```

See [`docs/screenshots/README.md`](docs/screenshots/README.md).

---

## Reporting issues

When opening an issue, include:

- **What you expected** vs **what happened**
- Steps to reproduce
- Environment (OS, Python/Node versions, Docker vs native)
- Relevant logs or API error messages (redact tokens)

---

## Code of conduct

Be respectful and constructive. This is a learning project — clear explanations and kind reviews help everyone.

---

## Questions?

Open a GitHub issue or reach out via [GitHub @developsumitkumar](https://github.com/developsumitkumar).
