# Phase 9 — Transfers & Expenses

**Status:** Complete  
**Depends on:** Phase 8 (complete)

## Objective

P2P wallet transfers between registered users and external expense payments, both writing
ledger entries atomically with KYC gating.

## Database

Migration `006_payments`:

- **`payments`** — external expense records (`receiver_name`, `amount`, `purpose`, `category`, `status`)
- Transfers do not use a separate table — dual `ledger_entries` rows share a `reference_id`

## Backend APIs

| Method | Path | Auth | Description |
|---|---|---|---|
| POST | `/api/transfer/send` | User | P2P transfer by `receiver_email` + `amount` |
| POST | `/api/payments/initiate` | User | Record external expense; debits wallet |
| GET | `/api/payments/my` | User | List user's expense payments |

### Transfer rules

- Sender and receiver must both have `kyc_status == APPROVED`
- Cannot transfer to yourself
- Both wallets locked in deterministic order (`user_id`) to avoid deadlocks
- Creates paired ledger entries: `TRANSFER_OUT` (debit) + `TRANSFER_IN` (credit)
- Shared `reference_id` links the two entries

### Payment rules

- KYC required before payment
- Creates `payments` row with `status = SUCCESS`
- Debits wallet via ledger `EXPENSE` entry with `reference_id = payment.id`

## Frontend

- **Send Money** (`/send-money`) — transfer form with balance display
- **Expenses** (`/expenses`) — record payment form + payment history table

## Next

Phase 11 — Modern frontend redesign. Say **"start Phase 11"**.
