# Phase 12 — Analytics Layer

**Status:** Complete  
**Depends on:** Phase 11 (complete)

## Objective

Ledger analytics APIs (Java parity) and a Tremor-powered analytics dashboard for category
and monthly spending insights.

## Backend APIs

| Method | Path | Description |
|---|---|---|
| GET | `/api/ledger/category` | Debit totals grouped by category (`UNCATEGORIZED` fallback) |
| GET | `/api/ledger/monthly` | Debit totals grouped by month (`YYYY-MM`) |

Both endpoints use PostgreSQL `GROUP BY` aggregations on `ledger_entries` (debits only).

## Frontend

- **`/analytics`** — KPI cards + Tremor DonutChart (categories) + BarChart (monthly trend)
- **Dashboard** — mini monthly BarChart linked to full analytics page
- Sidebar nav entry: **Analytics**

## Dependencies added

- `@tremor/react` (charts)
- Tailwind `@source` for Tremor component classes

## Next

Phase 14 — Deploy & polish. Say **"start Phase 14"**.
