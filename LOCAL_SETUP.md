# Running FinVault locally (macOS and Windows)

This guide takes a fresh machine to the full stack running:

| Part | What it is | Port |
|---|---|---|
| PostgreSQL | operational database (users, wallets, ledger…) | 5432 |
| Backend | FastAPI API + admin data-platform endpoints | 8000 |
| Frontend | React app (customer app + admin **Data engineering** page) | 5173 |
| Airflow | orchestrates the data pipeline (export → quality gate → Spark → dbt) | 8080 |
| dbt docs | optional docs site for the warehouse | 8081 |

There are two levels; do the first, then the second if you want the data-engineering stack:

- **Level 1 — the app:** Postgres + backend + frontend. Works natively on macOS and Windows.
- **Level 2 — the data platform:** PySpark pipeline, Delta Lake, dbt, Airflow, benchmarks.
  Works natively on macOS and Linux. **On Windows use WSL2** (Airflow doesn't run on native Windows, and Spark needs extra Hadoop binaries there).

> **Windows users — pick one route:**
> - **Route A (recommended): everything inside WSL2 Ubuntu.** Follow the **macOS/Linux** commands in this guide inside the Ubuntu terminal. Section 0 shows how to set WSL2 up.
> - **Route B: native Windows for Level 1 only.** Use the **Windows (PowerShell)** commands. For Level 2 you'd still need WSL2.

---

## 0. Windows only — set up WSL2 (Route A)

1. Open **PowerShell as Administrator** and run:
   ```powershell
   wsl --install -d Ubuntu-24.04
   ```
2. Restart when asked, open **Ubuntu** from the Start menu, and create a Linux user.
3. Keep the project **inside the Linux filesystem** (e.g. `~/code/FinVault-`), not under `/mnt/c/...` — it's much faster.
4. Install the basics inside Ubuntu:
   ```bash
   sudo apt update && sudo apt install -y build-essential git curl unzip
   ```
5. From here on, use the **macOS/Linux** commands in this guide (use `apt` where macOS says `brew`). Browser URLs like `http://localhost:5173` work from Windows.

---

## 1. Install the prerequisites

| Tool | Version | Why |
|---|---|---|
| Git | any | clone the repo |
| Python | **3.13** (3.12 also works for Level 1) | backend, pipeline, Airflow (Airflow 3.3 supports Python ≤ 3.13 — **not 3.14+**) |
| Node.js | **20+** | frontend |
| PostgreSQL | **15+** | database |
| Java (JDK) | **17 or 21** | Spark (Level 2) |
| GNU make | any | shortcuts like `make dev` (optional — every command is also listed in full) |

### macOS (Homebrew)

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"   # if you don't have Homebrew
brew install git python@3.13 node@20 postgresql@15 openjdk@17
brew services start postgresql@15
echo 'export JAVA_HOME="$(/usr/libexec/java_home -v 17)"' >> ~/.zshrc
echo 'export PATH="/opt/homebrew/opt/node@20/bin:/opt/homebrew/opt/postgresql@15/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc
python3.13 --version && node --version && psql --version && java -version
```

> If you installed Python from **python.org** instead of Homebrew, run *“Install Certificates.command”* from `/Applications/Python 3.13/` once — otherwise some installs fail with `CERTIFICATE_VERIFY_FAILED`.

### Linux / WSL2 Ubuntu

```bash
sudo add-apt-repository -y ppa:deadsnakes/ppa && sudo apt update
sudo apt install -y python3.13 python3.13-venv python3.13-dev postgresql postgresql-contrib openjdk-17-jdk
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash - && sudo apt install -y nodejs
sudo service postgresql start
python3.13 --version && node --version && psql --version && java -version
```

### Windows native (Route B, PowerShell)

```powershell
winget install --id Git.Git -e
winget install --id Python.Python.3.13 -e
winget install --id OpenJS.NodeJS.LTS -e
winget install --id PostgreSQL.PostgreSQL.15 -e     # remember the 'postgres' password you choose
```

Close and reopen PowerShell, then check:

```powershell
py -3.13 --version; node --version; & "C:\Program Files\PostgreSQL\15\bin\psql.exe" --version
```

> PowerShell may block virtual-env activation scripts. Allow them for your user once:
> `Set-ExecutionPolicy -Scope CurrentUser RemoteSigned`

---

## 2. Get the code

```bash
git clone https://github.com/developsumitkumar/FinVault-.git
cd FinVault-
```

---

## 3. Create the database

The app expects a role `finvault_app` (password `change-me`) owning two databases: `finvault` (the app) and `finvault_test` (backend tests).

**macOS** (Homebrew Postgres lets your user connect without a password):

```bash
psql postgres -c "CREATE ROLE finvault_app WITH LOGIN PASSWORD 'change-me' CREATEDB;"
psql postgres -c "CREATE DATABASE finvault OWNER finvault_app;"
psql postgres -c "CREATE DATABASE finvault_test OWNER finvault_app;"
```

**Linux / WSL2:**

```bash
sudo -u postgres psql -c "CREATE ROLE finvault_app WITH LOGIN PASSWORD 'change-me' CREATEDB;"
sudo -u postgres psql -c "CREATE DATABASE finvault OWNER finvault_app;"
sudo -u postgres psql -c "CREATE DATABASE finvault_test OWNER finvault_app;"
```

**Windows native (PowerShell)** — enter the `postgres` password you set during install:

```powershell
$psql = "C:\Program Files\PostgreSQL\15\bin\psql.exe"
& $psql -U postgres -c "CREATE ROLE finvault_app WITH LOGIN PASSWORD 'change-me' CREATEDB;"
& $psql -U postgres -c "CREATE DATABASE finvault OWNER finvault_app;"
& $psql -U postgres -c "CREATE DATABASE finvault_test OWNER finvault_app;"
```

> **Docker alternative:** if you have Docker, `docker compose up -d postgres` starts a ready-made Postgres with the same user, password and `finvault` database (create `finvault_test` yourself if you want to run tests).

---

## 4. Configure environment files

The backend reads `backend/.env`; the data pipeline reads `.env` at the repo root. Create both from the template:

**macOS / Linux / WSL2:**

```bash
cp .env.example .env
cp .env.example backend/.env
```

**Windows (PowerShell):**

```powershell
Copy-Item .env.example .env
Copy-Item .env.example backend\.env
```

Then **edit `backend/.env`** and change these two lines so the AI features work without API keys:

```
DECISION_PROVIDER=stub
LLM_PROVIDER=stub
```

(`jev` needs an OpenRouter key; `ollama` needs Ollama running locally. `stub` gives deterministic offline answers.)

For anything beyond your own machine, also change `JWT_SECRET_KEY` to a long random string.

---

## 5. Backend (FastAPI)

**macOS / Linux / WSL2:**

```bash
cd backend
python3.13 -m venv .venv
.venv/bin/pip install --upgrade pip
.venv/bin/pip install -r requirements.txt
.venv/bin/alembic upgrade head          # creates all tables + the admin account
.venv/bin/uvicorn app.main:app --reload --port 8000
```

**Windows (PowerShell):**

```powershell
cd backend
py -3.13 -m venv .venv
.venv\Scripts\python -m pip install --upgrade pip
.venv\Scripts\pip install -r requirements.txt
.venv\Scripts\alembic upgrade head
.venv\Scripts\uvicorn app.main:app --reload --port 8000
```

Check it: open http://localhost:8000/docs (Swagger) and http://localhost:8000/api/health.

Leave this terminal running.

---

## 6. Frontend (React + Vite)

In a **new terminal**, from the repo root:

```bash
cd frontend
npm install --legacy-peer-deps
npm run dev
```

(Same commands on Windows.) Open **http://localhost:5173**.

---

## 7. Accounts and demo data

The migration in step 5 creates an **admin** account:

| Email | Password | Role |
|---|---|---|
| `sumit@test.com` | `123456` | ADMIN — sees **Data engineering** and **Admin · KYC review** |

Optional demo data (another terminal, from the repo root):

**macOS / Linux / WSL2:**

```bash
cd backend
PYTHONPATH=. .venv/bin/python scripts/seed_demo_data.py --users 10 --ledger-per-user 50 --connections 30   # quick (seconds)
PYTHONPATH=. .venv/bin/python scripts/seed_demo_data.py                                                   # full: ~1,000 users, years of ledger history (several minutes)
PYTHONPATH=. .venv/bin/python scripts/seed_support_staff.py                                               # Care portal staff
```

**Windows (PowerShell):**

```powershell
cd backend
$env:PYTHONPATH="."
.venv\Scripts\python scripts\seed_demo_data.py --users 10 --ledger-per-user 50 --connections 30
.venv\Scripts\python scripts\seed_support_staff.py
```

Seeded customers log in with `user0001@finvault.seed` … and password `password123456`; support staff use `owner@finvault.support` / `manager01@…` / `agent01@…` with `SupportPass123!`.

✅ **Level 1 done** — the app runs. Continue for the data platform.

---

## 8. Data pipeline environment (PySpark, Delta Lake, dbt)

macOS / Linux / WSL2 only (on Windows, do this inside WSL2). From the repo root:

```bash
python3.13 -m venv .venv
.venv/bin/pip install --upgrade pip
.venv/bin/pip install -r data-pipeline/requirements.txt
```

> If that fails with `CERTIFICATE_VERIFY_FAILED` (python.org Python on macOS), install with the certifi bundle:
> ```bash
> .venv/bin/pip install certifi
> SSL_CERT_FILE="$(.venv/bin/python -m certifi)" .venv/bin/pip install -r data-pipeline/requirements.txt
> ```

Check Spark + Delta work (runs the pipeline test suite, ~3 minutes):

```bash
cd data-pipeline && ../.venv/bin/python -m pytest -q && cd ..
```

Notes:
- The first Spark run downloads the Delta Lake jars (needs internet once).
- The first time the backend reads a Delta table it downloads DuckDB's `delta` extension (internet once).
- Tests never touch your real lake — they run in temp folders.

---

## 9. Airflow

Airflow gets its own environment (its dependencies are pinned and conflict with the app's):

```bash
python3.13 -m venv .venv-airflow
.venv-airflow/bin/pip install --upgrade pip
.venv-airflow/bin/pip install "apache-airflow==3.3.2" \
  --constraint "https://raw.githubusercontent.com/apache/airflow/constraints-3.3.2/constraints-3.13.txt"
```

Start it (keep the terminal open):

```bash
./scripts/airflow-standalone.sh        # or: make airflow
```

- UI: **http://127.0.0.1:8080**, user `admin`.
- Its password is generated on first start into `data-pipeline/airflow_home/simple_auth_manager_passwords.json.generated` (gitignored). The FinVault backend reads it automatically — you only need it for Airflow's own UI.
- The DAG `finvault_ledger_pipeline` starts **paused** (it's scheduled daily at 02:00 once switched on).

---

## 10. Run the whole stack with one command

Once steps 5, 6, 8 and 9 have been done once, you can start everything together (macOS / Linux / WSL2):

```bash
make dev        # = ./scripts/dev.sh : checks Postgres, then API (8000) + Airflow (8080) + frontend (5173)
```

`Ctrl+C` stops all of it. (Stop any backend/frontend you started by hand first — the ports must be free.)

---

## 11. Use the Data engineering page

1. Open http://localhost:5173, log in as `sumit@test.com` / `123456`.
2. In the sidebar, under **Admin**, open **Data engineering**.
3. In the Airflow panel, switch the DAG to **Scheduled** (or leave it paused and still trigger runs — paused DAGs queue them), tick **Full refresh** for the first run, and click **Run pipeline**.
4. Watch the task timeline: `export_from_postgres → quality_gate → run_bronze_silver_gold → dbt_build → dbt_docs` (about a minute on the demo data).
5. Explore the panels: lineage, data quality (with failing-row samples), warehouse (dbt lineage + model previews), pipeline runs, lakehouse (Delta versions), benchmarks.

---

## 12. Handy commands

All from the repo root (macOS / Linux / WSL2):

| Command | Does |
|---|---|
| `make dev` | start API + Airflow + frontend |
| `make airflow` | start Airflow only |
| `make quality` | run the 17 ledger data-quality checks in the terminal |
| `make dbt` | build the dbt warehouse (seeds → snapshot → models → tests) |
| `make dbt-docs` | dbt docs site on http://localhost:8081 |
| `make bench` | benchmarks on synthetic data (1M, 5M, 10M rows — ~20 min; `SCALES="1000000"` for a quick one) |
| `make bench-report` | rebuild `docs/benchmarks` from the latest run |
| `cd backend && PYTHONPATH=. .venv/bin/pytest` | backend tests (needs the `finvault_test` DB) |
| `make test-pipeline` | pipeline tests (PySpark + Delta) |

Without `make` (e.g. Windows native), each target's command is in the `Makefile` — copy it into your terminal.

---

## 13. Troubleshooting

| Symptom | Fix |
|---|---|
| `connection refused` on 5432 | Postgres isn't running: `brew services start postgresql@15` (macOS) · `sudo service postgresql start` (Linux/WSL) · Services app → *postgresql-x64-15* (Windows) |
| `password authentication failed for user "finvault_app"` | Re-run the `CREATE ROLE` step, or check `DATABASE_URL` in `backend/.env` |
| Frontend shows network errors | Backend not running on 8000, or `CORS_ORIGINS` in `backend/.env` doesn't include `http://localhost:5173` |
| Assistant / Help errors about OpenRouter or Ollama | Set `DECISION_PROVIDER=stub` and `LLM_PROVIDER=stub` in `backend/.env`, restart the backend |
| `PYTHON_VERSION_MISMATCH` from Spark | Handled in code (workers are pinned to the venv's Python); make sure you run with `.venv/bin/python` |
| `JAVA_HOME is not set` / Spark won't start | Install JDK 17 and set `JAVA_HOME` (step 1) |
| `CERTIFICATE_VERIFY_FAILED` during `pip install` | See the certifi note in step 8 |
| Airflow: `No such file or directory: 'airflow'` | Start it with `./scripts/airflow-standalone.sh` (it puts the venv on `PATH`) |
| Data engineering page says **Airflow offline** | Airflow isn't running — `make airflow` |
| Warehouse panel says *dbt hasn't been run yet* | Run the pipeline once (or `make dbt`) |
| Port already in use | Something else holds 8000/5173/8080: stop it, or change the port flag |
| Windows: Spark errors about `winutils` / `HADOOP_HOME` | Run the data platform inside WSL2 (Route A) |
| `make: command not found` (Windows) | Use WSL2, or copy commands from the `Makefile` |

---

## 14. Stopping and resetting

- Stop servers with `Ctrl+C` in their terminals.
- Stop Postgres: `brew services stop postgresql@15` (macOS) · `sudo service postgresql stop` (Linux/WSL).
- Rebuild the lake from scratch: trigger a run with **Full refresh** (the lake is fully derived from Postgres; `data-pipeline/local/output` can be deleted safely).
- Benchmark data lives in `data-pipeline/local/bench` (gitignored, can be deleted).
