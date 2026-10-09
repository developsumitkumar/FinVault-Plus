# Customer Care AI + Support Ops + Personal Loans

> Status: **design locked for discussion → implement in phases**  
> Showcase tech: **TypeSafe Jev** (System One decisions) + generative LLM for text/UI payloads  
> Related: wallet/ledger, KYC, cards, transfers, split bills

---

## 0. What this initiative adds

Three product surfaces that work together:

1. **In-app AI assistant (customer)** — smart, not FAQ-only: answers how-to questions, reads the logged-in user’s data (JWT), can return text + tables + small chart specs, and can open a support ticket when a human is needed.
2. **Support console (staff)** — one modern **shared Kanban** for agents, managers, and owner (each sees a filtered board). Ticket inbox + live chat with the customer.
3. **Personal loans** — `personalLoans`: activity-based max amount, ROI, EMI schedule. Assistant explains eligibility; support can review limit increases under hierarchy rules.

**Superstar of the stack:** **Jev** does categorization, urgency, “AI vs human”, and escalation routing (typed + probabilities, low latency). The LLM never owns routing.

---

## 1. Locked decisions

| Topic | Decision |
|---|---|
| Assistant scope | Full: FAQ + authenticated user data + tables/charts in replies |
| Decision engine | **Jev first** (OpenRouter `/api/alpha/decisions` or TypeSafe). Showcase in README |
| Text / charts LLM | Local **Ollama** for $0 learning/dev; swappable via `LLM_PROVIDER` |
| Support login | Toggle on login page: **Customer** \| **Support** |
| Staff org | 10 agents → 2 managers → 1 owner |
| Kanban | **One modern board for all roles**, filtered by role/assignment |
| Escalation | Real tickets in DB, auto-assign 1 of 10 agents; escalate up hierarchy |
| Loan category | **Build personal loans** (not “always escalate / not offered”) |
| Privacy | Assistant tools see **only** the logged-in customer’s data |
| Staff DB freedom | Real-life scoped: read/assist/approve workflows + audit log; no silent ledger forgery |

---

## 2. Architecture (Jev + LLM)

```text
Customer message (JWT)
        │
        ▼
┌───────────────────────┐
│ Jev (System One)      │  category · urgency · needs_human?
│                       │  loan_intent? · confidence
└──────────┬────────────┘
           │
     ┌─────┴──────────────────────────┐
     │ high confidence, AI can answer │ needs human
     ▼                                ▼
┌─────────────────┐            ┌──────────────────┐
│ Tool layer      │            │ Create ticket     │
│ (user APIs)     │            │ Jev → category    │
│ balance, KYC,   │            │ auto-assign agent │
│ ledger, loan    │            └────────┬─────────┘
└────────┬────────┘                     ▼
         ▼                       Support Kanban + chat
┌─────────────────┐
│ LLM (Ollama)    │  prose + optional {table, chart} JSON
└─────────────────┘
```

### What Jev owns (showcase)

Parallel questions per message / ticket (choice / noul / score):

| Question key | Type | Example labels |
|---|---|---|
| `category` | choice | `refund`, `kyc`, `transfer`, `cards`, `passbook`, `split_bills`, `personal_loan`, `account_access`, `other` |
| `urgency` | score / choice | `low`, `medium`, `high`, `critical` |
| `needs_human` | noul | AI-layer signal (answer vs open ticket); does **not** auto-escalate Kanban |
| `loan_topic` | choice | `eligibility`, `emi_explain`, `limit_increase`, `application_status`, `none` |
| `safe_auto_reply` | noul | gate before LLM sends without review |

Jev does **not** compute EMI math or invent balances. Code + SQL do arithmetic; Jev only routes.

### What the LLM owns

- Natural language explanations (loan ROI/EMI in plain English, KYC steps, etc.)
- Structured answer payload the React widget can render:

```json
{
  "text": "…",
  "table": { "columns": ["month", "emi", "principal", "interest"], "rows": [["1", "…", "…", "…"]] },
  "chart": { "type": "line", "title": "Balance trend", "points": [{"x": "2026-01", "y": 1200}] }
}
```

