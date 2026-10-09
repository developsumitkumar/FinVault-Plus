# Phase 16 — Frontend redesign ("Vault")

**Status:** Implemented on `feat/frontend-modernization`. Verified against the real FastAPI + Postgres stack (seeded demo users, customer and support portals) and at phone, landscape, tablet, desktop and ultrawide sizes; README screenshots regenerated.

**Branch:** `feat/frontend-modernization`
**Goal:** The backend is a serious system (double-entry ledger, KYC state machine, card billing cycles, loan underwriting, Jev decision layer, medallion pipeline). The frontend currently presents it as a generic dashboard template. This phase changes the *look* substantially, makes every screen deliberately responsive across aspect ratios, and surfaces the engineering the backend already does — **without** changing routes, the API client, auth, or React Query hooks.

---

## 1. What stays (structure)

| Kept as-is | Why |
|---|---|
| `src/api/client.ts`, `src/hooks/queries.ts` | API surface is complete; every backend route is already wired |
| `src/auth/*` (AuthContext, ProtectedRoute, RoleGate) | Works; portal split is correct |
| Route table in `main.tsx` (all existing paths) | Bookmarks, docs, screenshots script depend on them |
| React 19 + Vite + Tailwind v4 + React Query + Framer Motion + lucide | Sound stack |

Only additive routing change: `/` renders the **public landing page** when signed out, the dashboard when signed in.

## 2. Problems found in the audit

1. **Clean `npm install` fails** — `@tremor/react@3` pulls `react-day-picker@8` (React 18 peer) against React 19.
2. **Charts render solid black** — `Dashboard.tsx` passes `colors={["indigo"]}`; Tremor's palette classes aren't generated under Tailwind v4.
3. **Mobile nav exposes 5 of 15 destinations** — Cards, Loans, Help, KYC, Import unreachable on phones.
4. **Flat 15-item sidebar** with duplicate-looking Passbook / Transactions.
5. **Fixed-width assumptions** — one `lg` breakpoint, no tablet rail, no ultrawide layout, no short-landscape handling, tables don't reflow.
6. **Backend depth invisible** — Jev triage (category / urgency / needs-human) is persisted but never shown to customers; ledger references, running balances, billing cycles and EMI schedules are rendered as plain lists; nothing tells a visitor this is a ledger-first platform with a data pipeline behind it.

## 3. Visual direction — "Vault"

A precise, instrument-panel feel: the product is a *ledger*, so numbers are the hero and they look engineered.

| Aspect | Old | New |
|---|---|---|
| Typefaces | Plus Jakarta Sans + Inter | **Geist** (UI/display) + **Geist Mono** (all amounts, refs, IDs, timestamps) |
| Base | Blue-grey gradient backdrop, floating rounded shell | Full-bleed app, no backdrop frame; quiet neutral surfaces with hairline borders |
| Accent | Indigo `#4c56d6` | **Vault green** primary (`#0d9f6e` light / `#34d399` dark) + ink neutrals; semantic red/amber kept |
| Dark mode | Derived afterthought | First-class; designed alongside light (deep graphite, not pure black) |
| Cards | 20px radius + heavy shadow everywhere | 14px radius, 1px border, shadow only on overlays |
| Data | Plain text figures | Tabular mono figures, signed colouring, inline sparklines, running balance |
| Motion | Page fade | Number count-ups, staggered list reveal, shared-layout nav indicator; all off under reduced-motion |

### 3.1 Revision 2: "ledger in motion" (approved direction for review)

Feedback on revision 1: too safe, reads like a template. Revision 2 adds a single visual idea that carries through the whole product: **money drawn as particles of light flowing into the vault**.

| Element | Treatment |
|---|---|
| Signature visual | Canvas flow field: particles orbit a "vault" ring and react to the cursor. Used in the landing hero and the final CTA; pauses off-screen, renders one static frame under reduced motion |
| Luminous palette | Mint `#34d399`, lime `#bef264`, cyan `#5eead4`, violet `#a78bfa` (violet = Jev / AI) on night `#060807`. Used only on "vault" surfaces; the rest of the UI stays quiet neutral |
| Type | Geist (300–600), Geist Mono for figures, **Instrument Serif italic** for one accent phrase per screen |
| Texture | SVG film grain over dark surfaces; drifting blurred colour blobs in the dashboard balance panel |
| Motion | Word-by-word blur-in headlines; scroll-reveal; sticky scroll story (ledger → Jev → pipeline) with per-step animated visuals; spring-sliding indicators (nav, segmented controls, switcher); chart line draw-in + scrubber; balance count-up and live ledger arrivals; 3D card tilt and sheen; magnetic buttons; cursor spotlight on panels; View Transitions for theme/page changes |
| Minimalism | One focal panel per screen (the dark vault panel); nav cut to 10 items (Home, Activity, Insights / Send, Splits, Cards, Loans, People / Account, Help) with everything else one tap away in the More sheet / ⌘K |

