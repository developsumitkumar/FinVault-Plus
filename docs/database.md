# Database Design

Engine: **PostgreSQL 15+**. Migrations managed with **Alembic** (`backend/alembic/`).

> **Schema evolution:** Sections below describe the **Phase 1** schema (accounts, transactions,
> CSV import). Phases 6–10 add wallet platform tables (`wallets`, `ledger_entries`, `kyc_submissions`,
> `connections`, `split_bills`, `payments`) per `docs/reference/java-port-reference.md`. Phase 7 documents
> the final merged schema.

## 1. Entity Overview

```
users ──< accounts ──< transactions >── categories
                            │
                            └──> merchants
                            │
                            └──> import_batches ──< import_errors
```

## 2. Tables

### `users`
| Column | Type | Notes |
|---|---|---|
| id | UUID, PK | |
| email | citext, UNIQUE NOT NULL | case-insensitive uniqueness |
| password_hash | text NOT NULL | never store plaintext |
| full_name | text NOT NULL | |
| role | text NOT NULL | `USER` or `ADMIN` (Phase 6) |
| kyc_status | text NOT NULL | `NONE`, `PENDING`, `APPROVED`, `REJECTED` (Phase 6) |
| phone | text, NULLABLE | Phase 6 |
| avatar_url | text, NULLABLE | Phase 6 |
| created_at | timestamptz NOT NULL DEFAULT now() | |
| updated_at | timestamptz NOT NULL DEFAULT now() | |

### `kyc_submissions` (Phase 8)
| Column | Type | Notes |
|---|---|---|
| id | UUID, PK | |
| user_id | UUID, FK, UNIQUE | one submission record per user |
| document_type | text | PASSPORT, DRIVERS_LICENSE, NATIONAL_ID |
| document_number | text | |
| document_file_name | text | original upload filename |
| document_file_path | text | local path under `uploads/kyc/` |
| document_mime_type | text | |
| status | text | PENDING, APPROVED, REJECTED |
| admin_remarks | text, nullable | set on reject |
| created_at / reviewed_at | timestamptz | |

### `wallets` (Phase 7)
| Column | Type | Notes |
|---|---|---|
| id | UUID, PK | |
| user_id | UUID, FK → users.id, UNIQUE | one wallet per user |
| balance | numeric(14,2) | current balance |
| currency | char(3) | default USD |
| created_at / updated_at | timestamptz | |

### `ledger_entries` (Phase 7)
| Column | Type | Notes |
|---|---|---|
| id | UUID, PK | |
| user_id | UUID, FK → users.id | |
| reference_id | UUID, nullable | links to payment/transfer/split later |
| entry_type | text | `DEBIT` or `CREDIT` |
| event_type | text | e.g. `OPENING_BALANCE`, `EXPENSE`, `TRANSFER_OUT` |
| amount | numeric(14,2) | always positive |
| category | text, nullable | |
| description | text | |
| balance_after | numeric(14,2) | running balance snapshot |
| created_at | timestamptz | immutable |

### `accounts`
| Column | Type | Notes |
|---|---|---|
| id | UUID, PK | |
| user_id | UUID, FK → users.id NOT NULL | ON DELETE CASCADE |
| account_name | text NOT NULL | e.g. "Chase Checking" |
| account_type | text NOT NULL | CHECK IN ('checking','savings','credit_card','cash','investment') |
| currency | char(3) NOT NULL DEFAULT 'USD' | ISO 4217 code |
| opening_balance | numeric(14,2) NOT NULL DEFAULT 0 | |
| created_at | timestamptz NOT NULL DEFAULT now() | |
| updated_at | timestamptz NOT NULL DEFAULT now() | |

Index: `(user_id)` — every account list query filters by owner.

### `categories`
| Column | Type | Notes |
|---|---|---|
| id | UUID, PK | |
| name | text NOT NULL | e.g. "Shopping", "Groceries" |
| parent_category_id | UUID, FK → categories.id, NULLABLE | supports simple category hierarchy |
| is_system | boolean NOT NULL DEFAULT false | distinguishes seeded defaults from user-created |

Constraint: UNIQUE `(name, parent_category_id)` to prevent duplicate siblings.

### `merchants`
| Column | Type | Notes |
|---|---|---|
| id | UUID, PK | |
| raw_name | text NOT NULL | as it appeared in source data, e.g. "AMAZON.COM*4F92J" |
| normalized_name | text NOT NULL | cleaned form, e.g. "Amazon" — this is the field Silver-layer standardization is responsible for keeping consistent |
| default_category_id | UUID, FK → categories.id, NULLABLE | |