Schema-validated on the backend before it hits the UI (hallucinated chart types rejected).

### Env shape (learning-friendly)

```bash
DECISION_PROVIDER=jev          # required for showcase path
JEV_API_KEY=...                # OpenRouter / TypeSafe
JEV_MODEL=typesafe/jev-1.13-20260917

LLM_PROVIDER=ollama            # free local text
OLLAMA_BASE_URL=http://127.0.0.1:11434
OLLAMA_MODEL=llama3.2
```

`DecisionClient` is a single interface so demos always call Jev in production config. Optional `DECISION_PROVIDER=stub` only for CI/offline unit tests — not the portfolio default.

**Cost note:** Jev via OpenRouter is typically very cheap for triage (cents per 1k decisions). Ollama keeps reply generation at $0. You need API access for Jev (early access / OpenRouter key); that is the one non-free dependency for the showcase.

---

## 3. Support organization & login

### Roles (extend `users.role`)

Today: `USER`, `ADMIN`.  
Add (or map support staff as users with):

- `SUPPORT_AGENT` (10 seeded)
- `SUPPORT_MANAGER` (2 seeded)
- `SUPPORT_OWNER` (1 seeded)

Keep existing `ADMIN` for platform admin (KYC queue, etc.) unless we later merge concepts.

### Login toggle

- `/login` toggle: **Customer** | **Support**
- Support path: only `SUPPORT_*` roles succeed; redirect → `/support`
- Customer path: unchanged app shell; assistant widget available when authenticated

### Hierarchy

```text
SUPPORT_OWNER (1)
├── SUPPORT_MANAGER (2)
│   └── SUPPORT_AGENT (10)  ← default ticket assignees
```

| Action | Agent | Manager | Owner |
|---|---|---|---|
| Work assigned tickets | ✓ | ✓ | ✓ |
| See unassigned / team queue | limited | ✓ | ✓ |
| Reassign tickets | — | ✓ | ✓ |
| Escalate to manager / owner | ✓ | ✓ | — |
| Approve loan limit increase | — | ✓ (cap) | ✓ (final) |
| Approve refund / goodwill credit workflow | — | ✓ | ✓ |
| Direct silent wallet edit | ✗ | ✗ | ✗ (must use audited action) |

Every staff mutation → `support_audit_log`.

---

## 4. Shared modern Kanban

One board component; filters differ by role.

**Columns (v1):**  
`NEW` → `TRIAGED` → `IN_PROGRESS` → `WAITING_ON_USER` → `ESCALATED` → `RESOLVED`

| Role | Default filter |
|---|---|
| Agent | Assigned to me + unassigned in my categories (optional) |
| Manager | My team’s open tickets + escalations |
| Owner | All open + escalated-to-owner |

Interactions: drag card between columns, open ticket drawer (chat + user context strip: KYC, balance snapshot, open loan), internal notes vs customer-visible replies.

---

## 5. Tickets & messaging (schema sketch)

```text
support_tickets
  id, user_id, assignee_id (nullable),
  category,              -- from Jev
  urgency,               -- from Jev
  status,                -- Kanban column
  subject, summary,
  jev_confidence, jev_raw jsonb,
  created_at, updated_at, resolved_at

support_messages
  id, ticket_id, sender_type (USER|AGENT|ASSISTANT|SYSTEM),
  sender_id nullable, body, payload jsonb (tables/charts),
  created_at

support_audit_log
  id, actor_id, action, entity_type, entity_id, detail jsonb, created_at

assistant_sessions / assistant_messages  -- optional separate from tickets for pure AI chat
```

**Auto-assign:** least-open-tickets among `SUPPORT_AGENT`, sticky if user already has an open ticket with an agent.

---

## 6. Personal loans (`personalLoans`)

### Product rules (real-life shaped, demo-tunable)

