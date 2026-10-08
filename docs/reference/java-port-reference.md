# Java → Python Port Reference

Maps the original [FinVault Java repo](https://github.com/developsumitkumar/FinVault) to this
Python/PostgreSQL implementation.

**Goal:** Feature and API parity. **Not** a line-by-line translation.

---

## Stack Mapping

| Java (original) | Python (this repo) |
|---|---|
| Spring Boot 3 | FastAPI |
| Spring Security + JWT | `python-jose` / `passlib` + FastAPI dependencies |
| MongoDB Atlas | PostgreSQL 15+ |
| Spring Data MongoDB | SQLAlchemy + Alembic |
| Embedded `SplitBillMember` in document | Normalized `split_bill_members` table |
| `User.balance` on document | `wallets.balance` + `ledger_entries` (source of truth) |
| React + MUI (pastel green) | React + Vite + **new modern UI** (see `docs/design/`) |

---

## Model Mapping

### User

| Java (`users` collection) | PostgreSQL |
|---|---|
| `id` | `users.id` (UUID) |
| `name` | `users.full_name` |
| `email` | `users.email` (citext, unique) |
| `password` | `users.password_hash` |
| `role` | `users.role` (`USER` / `ADMIN`) |
| `kycStatus` | `users.kyc_status` (`NONE` / `PENDING` / `APPROVED` / `REJECTED`) |
| `balance` | **Moved to `wallets.balance`** — do not store on user row |
| `phone` | `users.phone` (nullable) |
| `avatarUrl` | `users.avatar_url` (nullable) |
| `createdAt` | `users.created_at` |

### KYC

| Java (`kyc_docs`) | PostgreSQL `kyc_submissions` |
|---|---|
| `userId` | `user_id` FK |
| `documentType` | `document_type` |
| `documentNumber` | `document_number` |
| `status` | `status` |
| `remarks` | `admin_remarks` |

### Ledger Entry

| Java (`ledger_entries`) | PostgreSQL `ledger_entries` |
|---|---|
| `userId` | `user_id` FK |
| `paymentId` | `reference_id` (polymorphic: payment, transfer, split) |
| `type` | `entry_type` (`DEBIT` / `CREDIT`) |
| `amount` | `amount` (numeric, always positive) |
| `category` | `category` (text or FK to `categories`) |
| `description` | `description` |
| `balanceAfterTransaction` | `balance_after` |
| `createdAt` | `created_at` |

Add `event_type`: `TRANSFER_OUT`, `TRANSFER_IN`, `EXPENSE`, `SPLIT_DEDUCT`, `SPLIT_SETTLE`, etc.

### Payment (external expense)

| Java (`payments`) | PostgreSQL `payments` |
|---|---|
| `userId` | `user_id` |
| `receiverName` | `receiver_name` |
| `amount` | `amount` |
| `purpose` | `purpose` |
| `category` | `category` |
| `status` | `status` |
| `createdAt` | `created_at` |

Creates a ledger DEBIT entry and deducts wallet.

### Connection

| Java (`connections`) | PostgreSQL `connections` |
|---|---|
| `requesterUserId` | `requester_id` |
| `receiverUserId` | `receiver_id` |
| `status` | `status` (`PENDING` / `ACCEPTED` / `REJECTED`) |
| `note` | `note` |
| `createdAt` | `created_at` |

### Split Bill

| Java (`split_bills` + embedded members) | PostgreSQL |
|---|---|
| Bill header | `split_bills` table |
| Members array | `split_bill_members` table (normalized) |
| `createdByUserId` | `created_by_user_id` |
| `totalAmount` | `total_amount` |
| `perPersonAmount` | `per_person_amount` |
| `status` | `status` (`ACTIVE` / `SETTLED`) |
| Member `settlementStatus` | `split_bill_members.settlement_status` |

---

## API Mapping

Paths below use Java's `/api` prefix. Python FastAPI uses the same paths (no `/api` prefix unless
we add one — **decision: match Java** with `/api` prefix for frontend portability).

| Java endpoint | Python phase | Notes |
|---|---|---|
| `POST /api/auth/register` | 6 | |
| `POST /api/auth/login` | 6 | Returns JWT |
| `GET /api/user/profile` | 6 | |
| `PUT /api/user/profile` | 6 | |
| `POST /api/kyc/submit` | 8 | |
| `GET /api/kyc/status` | 8 | |
| `GET /api/admin/kyc/pending` | 8 | Admin only |
| `POST /api/admin/kyc/approve` | 8 | |
| `POST /api/admin/kyc/reject` | 8 | |
| `POST /api/connections/request` | 10 | |
| `POST /api/connections/{id}/accept` | 10 | |
| `POST /api/connections/{id}/reject` | 10 | |
| `POST /api/connections/{id}/withdraw` | 10 | |
| `DELETE /api/connections/{id}` | 10 | |
| `GET /api/connections/my` | 10 | |
| `POST /api/transfer/send` | 9 | Dual ledger entries |
| `POST /api/payments/initiate` | 9 | |
| `GET /api/payments/my` | 9 | |
| `GET /api/ledger/passbook` | 7 | |
| `GET /api/ledger/summary` | 7 | |
| `GET /api/ledger/category` | 12 | |
| `GET /api/ledger/monthly` | 12 | |
| `POST /api/split-bills` | 10 | |
| `GET /api/split-bills/my` | 10 | |
| `POST /api/split-bills/{id}/settle` | 10 | |

### Extra APIs (not in Java — keep from Phases 1–5)

| Endpoint | Phase | Purpose |
|---|---|---|
| `POST /imports/preview` | 2 | CSV bulk import |
| `POST /imports/{id}/confirm` | 2 | |
| `GET /health` | 1 | Health check |

---

## Business Rules to Port

From Java services (read source when implementing each phase):

1. **KYC gate** — wallet operations blocked until `kyc_status = APPROVED`
2. **Transfer** — sender debited, receiver credited, both get ledger entries, atomic transaction
3. **Expense** — wallet debited, ledger DEBIT with category
4. **Split bill** — full amount deducted from creator; members settle individually; creator credited on settle
5. **Auto-connection** — non-connected split members get connection invite
6. **Default wallet** — created on registration with starting balance (match Java default)

---

## Schema Evolution (existing tables)

Current `accounts` + `transactions` tables (Phase 1) served a personal-finance tracker.

**Phase 7+** introduces wallet/ledger as the primary money model. Options:

- **Recommended:** Keep `transactions` for CSV bulk import / bank-statement style data; add `wallets` + `ledger_entries` for app wallet flows. Pipeline exports **both** in Phase 13.
- `accounts` may map to external bank accounts linked to a user (future) or be deprecated in favor of wallet.

Final schema decision happens in Phase 7 — document in `docs/database.md` when implemented.

---

## Java Services → Python Services

| Java | Python (`backend/app/services/`) |
|---|---|
| `UserService` | `user_service.py` |
| `JwtService` + security config | `auth_service.py` + `core/security.py` |
| `KycService` | `kyc_service.py` |
| `LedgerService` | `ledger_service.py` |
| `TransferService` | `transfer_service.py` |
| `PaymentService` | `payment_service.py` |
| `ConnectionService` | `connection_service.py` |
| `SplitBillService` | `split_bill_service.py` |
| — | `ingestion_service.py` (already exists, Phase 2) |
