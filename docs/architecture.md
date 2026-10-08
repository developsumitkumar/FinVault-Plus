# Architecture

## 1. What FinVault Is

FinVault is a **full-stack Python fintech wallet** (port of the Java/Spring Boot app) **plus** a
**medallion data pipeline**. The app generates financial events; the pipeline processes them at scale.

```
┌─────────────────────────────────────────────────────────────┐
│  REACT FRONTEND (modern UI — see docs/design/)              │
│  Auth · Wallet · KYC · Transfers · Splits · Analytics       │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│  FASTAPI BACKEND (routes → services → repositories)        │
│  JWT auth · ledger writes · business rules                  │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│  POSTGRESQL — operational store + ledger                    │
│  users · wallets · ledger_entries · kyc · connections ·   │
│  split_bills · payments · transactions (CSV import)         │
└──────────────────────────┬──────────────────────────────────┘
                           │ export
                           ▼
┌─────────────────────────────────────────────────────────────┐
│  DATA PIPELINE (Phases 3–5, reconnect in Phase 13)          │
│  Bronze (raw) → Silver (cleaned) → Gold (aggregates)        │
│  Local PySpark · ADLS Gen2 · Databricks Delta               │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           ▼
                    Analytics dashboard
              (SQL real-time + Gold at scale)
```

See `PROJECT_PLAN.md` for phase order. Phases 0–5 built the pipeline shell; Phases 6–14 complete
the product and reconnect the cycle.

## 2. Data Architecture — Four Layers of Truth

| Layer | What it is | Where it lives |
|---|---|---|
| **Operational** | Live app state: wallets, ledger, KYC, connections, splits | PostgreSQL |
| **Raw / Bronze** | Immutable export snapshot for lineage and reprocessing | Local files → ADLS Gen2 |
| **Cleaned / Silver** | Validated, standardized, typed ledger events | Parquet/Delta |
| **Analytics / Gold** | Monthly spend, category trends, merchant summaries | Delta tables |

**Why medallion?** The app handles real-time money movement (ledger). Analytics at scale runs on
exported snapshots without hammering the operational DB. Gold tables are rebuildable from Silver.

## 3. Ledger Model (Phase 7+)

The Java app stored `balance` on the user document and wrote `LedgerEntry` rows for every event.
The Python port uses the same pattern, normalized for PostgreSQL:

- **`wallets`** — one per user, holds current `balance`
- **`ledger_entries`** — immutable audit trail; never updated or deleted
- Every transfer, expense, or split settlement writes ledger rows inside a **DB transaction**
  with row-level locking on the wallet

Passbook = ledger query. Analytics = aggregate ledger (or Gold tables for heavy workloads).

## 4. Local vs Cloud Execution

| Concern | Local (dev) | Cloud (portfolio demo) |
|---|---|---|
| App | FastAPI + Postgres on machine | Render / similar |
| Pipeline | `pyspark` + local Delta | Databricks + ADLS Gen2 |
| Purpose | Develop and test transforms | Prove cloud-scale processing |

Same transform code in `data-pipeline/common/transforms/` runs in both environments.

## 5. Technology Roles

| Technology | Role |
|---|---|
| PostgreSQL | ACID operational store, ledger integrity, relational constraints |
| FastAPI | Typed REST API, Pydantic validation, OpenAPI docs |
| React + Vite | Modern animated UI (Tailwind, Framer Motion — Phase 11) |
| PySpark / Spark | Distributed transforms on exported data |
| ADLS Gen2 | Cloud object storage for medallion layers |
| Databricks | Managed Spark + Delta Lake execution |
| Delta Lake | ACID tables on data lake, schema enforcement |
| Docker Compose | Reproducible local stack (Phase 14) |

## 6. Backend Layering

```
API / Routes            (FastAPI routers — request/response only)
      ↓
Service Layer            (business rules: KYC gates, ledger writes, split logic)
      ↓
Repository / Data Access  (SQLAlchemy queries)
      ↓
PostgreSQL
```

Rules:
- Route handlers never touch the database directly.
- Services never import FastAPI request/response objects.
- Ledger writes happen in services, inside transactions.
- Pydantic schemas ≠ SQLAlchemy models.

## 7. Java Port

Original app: [github.com/developsumitkumar/FinVault](https://github.com/developsumitkumar/FinVault)

API and behavior mapping: `docs/reference/java-port-reference.md`

## 8. Bulk CSV Import (Phase 2 — kept)

CSV import remains a first-class feature (bank statement ingestion). It writes to `transactions`
with quality tracking via `import_batches` / `import_errors`. Phase 13 exports wallet
`ledger_entries` (plus user/wallet dimensions) alongside legacy transactions into Bronze.
ledger events into Bronze.