1. **Eligibility gate**
   - KYC `APPROVED`
   - Account age ≥ N days (e.g. 30)
   - Minimum activity: e.g. ≥ M ledger events or inflow in last 90 days
   - No active loan in `DEFAULTED` / too many concurrent loans (v1: max 1 active)

2. **Max principal** (computed in code, not by LLM/Jev)
   - Base from trailing activity / average balance band
   - Caps: absolute min/max (e.g. $100 … $10,000 demo)
   - Formula documented in service + unit tests

3. **ROI (interest)**
   - Risk band from activity → APR (e.g. 10–24% demo)
   - Stored on the loan application / offer

4. **EMI**
   - Tenure options: 3 / 6 / 12 months (v1)
   - Standard amortization; schedule rows persisted
   - EMI debit from wallet on due date (job or on-demand simulator for demo)

### Tables (sketch)

```text
loan_products          -- optional catalog (personal_loan)
loan_offers            -- computed max, apr, tenure options, expires_at, user_id
loan_applications      -- user choice of amount/tenure, status
loans                  -- active agreement: principal, apr, tenure, status
loan_emi_schedule      -- month_no, due_date, emi, principal, interest, status
loan_limit_requests    -- user/support ask to raise max; manager/owner approval
```

Statuses (applications/loans): `DRAFT` → `OFFERED` → `APPLIED` → `APPROVED` → `ACTIVE` → `CLOSED` | `REJECTED` | `DEFAULTED`

### AI + support on loans

| User ask | Path |
|---|---|
| “How do personal loans work?” | Jev `personal_loan` + `eligibility` explain → LLM + KB (no ticket) |
| “What’s my max loan?” | Tool: compute/read `loan_offers` for this user → LLM explains |
| “Will my max increase if I ask support?” | Honest policy text + optional **limit increase request** ticket (`loan_limit_requests`) → manager/owner |
| “Why was I rejected?” | Tools + LLM; if dispute → human ticket |

Support **cannot** invent a higher limit in chat without creating an approved `loan_limit_requests` row (audit + Kanban).

---

## 7. Customer assistant capabilities (v1)

Fintech-style Care chat (Phase F+):

- Local **intent router** (balance, transactions, category spend, cards, loans, transfers, KYC, greeting, human handoff)
- **Multi-turn context** (last messages + follow-up entity carryover, e.g. “what about travel?”)
- Structured reply: `text` + optional `table` / `chart` + **`follow_ups`** chips
- Widget: starter prompts, typing state, tappable follow-ups

Authenticated tools (scoped to `current_user`):

- `get_account_snapshot` / `get_profile_kyc` / `get_wallet_balance`
- `list_recent_ledger` / category spend by month / MTD spend breakdown
- `list_recent_transfers` / payments
- `get_cards_summary`
- `get_loan_offer` / `get_active_loan` / `get_emi_schedule`
- `create_support_ticket` (when Jev `needs_human` or user asks for agent)

Knowledge pack (curated chunks): KYC, cards, loans, transfers, passbook, refunds, “we never ask for OTP”.

---

## 8. Implementation phases

| Phase | Deliverable |
|---|---|
| **A** | Roles + seed 10/2/1 staff + login toggle + `/support` shell |
| **B** | Tickets + messages + **shared Kanban** + agent↔user chat |
| **C** | **Jev** `DecisionClient` — category / urgency / needs_human on new messages & tickets |
| **D** | Auto-assign + escalation to manager/owner |
| **E** | Personal loans schema + offer/EMI engine + customer UI |
| **F** | Customer AI widget (Ollama) + tools + table/chart payload |
| **G** | Loan limit-increase workflow on Kanban + README / screenshots (Jev as headline) |

Do not start coding until this doc is accepted. Prefer A→B first so Kanban is real before AI.

---

## 9. README / portfolio angle

Call out explicitly:

> Support triage uses **TypeSafe Jev** (System One) for sub-second structured decisions — category, urgency, escalation — while a generative LLM writes customer-facing answers. Personal loans eligibility and EMI are computed in deterministic services; the assistant explains them and routes limit disputes to a human hierarchy on a shared Kanban.

