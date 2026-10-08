# AI Development Rules

FinVault is a portfolio-grade **full-stack fintech + data engineering** project.

The developer is learning while building. AI assistance must optimize for:

1. Understanding
2. Correct architecture
3. Maintainability
4. Explainability
5. Incremental development

## Before Changes

- Read `PROJECT_PLAN.md` and the relevant phase doc in `docs/phases/`.
- Read `docs/reference/java-port-reference.md` when porting Java behavior.
- Inspect existing code — avoid unnecessary rewrites.
- Check whether functionality already exists.

## Scope Control

- Implement **only the requested phase**.
- Never silently implement future phases.
- Never generate large amounts of speculative code.

When the user says **"start Phase N"**, implement Phase N only and stop.

## Architecture

Prefer **simple + correct + explainable** over complicated + impressive-looking.

Do not introduce:

- unnecessary microservices
- Kubernetes
- excessive abstractions
- technologies without a clear purpose

Follow existing layering:

```text
Routes → Services → Repositories → PostgreSQL
```

## Security

Never hardcode passwords, API keys, cloud credentials, tokens, or connection secrets.
Use environment configuration (`.env`, never committed).

## Data Engineering

Do not claim:

- Spark usage without meaningful Spark work
- Azure usage without actual integration
- Databricks usage without actual Databricks execution
- production-grade behavior without verification

## Java Port

When porting from [developsumitkumar/FinVault](https://github.com/developsumitkumar/FinVault):

- Match **behavior and API contracts**, not Java code structure literally.
- Use PostgreSQL relations instead of MongoDB embedded documents.
- Ledger entries are **immutable** — never update or delete them.

## Frontend

- Follow `docs/design/design-system.md` for visuals and `docs/design/ui-direction.md` for product UI decisions.
- Do **not** copy the old MUI pastel-green Java frontend.
- Match **features**, not **visuals**.

## After Every Phase

1. Run relevant tests.
2. Verify the implementation.
3. Update documentation and `PROJECT_PLAN.md`.
4. State what remains.
5. Stop.

The developer should be able to explain every major component in an interview.
