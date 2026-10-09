# FinVault Design System — "Vault"

**Status:** Active (Phase 16). Replaces the indigo / Plus Jakarta Sans system entirely.
**Scope:** every screen in `frontend/`: landing, auth, customer app, support console.
**Approved preview:** [`../assets/vault-preview.html`](../assets/vault-preview.html) · **Plan:** [`../phases/phase-16-frontend-redesign.md`](../phases/phase-16-frontend-redesign.md)

## 0. Read this first

1. This file is the source of truth for visual decisions. Check here before building or changing UI.
2. **No new colours, radii, shadows, fonts or easing curves** outside §2. Extrapolate from the closest pattern; if unsure, leave `// TODO(design):` and ask.
3. **One idea carries the product: _the ledger in motion_.** Money is drawn as light flowing into a vault. It appears as the particle flow field (landing hero, auth, final CTA) and as the dark luminous **vault panel**: exactly **one focal vault panel per screen**. Everything else stays quiet.
4. Analytics is the star of the product. When trading off scope, invest in Insights.
5. Never reproduce real brand logos or network wordmarks (§6.5).

## 1. Principles

| Principle | In practice |
|---|---|
| Quiet surfaces, one loud focal point | Hairline-bordered white panels; a single `VaultPanel` per screen carries colour and motion |
| Numbers are engineered | All money, refs, IDs and axes in **Geist Mono**; cents de-emphasised on hero figures |
| Motion with meaning | Motion explains state (a balance ticking, a ledger row arriving, a decision being made), never decoration for its own sake |
| Designed per aspect ratio | Phone, landscape phone, tablet, laptop, desktop, ultrawide each get a deliberate layout (§4) |
| Show the engineering | Ledger refs, running balances, Jev decisions and pipeline concepts are visible in the UI, not just in docs |

## 2. Tokens (`frontend/src/index.css`)

Legacy token names are kept so every screen inherits the palette. Use **only** these.

### 2.1 Colour

| Token | Light | Dark | Use |
|---|---|---|---|
| `surface-panel` | `#f3f3ef` | `#080a09` | Page background |
| `surface-card` | `#ffffff` | `#0f1312` | Panels, cards, inputs |
| `surface-muted` | `#ececE7` | `#161b19` | Nested surfaces, tracks, chips |
| `ink-900` | `#0b0f0d` | `#eef2ef` | Primary text, primary button fill |
| `ink-600` | `#565e59` | `#a0aaa4` | Secondary text |
| `ink-400` | `#8d948f` | `#69736e` | Muted text, axes, timestamps |
| `ink-inverse` | `#f3f3ef` | `#080a09` | Text on `ink-900` |
| `line-soft` / `line-strong` | 8% / 14% ink | 7% / 13% white | Hairlines, input rings |
| `brand-600` / `700` / `100` | `#10a874` / `#0a7a54` / `#dff3e9` | `#34d399` / `#6ee7b7` / `#0f2a1f` | Vault green: accents, focus, links, positive |
| `success-*`, `warning-*`, `danger-*` | semantic | semantic | Status only, always with a label or icon |
| `violet-500` / `100` | `#7656d6` / `#ece7fb` | `#a78bfa` / `#1d1733` | **AI / Jev only** (decisions, Care) |
| `info-500` / `100` | `#2f7fc1` / `#e2eef8` | `#7cb4e8` / `#13202c` | Neutral informational status |
| `night` / `night-2` | `#060807` / `#0d1110` | same | Vault surfaces (theme-independent) |
| `glow-mint` `glow-lime` `glow-cyan` `glow-violet` | `#34d399` `#bef264` `#5eead4` `#a78bfa` | same | Only on `night` surfaces: particles, chart lines, CTAs |

### 2.2 Type

| Role | Font | Spec |
|---|---|---|
| UI + headings | **Geist** 300–600 | Headings `font-medium`, tracking −0.04 to −0.055em; body 14.5px |
| Figures | **Geist Mono** (`.num`) | Every amount, reference, ID, axis tick, table number |
| Accent | **Instrument Serif** italic (`.serif-accent`) | One short phrase per screen header ("Evening, *Wei.*"). Never for numbers or body text |
| Eyebrow | `.eyebrow` | 11px, 0.14em tracking, uppercase, `ink-400` |