---

## 10. Open items (minor — decide during Phase A/E)

- Exact loan formula constants (caps, APR bands, activity thresholds)
- Whether `ADMIN` and `SUPPORT_OWNER` are the same person in seed data
- Soft vs hard EMI failure (retry days before `DEFAULTED`)
- Jev confidence thresholds after a 30–50 example eval set

---

## 11. Acceptance checklist (definition of done for the initiative)

- [x] **Phase A** — Support roles migration, seed 10/2/1, login toggle, `/support` shell  
- [x] **Phase B** — Tickets + messages + shared Kanban + agent↔user chat (`/help`, `/support/board`)  
- [x] **Phase C** — `DecisionClient` (Jev + stub): category / urgency / `needs_human` on tickets (stored; staff escalate hierarchy separately)  
- [x] **Phase D** — Sticky auto-assign + escalate agent → manager → owner (`reports_to`, `escalation_level`, `/escalate`)  
- [x] **Support UX** — Handed-off tickets stay visible (view only); `resolved_by`; full-page Kanban + sidebar; private notes  
- [x] Shared Kanban usable by all three roles with correct filters (filters live; drag/status live)  
- [x] New customer issues get a **Jev/stub category** stored on the ticket (`jev_confidence` / `jev_raw`)  
- [x] AI can answer with user-scoped data + optional table/chart  
- [x] Ticket path: **AI assistant (Phase F) → Jev triage → agent → manager → owner** (staff escalate only; Help does **not** auto-escalate)  
- [x] `needs_human` stored on triage for AI routing; does **not** skip the agent queue  
- [x] **Phase E** — Personal loans: offer engine, apply/disburse, EMI schedule + pay-next (`/loans`)  
- [x] User can receive a personal loan offer (max, ROI, EMI) from activity rules  
- [x] **Phase F** — Customer AI widget (Ollama/stub) + tools + table/chart payload; can open tickets  
- [x] **Phase G** — Loan limit-increase requests (`loan_limit_requests`) → manager/owner decide + audit; Care UI `/support/loan-limits`  
- [x] Limit-increase requests require manager/owner approval with audit log  
- [x] README documents Jev as the decision layer  


### Phase F env

```bash
LLM_PROVIDER=ollama          # or stub for CI
OLLAMA_BASE_URL=http://127.0.0.1:11434
OLLAMA_MODEL=llama3.2
# Install Ollama + pull model: ollama pull llama3.2
```

### Support path (locked)

```text
Customer
  │
  ├─► AI assistant (Phase F) — chat first; can open a ticket if needed
  │
  └─► Help “New ticket” (direct) ──► Jev triage (category/urgency)
                                      │
                                      ▼
                                   AGENT (Kanban)
                                      │ staff escalate
                                      ▼
                                   MANAGER
                                      │ staff escalate
                                      ▼
                                   OWNER
```

### Phase D hierarchy

```text
owner@finvault.support
├── manager01  ← agents 01–05
└── manager02  ← agents 06–10
```

- New tickets: sticky agent (open ticket) else least-busy agent; **never** auto-skip to manager  
- Staff **Escalate** or drag to `ESCALATED`: AGENT→MANAGER→OWNER only  
- `needs_human` is for the future AI layer (answer vs open ticket), not Kanban auto-escalate  

### Phase C env

```bash
# backend/.env — portfolio default is Jev; falls back to stub if no API key
DECISION_PROVIDER=jev
OPENROUTER_API_KEY=sk-or-...   # or JEV_API_KEY
# CI / offline: DECISION_PROVIDER=stub
```

### Phase A demo logins

| Role | Email | Password |
|---|---|---|
| Owner | `owner@finvault.support` | `SupportPass123!` |
| Manager | `manager01@finvault.support` | `SupportPass123!` |
| Agent | `agent01@finvault.support` … `agent10@…` | `SupportPass123!` |

```bash
cd backend && alembic upgrade head
make seed-support
# Login page → toggle Support → enter care console (/support)
```

