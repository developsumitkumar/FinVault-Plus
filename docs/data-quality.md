# Data Quality (Phase 2)

This document describes how bulk transaction ingestion validates, classifies, and reports data quality.

## Flow

```text
CSV upload / pasted tabular data
        ↓
Schema validation (required columns present)
        ↓
Per-row validation (types, required fields, business rules)
        ↓
Duplicate detection (within file + against existing transactions)
        ↓
Preview response (counts + error samples)
        ↓
User confirmation
        ↓
Insert accepted rows into PostgreSQL (source = bulk_import)
        ↓
Persist rejected rows in import_errors
```

## Input format

### Required columns

| Column | Type | Notes |
|---|---|---|
| `transaction_date` | date | `YYYY-MM-DD`, `MM/DD/YYYY`, or `DD/MM/YYYY` |
| `amount` | decimal | Non-zero; sign normalized using `transaction_type` |
| `transaction_type` | enum | `debit` or `credit` |

### Optional columns

| Column | Type | Notes |
|---|---|---|
| `description` | text | Free-text note |
| `merchant` | text | Used for duplicate detection when description is empty |

The target account is selected in the UI/API form (`account_id`), not in the CSV.

## Validation layers

### 1. Schema validation

Fails the entire file if required columns are missing. Creates an `import_batches` record with
`status = failed` and an `import_errors` row with `error_type = schema_mismatch`.

### 2. Row validation

Each data row is classified independently:

| Result | Meaning |
|---|---|
| **valid** | Ready to import after confirmation |
| **invalid** | Missing values, bad types, or business-rule violation |
| **duplicate** | Matches another row in the file or an existing transaction |

Invalid and duplicate rows are stored in `import_errors` with a reason — they are never silently dropped.

### 3. Business rules

- `debit` rows must end up with a **negative** amount
- `credit` rows must end up with a **positive** amount
- Amount cannot be zero
- Unsigned amounts in the file are normalized using `transaction_type`

### 4. Duplicate detection

Business key:

```text
(account_id, transaction_date, amount, counterparty)
```

Where `counterparty` is `description` if present, otherwise `merchant` (case-insensitive).

Duplicates are checked:

1. Within the uploaded file (second occurrence is rejected)
2. Against existing transactions for the same account in PostgreSQL

## API endpoints

| Method | Path | Purpose |
|---|---|---|
| POST | `/imports/preview` | Parse + validate; returns preview |
| POST | `/imports/{batch_id}/confirm` | Import validated rows |
| GET | `/imports/{batch_id}` | Batch summary + errors |
| GET | `/imports` | List recent import batches |

## Data model

- `import_batches` — one record per upload attempt, with row counts and status
- `import_errors` — rejected rows with `raw_data` preserved as JSON
- `transactions.import_batch_id` — links accepted rows back to their batch

`import_batches.validated_rows` temporarily caches accepted rows between preview and confirm.

## Example preview result

```text
Total rows:      100
Valid rows:       94
Invalid rows:      4
Duplicates:        2
```

## Sample CSV

```csv
transaction_date,amount,transaction_type,description,merchant
2026-01-15,-25.00,debit,Coffee,Starbucks
2026-01-16,1500.00,credit,Paycheck,
2026-01-17,12.50,debit,Groceries,Whole Foods
```

## What's deferred

- JSON/Parquet bulk formats
- Merchant entity resolution (Silver-layer concern)
- Async/background processing for very large files
- Paste-only validation without `import_batches` persistence for schema failures beyond the failed batch record