Hero figures: `clamp(44px, 10cqi, 92px)`, weight 400, tracking −0.06em, cents at 35% white. Page titles: `clamp(28px, 3.6vw, 44px)`.

### 2.3 Shape, depth, texture

| Token | Value | Use |
|---|---|---|
| `rounded-xl` | 20px | Panels, cards, vault panels |
| `rounded-[14px]` | 14px | Inputs, nested surfaces, tooltips |
| `rounded-full` | — | Buttons, chips, segmented controls |
| `.hairline` | `0 0 0 1px line-soft` | Default panel edge (no drop shadow) |
| `.pop-shadow` | hairline + `0 18px 40px -18px rgb(0 0 0/.35)` | **Overlays only**: menus, sheets, palette, tooltips, tab bar |
| `.grain` | SVG noise at 7% overlay | Dark vault surfaces; host must be positioned |
| `.glass` / `.glass-surface` | blurred translucent fills | Controls on `night`; floating tab bar / sticky bars |

### 2.4 Motion (`components/motion/*`)

| Primitive | Behaviour |
|---|---|
| `EASE_OUT_EXPO` `[0.2, 0.8, 0.2, 1]` | Default easing for everything |
| Spring `stiffness 420, damping 34` | Sliding indicators (`layoutId`): nav, segmented controls, palette selection |
| `Reveal` / `Stagger` | Opacity + 18px rise + 6px blur → crisp, 0.8s; on view or immediate |
| `WordReveal` | Headlines rise word by word with blur (70ms stagger) |
| `CountUp` | Numbers tween (ease-out quart, ~1.1s) from previous value; used for balances and KPIs |
| `FlowField` | Canvas particle vortex; pauses off-screen / hidden tab; one still frame under reduced motion |
| `VaultPanel` | Night surface with three drifting blurred colour fields + grain |
| `Magnetic` | Primary CTAs drift toward the pointer (fine pointers only) |
| `Card spotlight` | Cursor-following radial highlight on panels (fine pointers only) |
| Theme switch | View Transitions cross-fade |

**`prefers-reduced-motion`:** every primitive above degrades to its end state; global CSS clamps durations to 0.01ms.

## 3. Components

| Component | Notes |
|---|---|
| `Button` | Variants `default` (ink), `glow` (mint, on dark), `glass` (on dark), `outline`, `secondary`, `ghost`, `destructive`; scale 0.96 on press |
| `Card` (+ `spotlight`) | Hairline panel, container-query host (`@min-[…]:` works inside) |
| `Segmented` / `Tabs` | Pill control with spring indicator; `tone="glass"` on vault panels |
| `Badge` / `StatusPill` | Tinted pill + darker text; `violet` reserved for Jev |
| `StatCard` | label · value · signed delta (`upIsGood`) · hint · sparkline |
| `PassbookTable` | Ledger row: initials, description, time · **ref** · category, signed mono amount, **running balance** |
| `MiniCard` | Brand-agnostic payment card with tilt + sheen; frozen overlay |
| `KycGate` | Compact amber strip with 3-step progress |
| `CommandPalette` (⌘K), `BottomSheet`, `MoreSheet` | All destinations + quick actions; sheets are drag-to-dismiss |
| `AssistantWidget` (⌘J) | Night header, chat bubbles, Recharts payloads, **Jev triage strip** (provider, category, urgency, p(human)) |

## 4. Layout & responsiveness

