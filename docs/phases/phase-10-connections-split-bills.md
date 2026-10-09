# Phase 10 — Connections & Split Bills

**Status:** Complete  
**Depends on:** Phase 9 (complete)

## Objective

Social connection graph between users and group bill splitting with member settlement,
matching Java FinVault behavior including auto-connection invites.

## Database

Migration `007_connections_split_bills`:

- **`connections`** — `requester_id`, `receiver_id`, `status` (`PENDING` / `ACCEPTED` / `REJECTED`), `note`
- **`split_bills`** — bill header with `total_amount`, `per_person_amount`, `status` (`PENDING` / `SETTLED`)
- **`split_bill_members`** — normalized members with `connection_status` and `settlement_status`

## Backend APIs

### Connections

| Method | Path | Description |
|---|---|---|
| POST | `/api/connections/request` | Send connection request |
| POST | `/api/connections/{id}/accept` | Accept request |
| POST | `/api/connections/{id}/reject` | Reject request |
| POST | `/api/connections/{id}/withdraw` | Withdraw pending sent request |
| DELETE | `/api/connections/{id}` | Remove accepted connection |
| GET | `/api/connections/my` | List connections with direction (`SENT` / `RECEIVED` / `CONNECTED`) |

### Split Bills

| Method | Path | Description |
|---|---|---|
| POST | `/api/split-bills` | Create split bill |
| GET | `/api/split-bills/my` | List bills created by or including user |
| POST | `/api/split-bills/{id}/settle` | Member pays their share to creator |

### Business rules

- Rejected connections can be re-requested (record reset to `PENDING`)
- Split creation debits creator for full `total_amount` (`SPLIT_DEDUCT` ledger entry)
- `per_person_amount = total_amount / (members + creator)`
- Non-connected members get `INVITED` status and an auto connection request
- Accepting a connection updates pending split bill member rows to `CONNECTED`
- Settlement transfers `per_person_amount` from member to creator (`SPLIT_BILL` category)
- Bill status becomes `SETTLED` when all members have settled

## Frontend

- **Connections** (`/connections`) — send/accept/reject/withdraw/remove
- **Split Bills** (`/split-bills`) — create bill, view members, settle share

## Next

Phase 12 — Analytics layer. Say **"start Phase 12"**.
