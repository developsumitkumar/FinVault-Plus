# Phase 7 — Wallet & Ledger

**Status:** Complete  
**Depends on:** Phase 6 (complete)

## Objective

Introduce wallet balance and immutable ledger entries as the source of truth for
in-app money movement. Expose passbook and summary APIs matching the Java FinVault.

## Database

Migration `004_wallets_ledger`:

- **`wallets`** — one per user (`balance`, `currency`)
- **`ledger_entries`** — immutable audit trail (`entry_type`, `event_type`, `amount`, `balance_after`)

Backfills wallets for existing dev/admin users with $10,000 opening balance.

## Backend

| Module | Purpose |
|---|---|
| `LedgerService` | Open wallet, debit/credit (for future phases), passbook, summary |
| `GET /api/ledger/passbook` | Filterable ledger history |
| `GET /api/ledger/summary` | Balance, total spent, total income |
| `GET /api/ledger/wallet` | Wallet details |

Registration now creates a wallet with **$10,000** opening balance + `OPENING_BALANCE` credit entry (matches Java).

## Frontend (minimal)

- **Wallet** page — balance summary + passbook table

## Completion Criteria

- [x] Wallets and ledger tables migrated
- [x] Passbook and summary APIs
- [x] Debit/credit helpers ready for Phase 9 transfers/expenses
- [x] Register creates wallet automatically
- [x] Tests passing

## Next

Phase 8 — KYC & admin. Say **"start Phase 8"**.
