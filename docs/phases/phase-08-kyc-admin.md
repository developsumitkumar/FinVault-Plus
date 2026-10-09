# Phase 8 — KYC & Admin

**Status:** Complete  
**Depends on:** Phase 7 (complete)

## Objective

KYC submission with mock document upload (PDF/JPG/PNG), admin review workflow, and
wallet operation gating until approved.

## Database

Migration `005_kyc_submissions`:

- **`kyc_submissions`** — one per user, stores document metadata + file path
- Files stored locally under `backend/uploads/kyc/{user_id}/`

## Backend APIs

| Method | Path | Auth | Description |
|---|---|---|---|
| POST | `/api/kyc/submit` | User | Multipart: `document_type`, `document_number`, `document` file |
| GET | `/api/kyc/status` | User | Current submission status |
| GET | `/api/kyc/document/{id}` | User/Admin | Download uploaded document |
| GET | `/api/admin/kyc/pending` | Admin | Pending queue with user info |
| POST | `/api/admin/kyc/approve` | Admin | Approve submission |
| POST | `/api/admin/kyc/reject` | Admin | Reject with reason |

### Document upload rules

- Allowed: **PDF**, **JPG**, **JPEG**, **PNG**
- Max size: **5 MB**
- Document types: `PASSPORT`, `DRIVERS_LICENSE`, `NATIONAL_ID`
- Resubmit allowed after **REJECTED**

### KYC gate

Wallet debits/credits (except `OPENING_BALANCE`) require `user.kyc_status == APPROVED`.

## Frontend

- **KYC** page — upload form + status + view document
- **Admin KYC** page — pending queue, approve/reject, view documents

## Next

Phase 10 — Connections & split bills. Say **"start Phase 10"**.