Implementation note: the flow field and other effects ship as isolated React components (`<FlowField />`, `<Reveal />`, `<Magnetic />`, `<SpotlightPanel />`, `<CountUp />`) using Framer Motion where it helps; all respect `prefers-reduced-motion`.

Static preview of the direction (landing + dashboard, light/dark, all breakpoints): [`../assets/vault-preview.html`](../assets/vault-preview.html).

Tokens live in `index.css` (`@theme` + `html.dark`); `docs/design/design-system.md` is rewritten to match at the end of the phase.

## 4. Responsive system — designed per aspect ratio, not just width

| Class | Detection | Navigation | Layout |
|---|---|---|---|
| Small phone (≤ 374px) | width | Bottom tab bar (4 + **More** sheet) | Single column, compact type scale |
| Phone (375–639) | width | Bottom tab bar + More sheet | Single column, sticky page actions |
| Landscape phone / short screens | `max-height: 500px` | Side icon rail (bottom bar would eat the height) | Two columns, reduced vertical padding |
| Tablet portrait (640–1023) | width | Icon rail with tooltips | 2-column grids |
| Tablet landscape / laptop (1024–1439) | width | Full grouped sidebar (collapsible) | 3-column grids |
| Desktop (1440–1919) | width | Grouped sidebar | 12-col grid + optional right context rail |
| Ultrawide (≥ 1920) | width | Grouped sidebar | Content max-width + persistent right rail (activity / assistant) |

Mechanics: Tailwind v4 custom breakpoints (`xs`, `3xl`) + custom variant `short` for `max-height`, **container queries** on cards so a widget adapts to its slot not the viewport, `clamp()` fluid type, `env(safe-area-inset-*)` for notched devices, tables that become stacked row-cards below `md`.

Navigation grouping (same routes):
- **Money:** Dashboard, Wallet, Passbook, Transactions, Analytics
- **Move:** Send, Expenses, Split bills, Import
- **Products:** Cards, Loans
- **People:** Connections
- **Account:** Profile, KYC, Help, (Admin KYC)

Plus a **⌘K command palette** (all destinations + quick actions) on every size.

## 5. Showing what the backend does

| Backend capability | Where it becomes visible |
|---|---|
| Double-entry ledger, running balance | Passbook rows show ledger ref (mono), running balance column, debit/credit legs on expand |
| Ledger aggregates (`summary`, `monthly-flow`, `category`) | Dashboard hero with balance + 6-month flow sparkline; Analytics rebuilt with net-flow and category breakdown |
| KYC state machine + history | KYC page shows a status timeline (submitted → review → approved/rejected → superseded) |
| Card billing cycles, freeze, pay-off | Cards page: cycle timeline, statement due, utilisation meter |
| Loan offer / underwriting / EMI | Loans page: offer breakdown, EMI amortisation chart, payment progress |
| Jev triage on every Care turn & ticket | Assistant and Help tickets show triage chips: category · urgency · human-handoff probability ("Decided by Jev", "Written by LLM") |
| Support hierarchy (agent → manager → owner) | Support board: escalation trail, Jev confidence per ticket |
| Medallion pipeline, Airflow, Delta | Landing page "Under the hood" section with architecture diagram + live `/api/health` status |

## 6. Implementation order

Each step ends with `npm run build` passing.

1. **Foundation** — remove Tremor, add `recharts` directly; new tokens, fonts, dark/light; rebuild primitives (`Button`, `Card`, `Input`, `Select`, `Badge`, `Tabs`, `Skeleton`) and add `Sheet`, `Dialog`, `EmptyState`, `Money`, `DataList` (table ↔ card reflow), chart wrappers (`AreaTrend`, `BarSeries`, `Donut`, `Sparkline`).
2. **Shells** — `AppShell` (grouped sidebar / rail / bottom bar + More sheet / right rail, command palette), `AuthLayout`, `SupportShell`.
3. **Landing page** — hero, product tour, "Under the hood" (ledger, Jev, pipeline), tech stack, CTA; fully responsive.
4. **Money pages** — Dashboard, Wallet, Passbook, Transactions, Analytics, Send, Expenses.
5. **Product pages** — Cards, Loans, Split bills, Connections, Import.
6. **Account & Care** — Profile, KYC, Help, Assistant widget (Jev chips), Admin KYC.
7. **Support portal** — Home, Board, Loan limits, Notes.
8. **Verification & docs** — viewport matrix (320×568, 375×812, 812×375, 768×1024, 1024×768, 1280×800, 1440×900, 1920×1080, 2560×1080) in light + dark; reduced motion; rewrite `design-system.md` / `ui-direction.md`; refresh README screenshots via `scripts/capture-readme-screenshots.mjs`.

## 7. Out of scope

Backend changes, new API endpoints, auth changes, route renames.
