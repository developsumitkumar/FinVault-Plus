# Screenshots

Product screenshots for the README and portfolio materials.

| File | Page | Description |
|---|---|---|
| `login.png` | `/login` | Split-hero auth screen |
| `dashboard.png` | `/` | Wallet overview, quick actions, recent passbook |
| `analytics.png` | `/analytics` | Charts — balance trend, categories, income vs expense |
| `passbook.png` | `/passbook` | Ledger with filters and export |
| `connections.png` | `/connections` | Social graph, user search, connection requests |
| `split-bills.png` | `/split-bills` | Split bill wizard and settlement progress |
| `profile.png` | `/profile` | Profile, avatar picker, custom upload |
| `api-docs.png` | `/docs` | FastAPI OpenAPI (Swagger) |

## Regenerating

With the local stack running (`uvicorn` on `:8000`, `npm run dev` on `:5173`):

```bash
npm install playwright --no-save
npx playwright install chromium
node scripts/capture-readme-screenshots.mjs
```

Optional environment variables:

| Variable | Default |
|---|---|
| `SCREENSHOT_BASE_URL` | `http://localhost:5173` |
| `SCREENSHOT_API_URL` | `http://localhost:8000/docs` |
| `SCREENSHOT_EMAIL` | `user0001@finvault.seed` |
| `SCREENSHOT_PASSWORD` | `password123456` |

Use a seeded demo user (`make seed-demo`) so charts and passbook have data.
