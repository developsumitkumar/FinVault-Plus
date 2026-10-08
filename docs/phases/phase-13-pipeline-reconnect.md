# Phase 13 — Pipeline Reconnect

Reconnect the wallet **ledger** (operational source of truth) to the medallion pipeline
alongside legacy bank-style **transactions**.

## Flow

```text
PostgreSQL
  ├── ledger_entries (+ users/wallets dims)  →  Bronze/ledger_entries
  └── transactions (CSV import)              →  Bronze/transactions
        ↓ PySpark
Silver (cleaned ledger events + legacy transactions)
        ↓ PySpark
Gold (ledger_monthly_spending, ledger_category_spending, …)
        ↓
Analytics UI (Phase 12 APIs read Postgres live; Gold for batch/scale)
```

## Bronze export

`data-pipeline/local/export_operational.py` now exports:

| Dataset | Bronze path | Source tables |
|---|---|---|
| Ledger entries | `bronze/ledger_entries/ledger_entries.csv` | `ledger_entries` ⋈ `users` ⋈ `wallets` |
| Users (dim) | `bronze/users/users.csv` | `users` |
| Wallets (dim) | `bronze/wallets/wallets.csv` | `wallets` |
| Transactions (legacy) | `bronze/transactions/transactions.csv` | `transactions` ⋈ `accounts` |

```bash
cd data-pipeline
source .venv/bin/activate
python -m local.export_operational          # export all
python -m local.run_pipeline --export       # export + transform both paths
python -m local.run_pipeline --ledger-only --export
```

## Ledger Silver transforms

`common/transforms/ledger_silver.py`:

- Normalizes `entry_type` (`DEBIT` / `CREDIT`) and `event_type`
- Uppercases categories (`UNCATEGORIZED` fallback)
- Computes `spend_amount` for debits (used by Gold aggregates)
- Deduplicates on `ledger_entry_id`

Supported event types: `OPENING_BALANCE`, `TRANSFER_OUT`, `TRANSFER_IN`, `EXPENSE`,
`SPLIT_DEDUCT`, `SPLIT_BILL`.

## Ledger Gold tables

| Table | Mirrors |
|---|---|
| `ledger_monthly_spending` | `GET /api/ledger/monthly` |
| `ledger_category_spending` | `GET /api/ledger/category` |
| `ledger_event_summary` | Event-type breakdown |
| `ledger_user_summary` | Per-user totals |

Outputs land under `data-pipeline/local/output/gold/ledger_*`.

## Verify end-to-end cycle

**Fixture (no database):**

```bash
python -m local.verify_ledger_cycle --fixture
```

**Live Postgres:**

```bash
# App running with wallet activity in PostgreSQL
python -m local.verify_ledger_cycle --export
```

**Databricks / Delta (local):**

```bash
python -m databricks.run_medallion --run-id manual-run --local-delta --ledger
```

## Tests

```bash
cd data-pipeline && pytest -q
```

17 pipeline tests including ledger transforms, local parquet run, Delta medallion, and
fixture cycle verification.

## Deliverables

- [x] Export `ledger_entries` + `users` / `wallets` dims to Bronze
- [x] Ledger Silver/Gold transforms for wallet event types
- [x] `run_pipeline` runs transactions + ledger paths (flags: `--ledger-only`, `--transactions-only`)
- [x] Databricks Delta path for ledger (`--ledger` on `run_medallion.py`)
- [x] Azure uploader skips missing Gold datasets; ledger prefixes supported
- [x] `verify_ledger_cycle.py` fixture + live modes
- [x] pytest coverage (17 tests)

---

Phase 14 — Deploy & polish. Say **"start Phase 14"** (complete).
