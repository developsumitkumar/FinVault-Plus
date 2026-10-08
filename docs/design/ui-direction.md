# UI direction

FinVault's frontend keeps **feature parity** with the original Java app, not visual parity.

**Visual source of truth:** [`design-system.md`](design-system.md). Do not invent colours, radii, shadows, fonts or easing outside that file.

---

## Locked product decisions (Phase 16, "Vault")

| Decision | Choice |
|---|---|
| **Concept** | "The ledger in motion": money as light flowing into a vault (particle flow field + dark vault panels) |
| **Theme** | Light default + first-class dark mode (persisted toggle, View Transitions cross-fade) |
| **Entry** | Public landing page at `/` for signed-out visitors; dashboard at `/` once signed in |
| **Login / Register** | Split screen: flow-field night panel + form |
| **Animation** | Rich but purposeful; everything honours `prefers-reduced-motion` |
| **Charts** | Recharts 3 with design-system tokens; **Insights is the star of the product** |
| **Navigation** | Grouped sidebar → icon rail → floating tab bar + More sheet, plus ⌘K palette and ⌘J Care |

---

## Principles

1. Quiet surfaces, one luminous focal point per screen
2. Numbers are engineered: mono figures, running balances, ledger refs
3. Motion explains state
4. Designed for every aspect ratio, including landscape phones and ultrawide
5. The backend's depth is visible: ledger, Jev triage, pipeline, billing cycles, EMIs

---

## Stack

| Layer | Choice |
|---|---|
| Framework | React 19 + Vite + TypeScript |
| Styling | Tailwind CSS v4 (tokens in `src/index.css`) |
| Animation | Framer Motion + canvas (`FlowField`) |
| Charts | Recharts 3 (`components/charts/*`, `lib/chart-colors.ts`) |
| Analytics | Client-side engine in `lib/analytics.ts` over the full ledger |

---

## Related assets

- Approved visual preview: [`../assets/vault-preview.html`](../assets/vault-preview.html)
- Card flip preview (legacy): [`../assets/card-components-preview.html`](../assets/card-components-preview.html)
- Cards feature spec: [`../features/cards.md`](../features/cards.md)
