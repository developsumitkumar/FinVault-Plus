# Screenshots

Product screenshots for the README and portfolio materials.

| File | Page | Description |
|---|---|---|
| `finvault-tour.gif` | Whole app (animated) | ~40 s walkthrough: landing → home → insights → passbook → send → cards → loans → people → help |
| `finvault-tour.mp4` | Whole app (video) | Higher-quality version of the tour |
| `page-tour.png` | `/passbook` | Animated page tour mid-chapter (double-entry scene) |
| `landing.png` | `/` (signed out) | Flow-field hero, live API health |
| `login.png` | `/login` | Split auth screen with flow field |
| `dashboard.png` | `/` | Vault balance panel, cards, 30-day stats, live activity |
| `analytics.png` | `/analytics` | Insights overview: balance forecast, KPIs, money-flow Sankey |
| `insights-spending.png` | `/analytics?tab=spending` | Category donut, mix over time, treemap, movers |
| `insights-patterns.png` | `/analytics?tab=patterns` | Calendar + hour heatmaps, radar, outliers, recurring |
| `passbook.png` | `/passbook` | Search, filter chips, pagination and export |
| `send-money.png` | `/send-money` | Transfer composer, funding-source tiles, pay again, recent transfers |
| `cards.png` | `/cards` | 3D card stage, freeze switch, limits, card standing |
| `loans.png` | `/loans` | Active loan: EMI ring, balance chart, next EMI |
| `connections.png` | `/connections` | Network orbit, requests inbox, network grid, discovery |
| `split-bills.png` | `/split-bills` | Live split preview, bill cards with progress rings |
| `profile.png` | `/profile` | Identity panel, avatar studio, details, preferences |
| `help.png` | `/help` | Ask Jev inline, topics, tickets |
| `api-docs.png` | `/docs` | FastAPI OpenAPI (Swagger) |
| `airflow-dag-runs.png` | Airflow UI `/dags/.../runs` | Scheduled + manual DAG runs (all Success) |
| `airflow-export-task.png` | Airflow task overview | `export_from_postgres` — ~4s, 0 failures |
| `airflow-spark-task.png` | Airflow task overview | `run_bronze_silver_gold` — ~10–12s, 0 failures |
| `airflow-scheduled-logs.png` | Airflow task logs | Scheduled run recovered on retry; Spark row counts |
| `airflow-manual-logs.png` | Airflow task logs | Manual run recovered on Try 3 |
| `jev-cinematic.gif` | Care cinematic (animated) | Full 20s walkthrough converted from `jev.mp4` for README inline play |
| `jev-decision-vs-generation.jpg` | Architecture diagram | Jev vs SQL vs LLM ownership (README Care section) |
| `jev-request-path.jpg` | Architecture diagram | Customer message → DecisionClient → AI or Kanban path |
| `jev.mp4` | Demo video | TypeSafe Jev / Care walkthrough |

## Regenerating

With the local stack running (`uvicorn` on `:8000`, `npm run dev` on `:5173`):

```bash
npm install playwright --no-save
npx playwright install chromium
node scripts/capture-readme-screenshots.mjs
```

Page screenshots fold the animated page tours so the page body is visible; `page-tour.png` is taken with motion on.

### Tour GIF

```bash
node scripts/record-readme-tour.mjs   # writes .tour-video/*.webm
V=$(ls .tour-video/*.webm)
ffmpeg -y -i "$V" -filter_complex "[0:v]setpts=PTS/2,fps=8,scale=760:-1:flags=lanczos,split[a][b];[a]palettegen=max_colors=80:stats_mode=diff[p];[b][p]paletteuse=dither=bayer:bayer_scale=5:diff_mode=rectangle" docs/screenshots/finvault-tour.gif
ffmpeg -y -i "$V" -vf "setpts=PTS/1.25,scale=1280:-2" -c:v libx264 -pix_fmt yuv420p -crf 26 -preset slow -movflags +faststart -an docs/screenshots/finvault-tour.mp4
```

The GIF is played at 2× speed and kept around 8 MB so GitHub renders it inline.

Optional environment variables:

| Variable | Default |
|---|---|
| `SCREENSHOT_BASE_URL` | `http://localhost:5173` |
| `SCREENSHOT_API_URL` | `http://localhost:8000/docs` |
| `SCREENSHOT_EMAIL` | `user0045@finvault.seed` (has card + loan) |
| `SCREENSHOT_PASSWORD` | `password123456` |

Use a seeded demo user (`make seed-demo`) so charts and passbook have data.
