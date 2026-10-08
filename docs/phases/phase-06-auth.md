# Phase 6 — Auth & Users

**Status:** Complete  
**Depends on:** Phases 0–5 (complete)

## Objective

Replace the hardcoded dev user with real authentication. Users can register, log in, and access
protected endpoints with JWT.

## Backend

### New / updated modules

- `backend/app/core/security.py` — password hashing, JWT create/verify
- `backend/app/services/auth_service.py` — register, login
- `backend/app/services/user_service.py` — profile get/update
- `backend/app/api/routes/auth.py` — `/api/auth/*`
- `backend/app/api/routes/user.py` — `/api/user/*`
- `backend/app/api/deps.py` — `get_current_user`, `require_admin`

### Endpoints

| Method | Path | Auth | Description |
|---|---|---|---|
| POST | `/api/auth/register` | Public | Create user + default account |
| POST | `/api/auth/login` | Public | Return JWT + user summary |
| GET | `/api/user/profile` | User | Current user profile |
| PUT | `/api/user/profile` | User | Update name, phone |

### Database migration

Migration `003_user_auth_fields` extends `users`:

- `role` — `USER` | `ADMIN`
- `kyc_status` — `NONE` | `PENDING` | `APPROVED` | `REJECTED`
- `phone`, `avatar_url` — nullable

Seeded admin: `sumit@test.com` / `123456` (dev only).

## Frontend (minimal)

- Split-hero login and register pages
- Auth context + JWT in localStorage
- Protected routes; redirect to `/login` when unauthenticated

## Completion Criteria

- [x] Register and login work via API and UI
- [x] JWT protects transaction/import/account routes
- [x] Admin user seeded for Phase 8
- [x] `PROJECT_PLAN.md` updated

## Next

Phase 7 — Wallet & ledger. Say **"start Phase 7"**.