| Class | Detection | Navigation | Layout |
|---|---|---|---|
| Small phone | < 380px (`xs`) | Floating glass tab bar (Home · Insights · ＋ · Cards · More) | 1 column; KYC chip hidden |
| Phone | 380–639px | Tab bar + **More** sheet (every route) + ＋ action sheet | 1 column; filter rows scroll horizontally |
| Landscape phone / short | `short:` = `max-height: 500px` + landscape | 76px icon rail (tab bar hidden) | Reduced vertical padding; secondary sidebar items hidden |
| Tablet | 640–1099px | Icon rail with tooltips | 1 column analytics; container queries reflow cards |
| Laptop / desktop | ≥ 1100px and height > 500px | 236px grouped sidebar (collapsible, persisted) | 12-col grids from `xl` (1280px) |
| Ultrawide | ≥ 1800px (`3xl`) | Sidebar | Content widens to 1880px; dashboard gains a sticky right rail |

Rules: `clamp()` for type and page gutters (`clamp(16px, 2.4vw, 32px)`); `env(safe-area-inset-*)` on bars and sheets; cards use container queries (`[container-type:inline-size]` + `@min-[Npx]:`) so a widget adapts to its slot, not the viewport.

Navigation groups (same routes as before): **Money** Home `/`, Insights `/analytics`, Activity `/passbook` · **Move** Send, Split bills · **Products** Cards, Loans · **People** · **Account** Account, Help. Wallet, Transactions, Expenses, Import, Verification and Admin live in More / ⌘K.

## 5. Charts (Recharts 3)

Tremor was removed: it pinned React 18 (broken clean installs) and rendered black under Tailwind v4.

- **Theme:** `useChartTokens()` → `lib/chart-colors.ts`. Never hard-code chart hex in pages.
- **Categorical:** fixed 7-slot order + grey **Other** (validated with the dataviz palette validator: light and dark pass every hard gate; light-mode aqua/yellow/magenta are below 3:1, so every categorical chart has a legend and a table view). A category's slot comes from **all-time** spend rank, so its colour never changes with filters.
- **Status:** income = `brand`, spending = `danger`; up/down deltas coloured by whether up is good.
- **Sequential:** one green ramp (`SEQUENTIAL_LIGHT/DARK`) for heatmaps and pivots.
- **Marks:** 2px lines, ≤ 24px bars with 4px rounded data-ends, area fills as soft gradients, solid hairline grids (dashes only for projections and averages), no dual axes.
- **Interaction:** crosshair tooltip on line/area, per-mark tooltips on bars/cells, values lead labels; every `ChartCard` has a chart ↔ **table** toggle.
- **Filters:** one row above the charts (tab, range, granularity, compare, categories); it scopes everything below.

### Insights page (`/analytics`)

| Tab | Contents |
|---|---|
| Overview | Vault balance hero (7-day average + 30-day linear forecast with ±1σ band), 8 KPI tiles with deltas + sparklines, money-flow **Sankey** (income sources → wallet → categories), generated insights, income vs spending |
| Cash flow | Net flow bars with brush zoom, cumulative net, period vs previous (ghost bars), month-to-date pace vs last month and 3-month average, **runway** gauge, flows by ledger event |
| Spending | Interactive donut + ranked bars, category mix over time (amount / share, toggleable legend), treemap, change vs previous (diverging), frequency × ticket-size bubble chart, top payees / payers |
| Patterns | 53-week calendar heatmap, weekday × hour heatmap, weekly rhythm radar, ticket-size histogram, expense scatter with **2σ outlier** detection, **recurring-payment** detection with next dates |
| Products | Card utilisation gauge, statements by status, card spend by category, loan amortisation (principal / interest, paid vs upcoming), outstanding principal, split-bill settlement |
| Explorer | **What-if savings simulator**, **goal planner**, category × month pivot heatmap, searchable / sortable transaction explorer with CSV export |

All series are computed client-side in `lib/analytics.ts` from `GET /api/ledger/passbook` (full history), so every chart agrees with the ledger.

## 6. Patterns

### 6.1 Landing (`/` when signed out)
Floating pill nav (dark over the hero, light after scroll) → flow-field hero with live `/api/health` badge and word-reveal headline → stack marquee → sticky scroll story (ledger legs → Jev decision → medallion pipeline) → bento product grid with in-view micro-animations → flow-field CTA.

