# Cursor Workflow

Use this workflow for the entire project.

## Before Every Phase

Tell Cursor:

> Read `PROJECT_PLAN.md`, `docs/rules/AI_RULES.md`, and the relevant phase doc in `docs/phases/`.
> Implement only this phase. Do not start future phases.

## Phase Progression

| Say this | What gets built |
|---|---|
| `start Phase 6` | Auth, JWT, register/login, user profile |
| `start Phase 7` | Wallet, ledger, passbook |
| `start Phase 8` | KYC + admin approval |
| `start Phase 9` | Transfers + expenses |
| `start Phase 10` | Connections + split bills |
| `start Phase 11` | Modern frontend (all pages) |
| `start Phase 12` | Analytics APIs + charts |
| `start Phase 13` | Pipeline reconnect (ledger → Bronze) |
| `start Phase 14` | Docker, deploy, polish |
| `start Phase 16` | Payment cards |
| `start Phase 17` | Incremental pipeline + Airflow |

Phases 0–5 are complete (foundation + data pipeline).

## After Every Phase

Tell Cursor:

> Run relevant tests, update documentation and `PROJECT_PLAN.md`, summarize what changed, and stop.

## If Cursor Over-Scopes

> STOP. You are implementing functionality from a future phase. Revert/avoid the future-phase work and continue only with the current phase requirements.

## Reference Material

- Doc index: `docs/README.md`
- Java source: [github.com/developsumitkumar/FinVault](https://github.com/developsumitkumar/FinVault)
- Port mapping: `docs/reference/java-port-reference.md`
- Design system: `docs/design/design-system.md`
- UI direction: `docs/design/ui-direction.md`
- Cards spec: `docs/features/cards.md`
