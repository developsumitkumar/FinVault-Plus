# Azure Databricks Pipeline (Phase 5)

This document describes how FinVault runs the **medallion pipeline** on Azure Databricks, reading from ADLS Gen2 and writing **Delta Lake** tables.

## Role of Databricks

| Concern | Phase 3 (local) | Phase 4 (upload) | Phase 5 (Databricks) |
|---|---|---|---|
| PySpark transforms | Local machine | — | Databricks cluster |
| Storage | Local disk | ADLS Gen2 files | ADLS Gen2 **Delta tables** |
| Orchestration | Manual CLI | Manual upload | Notebook / Job |
| Purpose | Prove transform logic | Prove cloud storage | Prove cloud **processing** at scale |

Databricks is the managed Spark runtime. The **same transform functions** in `data-pipeline/common/transforms/` run locally (Phase 3) and on Databricks (Phase 5).

## End-to-end flow

```text
PostgreSQL
    ↓ Phase 3 export + transforms (local)
    ↓ Phase 4 upload
ADLS Gen2 Bronze CSV   bronze/transactions/{run_id}/transactions.csv
    ↓ Phase 5 Databricks notebook/job
ADLS Gen2 Delta tables
    silver/transactions
    gold/monthly_spending
    gold/category_spending
    gold/merchant_summary
    gold/daily_transaction_metrics
    gold/account_summary
```

## Processing mode (v1)

**Full refresh (`overwrite`)** per run — the entire Silver and Gold tables are rebuilt from the Bronze snapshot identified by `run_id`.

Incremental processing is **not** implemented in v1. A production evolution would:

- Track `transaction_id` watermarks in a control table
- Use Delta `MERGE` for idempotent upserts
- Document retry semantics per batch

This is intentionally deferred so the portfolio stays explainable without unverified complexity.

## Delta table schemas

### Silver: `transactions`

| Column | Type | Notes |
|---|---|---|
| transaction_id | string | Business key |
| account_id | string | |
| account_name | string | |
| amount | decimal(14,2) | Signed |
| currency | string | |
| transaction_type | string | debit / credit |
| transaction_date | date | |
| year_month | string | yyyy-MM |
| description | string | |
| normalized_merchant | string | Cleaned for analytics |
| category_name | string | Defaults to Uncategorized |
| source | string | manual / bulk_import |
| import_batch_id | string | nullable |
| spend_amount | decimal(14,2) | Positive for debits |
| created_at | timestamp | |
| processed_at | timestamp | Pipeline stamp |

### Gold tables

| Table | Grain | Key metrics |
|---|---|---|
| `monthly_spending` | account + month | total_spending, debit_count, total_income |
| `category_spending` | category + month | total_spending, transaction_count |
| `merchant_summary` | merchant | total_spending, transaction_count, avg_transaction_amount |
| `daily_transaction_metrics` | day | transaction_count, total_spending, total_income, avg |
| `account_summary` | account | transaction_count, total_spending, total_income |

## Configuration

Add to `.env` (never commit secrets):

```bash
DATABRICKS_HOST=https://<workspace>.azuredatabricks.net
DATABRICKS_TOKEN=<personal-access-token>
DATABRICKS_CLUSTER_ID=<cluster-id>

AZURE_STORAGE_ACCOUNT_NAME=<account>
AZURE_STORAGE_ACCOUNT_KEY=<key>
AZURE_STORAGE_CONTAINER_BRONZE=bronze
AZURE_STORAGE_CONTAINER_SILVER=silver
AZURE_STORAGE_CONTAINER_GOLD=gold
```

On Databricks, prefer **secret scopes** over plain env vars (see notebook).

## Run on Databricks

1. Attach repo to workspace.
2. Create secrets scope `finvault` with storage credentials.
3. Open `data-pipeline/databricks/notebooks/01_medallion_pipeline.py`.
4. Set `run_id` to match Phase 4 manifest (e.g. `2026-01-15T14-30-00Z`).
5. Run all cells or schedule via `jobs/medallion_pipeline.json`.

## Local verification (no Databricks account)

Validates Delta writes and shared transforms without cloud compute:

```bash
export JAVA_HOME=$(/usr/libexec/java_home -v 19)
cd data-pipeline
source .venv/bin/activate
python -m local.run_pipeline --export
python -m databricks.run_medallion --run-id local-dev --local-delta
```

Inspect: `local/output/delta/silver/transactions/`

## What Phase 5 does not claim

- Production SLA, autoscaling policies, or cost optimization
- Incremental MERGE / CDC from PostgreSQL
- Unity Catalog governance (can be added as a talking point)
- Verified execution on your specific Databricks workspace unless you run the notebook yourself

## Code layout

```text
data-pipeline/common/databricks/
  settings.py         # env + credential loading
  paths.py            # abfss:// and local Delta paths
  spark_session.py    # Spark + Delta + ADLS config
  pipeline.py         # read Bronze → write Silver/Gold Delta

data-pipeline/databricks/
  notebooks/01_medallion_pipeline.py
  jobs/medallion_pipeline.json
  run_medallion.py
```

## What's next (Phase 6)

Serve Gold Delta datasets through FastAPI and the React analytics dashboard.