### 6.2 Auth
Split screen: night panel with flow field (top band on phones), form on the right. Customer / Support staff segmented control.

### 6.3 Forms
Labels above fields; inputs 44px, 14px radius, inset ring, green focus ring with soft halo; errors in `danger-500` beneath.

### 6.4 Empty, loading, error
Empty: dashed hairline box, one sentence, one next step. Loading: shimmer skeletons shaped like the content. Error: tinted danger panel, plain statement of what happened.

### 6.5 No real logos
Counterparties and users use initials; payment cards show a plain-text tier and network label, never a wordmark styled to look official.

### 6.6 Support console
Same tokens; an always-dark (`night`) sidebar distinguishes staff from customers. Kanban columns scroll horizontally with snap below 2xl; ticket cards show Jev confidence.

### 6.7 Page tours (`components/hero/`)
Every signed-in route opens with a chaptered motion explainer of what the page does, rendered by the shells (`RouteHero`, lazy-loaded) from the route → tour map in `tours.tsx`.
- **Layout:** chapter list beside a night stage when the hero is ≥ 880px wide; on narrower containers the stage is stacked above story-style segments and the active chapter's text. On landscape phones (`short:`) it uses two columns and the lead is hidden.
- **Stage:** scenes are composed on a fixed 480 × 300 canvas (`ScaledStage`) so cursor paths and labels land identically at every size. Scenes come from a shared kit in `scenes.tsx` (flow, rows/search/filter, bars, donut, forecast, heatmap, export/import, card, form, chat, pipeline, gauge, split, network, kanban, scan, pages, schedule…).
- **Money colours** follow the app rule inside scenes too: debits red, credits green, balances blue.
- **Timing:** each chapter plays for 7 s on a CSS timer. The timer pauses on mouse hover, when the hero is off-screen and when the user presses pause. With reduced motion, autoplay is off and scenes render their final frame.
- **Dismissal:** "Hide" folds the hero into a one-line "Play tour" bar, and the choice is remembered per page in `localStorage` (`fv.hero.<id>`).
- To add a page, add a `Tour` to `TOURS` and its path to `BY_PATH`.

## 7. Feature UI notes (carried over)

#### Passbook (Phase 15)

- **Debit amounts** in table rows: `text-danger-500` (red). Credits: `text-success-700`.
- **Pagination**: show 50 rows initially; "Load more" appends the next 50 client-side.
- **Period filter**: presets (All, 30d, 90d, YTD, Custom date range) — maps to `start_date` / `end_date` API params.
- **Export**: CSV, Excel (`.xlsx` via `xlsx`), PDF (via `jspdf` + `jspdf-autotable`) — exports the **filtered** result set, not just visible rows.

#### Profile & avatars (Phase 15b)

- **Profile page** at `/profile` — edit name/phone, pick avatar, upload JPG/PNG.
- **50 animated preset avatars** (`preset:1` … `preset:50` in `avatar_url`); assigned on registration with least-used distribution (no single preset overloaded).
- **Custom upload** overrides preset; served via `GET /api/user/avatar/{user_id}`.
- Reuse `UserAvatar` component everywhere (connections, splits, header).

#### KYC history

- Multiple `kyc_submissions` per user; resubmit after rejection creates a **new** row; prior rejected row becomes `SUPERSEDED` (kept in history).
- `GET /api/kyc/history` returns full timeline for the user.

## 8. Checklist before shipping UI

- [ ] Only §2 tokens; no hex in components except inside `lib/chart-colors.ts`, `MiniCard` and vault art.
- [ ] Money, refs and axes use `.num` (Geist Mono).
- [ ] At most one `VaultPanel` per screen; one serif accent per header.
- [ ] Checked at 375×812, 812×375, 768×1024, 1280×800, 1440×900, 1920×1080 in light and dark; no horizontal scroll.
- [ ] Every chart: legend for ≥ 2 series, tooltip, table view; categorical colours from `colorFor`.
- [ ] Motion degrades under `prefers-reduced-motion`.
- [ ] No brand logos or network wordmarks.