Index: `(normalized_name)` — used heavily by merchant-based analytics and dedup matching.

### `transactions`
| Column | Type | Notes |
|---|---|---|
| id | UUID, PK | |
| account_id | UUID, FK → accounts.id NOT NULL | ON DELETE CASCADE |
| merchant_id | UUID, FK → merchants.id, NULLABLE | nullable because manual entries may skip merchant matching initially |
| category_id | UUID, FK → categories.id, NULLABLE | |
| amount | numeric(14,2) NOT NULL | signed: negative = money out, positive = money in |
| currency | char(3) NOT NULL | denormalized from account at write time, since historical transactions shouldn't silently change if account currency is ever edited |
| transaction_type | text NOT NULL | CHECK IN ('debit','credit') — paired with signed amount for query ergonomics (e.g. "sum all debits") |
| transaction_date | date NOT NULL | business date of the transaction, distinct from `created_at` |
| description | text | free text, as entered or as imported |
| source | text NOT NULL | CHECK IN ('manual','bulk_import') |
| import_batch_id | UUID, FK → import_batches.id, NULLABLE | set only when source = 'bulk_import' |
| created_at | timestamptz NOT NULL DEFAULT now() | |
| updated_at | timestamptz NOT NULL DEFAULT now() | |

Indexes:
- `(account_id, transaction_date)` — primary access pattern (statement views, monthly spend)
- `(merchant_id)`
- `(category_id)`
- `(import_batch_id)`

This table is the row-level source for the Bronze extraction — a Bronze snapshot is, at its
simplest, "all transactions as of extraction time, unmodified."

### `import_batches`
Tracks every bulk ingestion attempt as a first-class record, so "how do you handle bad data?"
(Section 22 of the spec) has a real, queryable answer instead of only a log line.

| Column | Type | Notes |
|---|---|---|
| id | UUID, PK | |
| user_id | UUID, FK → users.id NOT NULL | |
| source_filename | text NOT NULL | |
| status | text NOT NULL | CHECK IN ('pending','validated','completed','failed') |
| total_rows | integer NOT NULL DEFAULT 0 | |
| valid_rows | integer NOT NULL DEFAULT 0 | |
| invalid_rows | integer NOT NULL DEFAULT 0 | |
| duplicate_rows | integer NOT NULL DEFAULT 0 | |
| created_at | timestamptz NOT NULL DEFAULT now() | |
| completed_at | timestamptz, NULLABLE | |

### `import_errors`
| Column | Type | Notes |
|---|---|---|
| id | UUID, PK | |
| import_batch_id | UUID, FK → import_batches.id NOT NULL | ON DELETE CASCADE |
| row_number | integer NOT NULL | 1-indexed position in the source file |
| raw_data | jsonb NOT NULL | the offending row, preserved as-is for debugging |
| error_type | text NOT NULL | CHECK IN ('missing_value','invalid_type','duplicate','schema_mismatch','other') |
| error_message | text NOT NULL | human-readable reason |
| created_at | timestamptz NOT NULL DEFAULT now() | |

Index: `(import_batch_id)`.

## 3. Design Notes / Rationale

- **UUID primary keys** rather than serial integers: avoids leaking row counts, and matches
  well with the eventual distributed ingestion path (batch-generated IDs don't collide).
- **`import_batches` / `import_errors` are the deliberate answer to "how do you handle bad
  data / duplicates?"** — rejected rows are persisted with a reason, not just counted.
- **`currency` is denormalized onto `transactions`** intentionally: historical transaction
  records must not change meaning if an account's currency setting is edited later.
- **No `budgets`, `goals`, `notifications`, or similar tables** from the original FinVault app
  are carried over in Phase 0 — they don't serve the data-engineering narrative and would only
  add operational surface area without a corresponding pipeline benefit. They can be added later
  if a phase genuinely needs them.
- **Soft deletes are not included** in this proposal (no `deleted_at` columns). Given the
  pipeline reads from the operational store into Bronze, hard deletes are simpler for v1;
  this is called out as an explicit trade-off, not an oversight — worth revisiting once
  incremental/CDC-style extraction is discussed in Phase 5.

## 4. What's Deliberately Deferred

- Multi-currency conversion / FX rates table.
- Row-level audit history (e.g. `transaction_history` for edits) — could be added in Phase 7
  if time allows, framed as a "what would you change for production" talking point.
- Partitioning strategy for `transactions` — not needed at portfolio data volumes; the
  partitioning conversation instead happens in the Bronze/Silver Parquet/Delta layout (Phase 3+),
  where it's actually relevant to Spark's read performance.
