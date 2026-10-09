# FinVault documentation

Single map of project docs. Prefer these paths in PRs and AI prompts.

## Start here

| Doc | Purpose |
|---|---|
| [`../README.md`](../README.md) | Project overview, quick start, feature list |
| [`../CONTRIBUTING.md`](../CONTRIBUTING.md) | How to contribute locally |
| [`../PROJECT_PLAN.md`](../PROJECT_PLAN.md) | Phase roadmap and status |

## Rules (for humans + AI)

| Doc | Purpose |
|---|---|
| [`rules/AI_RULES.md`](rules/AI_RULES.md) | Scope, layering, and AI coding constraints |
| [`rules/CURSOR_WORKFLOW.md`](rules/CURSOR_WORKFLOW.md) | Phase-by-phase Cursor workflow |

## Design

| Doc | Purpose |
|---|---|
| [`design/design-system.md`](design/design-system.md) | **Visual source of truth** — tokens, components, patterns |
| [`design/ui-direction.md`](design/ui-direction.md) | Product UI decisions (theme, motion, stack) |

## Architecture & data

| Doc | Purpose |
|---|---|
| [`architecture.md`](architecture.md) | System layers and boundaries |
| [`architecture/jev-decision-layer.md`](architecture/jev-decision-layer.md) | **Jev** decision client, call sites, fallbacks, Mermaid flows |
| [`database.md`](database.md) | Schema notes |
| [`data-pipeline.md`](data-pipeline.md) | Medallion pipeline overview |
| [`data-quality.md`](data-quality.md) | Import / DQ validation |
| [`data-governance.md`](data-governance.md) | PII / PAN-CVV demo trade-offs |
| [`data-pipeline-learning-notes.md`](data-pipeline-learning-notes.md) | Incremental load + Airflow build notes |
| [`azure-storage.md`](azure-storage.md) | ADLS Gen2 |
| [`databricks.md`](databricks.md) | Databricks / Delta |
| [`deployment.md`](deployment.md) | Deploy guide |

## Features

| Doc | Purpose |
|---|---|
| [`features/cards.md`](features/cards.md) | Payment cards spec (debit / credit / Black) |
| [`features/customer-care-ai.md`](features/customer-care-ai.md) | Support ops, Jev + LLM assistant, personal loans |

## Reference

| Doc | Purpose |
|---|---|
| [`reference/java-port-reference.md`](reference/java-port-reference.md) | Java FinVault → Python mapping |
| [`phases/`](phases/) | Per-phase implementation briefs |
| [`screenshots/`](screenshots/) | UI screenshots for README |
| [`assets/`](assets/) | Design previews / images |

## What we intentionally do **not** keep

- Duplicate specs (e.g. a second `Cards.md` under `backend/`)
- Archived copies of the old Java README (link the public repo instead)
- Generated pipeline / Airflow runtime files (gitignored)
