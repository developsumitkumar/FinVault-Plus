# Local PySpark Pipeline

This document describes the **Phase 3** local data pipeline. It is a development equivalent
of the cloud medallion architecture — not a substitute for Azure Data Lake Storage or Databricks.

## Flow

```text
PostgreSQL (operational)
        ↓ export_operational.py
Bronze CSV/Parquet  (raw snapshot + extracted_at)
        ↓ PySpark
Silver Parquet  (cleaned, standardized, deduplicated)
        ↓ PySpark
Gold Parquet    (analytics-ready aggregates)
```

## Local vs cloud

| Concern | Phase 3 (local) | Phase 4+ (cloud) |
|---|---|---|
| Storage | `data-pipeline/local/output/` on disk | Azure Data Lake Storage Gen2 |
| Compute | Local PySpark (`local[*]`) | Azure Databricks |
| Table format | Parquet | Delta Lake |
| Orchestration | Manual CLI | Databricks Jobs |

The transformation logic in `data-pipeline/common/` is written to be portable — the same
functions are intended to run on Databricks in Phase 5 with different I/O paths.

## Directory layout

```text
data-pipeline/
  common/
    schemas.py              # explicit Spark schemas
    transforms/
      bronze.py             # minimal metadata enrichment
      silver.py             # cleaning, dedup, normalization
      gold.py               # business aggregates
  local/
    export_operational.py   # PostgreSQL → Bronze CSV
    run_pipeline.py         # Bronze → Silver → Gold CLI
  tests/
```

Generated outputs (gitignored):

```text
data-pipeline/local/output/
  bronze/transactions/
  silver/transactions/
  gold/monthly_spending/
  gold/category_spending/
  gold/merchant_summary/
  gold/daily_transaction_metrics/
```

## Prerequisites

- **Java 17 or 19** (`java -version`) — Java 21+ may fail with Spark networking errors on macOS
- Python 3.11+ (3.12 recommended; 3.15 works with the pipeline venv)
- PostgreSQL running with FinVault data (Phases 1–2)
- Pipeline dependencies installed

If you have multiple Java versions installed, set `JAVA_HOME` before running:

```bash
export JAVA_HOME=$(/usr/libexec/java_home -v 19)
```

## Setup

```bash
cd data-pipeline
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

Ensure the root `.env` file has a valid `DATABASE_URL`.

## Run the pipeline

Export from PostgreSQL and run all transforms (legacy transactions **and** wallet ledger):

```bash
cd data-pipeline
source .venv/bin/activate
python -m local.run_pipeline --export
```

Ledger-only path:

```bash
python -m local.run_pipeline --ledger-only --export
```

Verify the ledger cycle without a database:

```bash
python -m local.verify_ledger_cycle --fixture
```

Or export and run separately:

```bash
python -m local.export_operational
python -m local.run_pipeline
```

Expected output (combined run):

```text
Pipeline completed:
  transactions_bronze: N rows
  transactions_silver: N rows
  ledger_bronze: N rows
  ledger_silver: N rows
  ledger_monthly_spending: ...
  ledger_category_spending: ...
  ...
```

## Inspect outputs

Parquet files can be inspected with PySpark or any Parquet viewer:

```bash
python -c "
from pyspark.sql import SparkSession
spark = SparkSession.builder.master('local[*]').getOrCreate()
spark.read.parquet('local/output/gold/monthly_spending').show()
spark.stop()
"
```

## Transformations demonstrated

### Bronze
- Read with **explicit schema** (no inference drift)
- Stamp pipeline metadata (`_pipeline_layer`, `_ingested_at`)

### Silver
- Filter invalid transaction types
- Type casting (`date`, `decimal`)
- `withColumn` for `normalized_merchant`, `year_month`, `spend_amount`
- Null handling (`coalesce`, default category)
- **Deduplication** on `(account_id, transaction_date, amount, normalized_merchant)`

### Gold — transactions (legacy CSV import)

| Dataset | Grain | Metrics |
|---|---|---|
| `monthly_spending` | account + month | total spending, debit count, income |
| `category_spending` | category + month | total spending, transaction count |
| `merchant_summary` | merchant | total spending, count, average |
| `daily_transaction_metrics` | day | count, spending, income, average |

### Bronze/Silver/Gold — wallet ledger (Phase 13)

| Layer | Path | Notes |
|---|---|---|
| Bronze | `bronze/ledger_entries/` | `ledger_entries` joined with `users` and `wallets` |
| Bronze dims | `bronze/users/`, `bronze/wallets/` | dimension snapshots |
| Silver | `silver/ledger_entries/` | normalized event types, `spend_amount` for debits |
| Gold | `gold/ledger_monthly_spending` | mirrors `GET /api/ledger/monthly` |
| Gold | `gold/ledger_category_spending` | mirrors `GET /api/ledger/category` |
| Gold | `gold/ledger_event_summary` | counts by `event_type` |
| Gold | `gold/ledger_user_summary` | per-user totals |

See [`docs/phases/phase-13-pipeline-reconnect.md`](./phases/phase-13-pipeline-reconnect.md).

## Tests

```bash
cd data-pipeline
source .venv/bin/activate
pytest
```

Tests cover Silver deduplication/normalization, Gold aggregations, ledger transforms, and
end-to-end pipeline runs against fixture data (17 tests).

## What's deferred to later phases

- Azure ADLS upload (Phase 4)
- Databricks execution (Phase 5)
- Delta Lake table format
- Serving Gold data through FastAPI/React (Phase 6)
- Scheduled orchestration
