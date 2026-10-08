# Azure Storage Integration (Phase 4)

This document describes how FinVault pushes local pipeline artifacts to **Azure Data Lake Storage Gen2** while keeping local development fully supported.

## LOCAL MODE vs AZURE MODE

| | LOCAL MODE | AZURE MODE |
|---|---|---|
| **Config** | `STORAGE_MODE=local` (default) | `STORAGE_MODE=azure` + credentials |
| **Pipeline output** | `data-pipeline/local/output/` | Same local output **plus** upload to ADLS Gen2 |
| **Azure account required?** | No | Yes |
| **PySpark execution** | Local machine (Phase 3) | Still local in Phase 4 — cloud Spark is Phase 5 |
| **Purpose** | Day-to-day dev, tests, portfolio demo without cloud cost | Prove cloud storage integration for interviews |

**Important:** Phase 4 does not run Databricks or cloud Spark. It uploads the same Bronze/Silver/Gold files that Phase 3 produces locally. Phase 5 will consume them in Azure.

## Architecture

```text
PostgreSQL
    ↓ export
Local Bronze/Silver/Gold  (always written)
    ↓ optional upload (STORAGE_MODE=azure)
ADLS Gen2
  bronze/transactions/{run_id}/
  silver/transactions/{run_id}/
  gold/{dataset}/{run_id}/
```

## Configuration

Add to the repo root `.env` (see `.env.example`):

```bash
# local (default) — no Azure needed
STORAGE_MODE=local

# azure — required for upload
STORAGE_MODE=azure
AZURE_STORAGE_ACCOUNT_NAME=your-account
AZURE_STORAGE_ACCOUNT_KEY=your-key
# or AZURE_STORAGE_CONNECTION_STRING=...

AZURE_STORAGE_CONTAINER_BRONZE=bronze
AZURE_STORAGE_CONTAINER_SILVER=silver
AZURE_STORAGE_CONTAINER_GOLD=gold
```

Never commit keys or connection strings. `.env` is gitignored.

## Naming conventions

| Layer | Container | Path pattern |
|---|---|---|
| Bronze | `bronze` | `transactions/{run_id}/transactions.csv` |
| Silver | `silver` | `transactions/{run_id}/part-*.parquet` |
| Gold | `gold` | `{dataset}/{run_id}/part-*.parquet` |
| Manifest | `bronze` | `_manifests/{run_id}.json` |

`run_id` format: `YYYY-MM-DDTHH-MM-SSZ` (UTC).

## Run locally (no Azure)

```bash
cd data-pipeline
source .venv/bin/activate
python -m local.run_pipeline --export
```

Outputs remain under `data-pipeline/local/output/`.

## Run pipeline and upload to Azure

1. Configure `STORAGE_MODE=azure` and credentials in `.env`.
2. Run:

```bash
export JAVA_HOME=$(/usr/libexec/java_home -v 19)

cd data-pipeline
source .venv/bin/activate
pip install -r requirements.txt
python -m local.run_pipeline --export --upload-azure
```

Or upload after a local run:

```bash
python -m local.upload_to_azure
```

## Verify upload

- Azure Portal → Storage account → Containers → browse `bronze`, `silver`, `gold`
- Open `bronze/_manifests/{run_id}.json` for file list and row counts

## Error handling

- Missing credentials → clear error at startup (`StorageConfigError`)
- Missing local artifacts → `AzureUploadError` with path
- Azure API failures → wrapped `AzureUploadError` with underlying message

## Code layout

```text
data-pipeline/common/storage/
  settings.py    # STORAGE_MODE + credential loading
  paths.py       # run_id and remote path conventions
  uploader.py    # ADLS Gen2 upload + manifest
local/
  upload_to_azure.py   # standalone upload CLI
```

## What's deferred

- Databricks jobs reading from ADLS (Phase 5)
- Delta Lake table format in cloud
- Managed identity / Key Vault (document as production improvement)
- Automated scheduled uploads
