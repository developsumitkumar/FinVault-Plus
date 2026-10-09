# From Zero to a Working, Orchestrated Data Pipeline — My Notes

**Starting point:** an application that is already running and writing data into Postgres (a wallet app — users, wallets, ledger entries — but this applies to any app with a database behind it). No `data-pipeline` folder exists yet. No pipeline code exists yet. Just a live operational database.

**End point:** a Bronze → Silver → Gold pipeline that only processes *new* data each time it runs, is safe to re-run without duplicating anything, and runs on its own every night via Apache Airflow — with zero manual terminal work required once it's set up.

This document is written so I can redo this whole thing, from scratch, on a *different* project, without needing anyone to explain it to me again. Every command is here. Every "why" is here. Read it top to bottom once, then use it as a step-by-step build guide the second time.

---

## Part 0 — The Big Picture (read this before touching a terminal)

### What is a "data pipeline," really?

An application's database (Postgres, MySQL, whatever) is built for one job: handling the app's day-to-day reads and writes fast and safely. It is **not** built for answering questions like "what were total transactions by category, per month, for the last year?" Running heavy analytical queries directly against your live production database is slow, and if the app is under real load, it can actually compete with real users for database resources and slow the app down for them.

A data pipeline's job is to **copy data out of the operational database, clean it up, and reshape it into a form that's fast and easy to analyze** — without ever touching the live app's performance.

### Medallion architecture: Bronze → Silver → Gold

This is just a naming convention for three stages of increasing refinement. It didn't come from any single company — it's become an informal industry-standard pattern:

- **Bronze** = raw data, copied out of the source system almost exactly as-is. No cleaning. This is your safety net — if something goes wrong downstream, you can always reprocess from Bronze without going back to the live database.
- **Silver** = cleaned, deduplicated, standardized. Same grain (same rows) as Bronze, just tidied up — fixed types, dropped garbage rows, normalized text, etc.
- **Gold** = aggregated, summarized, ready for a dashboard or report. "Total spending per user per month" is a Gold table. Nobody queries Bronze or Silver to build a chart — they query Gold.

### Why "before cloud services" context matters

This pattern (copy data out, clean it, summarize it) is not a cloud invention. Companies were doing "ETL" (Extract, Transform, Load) in on-premise data warehouses in the late 1980s and 1990s, on physical servers they owned, using scheduled batch jobs. The cloud didn't invent the problem — it made the infrastructure easier (elastic compute instead of a fixed server, managed storage instead of tape backups, and portable tools like Spark). The core ideas in this document — Bronze/Silver/Gold, incremental loading, orchestration — predate the cloud by decades. Learning them locally first, like we did, is learning the actual fundamentals, not a "toy" version of the real thing.

### Two problems every real pipeline eventually has to solve

1. **Full reload doesn't scale.** A naive pipeline re-reads and re-processes *every row, every time it runs* — even if nothing changed. Fine for 1,000 rows. Not fine for 500,000+ rows, run nightly, forever.
2. **Re-running must be safe.** Pipelines fail. Someone re-runs a failed job. If re-running can duplicate data, you have a silent, dangerous bug. A pipeline that's safe to re-run any number of times without changing the result (beyond the first successful run) is called **idempotent**.

Everything in Part 1 and Part 2 below is building toward solving both of these problems, in that order.

---

## Part 1 — Building the Pipeline (Local, Manual, Full-Reload First)

We deliberately built the *simplest possible working version first* (reads everything, every time), got it running end to end, and only *then* made it incremental. Don't try to build the "smart" version first — get something dumb-but-correct working, then improve it.

### Step 1.1 — Set up your Python environment

You need a virtual environment (a self-contained folder of Python packages, isolated from your system Python) so this project's dependencies don't collide with anything else on your machine.

```bash
cd /path/to/your-project
python3 -m venv .venv
source .venv/bin/activate
```

Your terminal prompt should now show `(.venv)` at the start of the line — that confirms it's active. Everything you install with `pip` from now on goes into this isolated folder, not your system Python.

Install what the pipeline needs:

```bash
pip install pyspark pandas sqlalchemy python-dotenv pg8000
```

What each one is for:
- **pyspark** — the actual transformation engine (Bronze → Silver → Gold logic runs on this).
- **pandas** — used just for the export step, reading SQL query results into a simple table structure before writing them to disk.
- **sqlalchemy** — the standard Python library for talking to SQL databases in a database-agnostic way.
- **python-dotenv** — loads database credentials from a `.env` file instead of hardcoding them in your code.
- **pg8000** — the actual low-level driver that knows how to speak Postgres's wire protocol (sqlalchemy needs a driver underneath it, specific to whichever database you're using).

### Step 1.2 — Design your folder structure

We used this layout (adapt names to your own project):

```
your-project/
  .env                          <- database credentials (never commit this to git)
  data-pipeline/
    common/
      config.py                 <- file paths, constants, shared across everything
      watermark.py               <- incremental-load tracking (built in Part 2)
      transforms/
        bronze.py                <- Bronze-layer helper functions
        silver.py / ledger_silver.py   <- Silver-layer transform logic
        gold.py / ledger_gold.py       <- Gold-layer aggregation logic
    local/
      export_operational.py      <- Postgres -> Bronze
      run_pipeline.py             <- Bronze -> Silver -> Gold (the Spark part)
      output/                     <- where Bronze/Silver/Gold data actually lands
```

**Why split `common` and `local`?** `common` holds logic that doesn't care *where* it runs (the actual transform functions). `local` holds the code specific to running this on your own laptop. Later, when you move to the cloud, you'd add a `cloud/` folder that reuses the same `common` code but with different infrastructure around it (cloud storage instead of local disk, etc.) — you don't rewrite your transform logic, just how it's invoked.

### Step 1.3 — Your `.env` file

```
DATABASE_URL=postgresql+pg8000://your_db_user:your_password@localhost:5432/your_db_name
```

Never commit this file to git. Add `.env` to your `.gitignore`.

### Step 1.4 — Writing the export script (Postgres → Bronze)

This is the only piece of the pipeline that talks to your live operational database. Everything after this reads from local files, never from Postgres again (for that run).

Core pattern (simplified — see your actual file for the full version):

```python
import os
from pathlib import Path
from datetime import UTC, datetime
import pandas as pd
from dotenv import load_dotenv
from sqlalchemy import create_engine, text

EXPORT_QUERY = text("""
    SELECT id, user_id, amount, category, created_at
    FROM your_table
    ORDER BY created_at
""")

def get_database_url() -> str:
    repo_root = Path(__file__).resolve().parents[2]
    load_dotenv(repo_root / ".env")
    return os.getenv("DATABASE_URL")

def export_table(output_path: Path) -> int:
    engine = create_engine(get_database_url())
    df = pd.read_sql(EXPORT_QUERY, engine)
    if df.empty:
        return 0
    df["extracted_at"] = datetime.now(UTC)
    output_path.mkdir(parents=True, exist_ok=True)
    df.to_csv(output_path / "your_table.csv", index=False)
    return len(df)
```

Key ideas here:
- `text(...)` from sqlalchemy wraps a raw SQL string safely. When you need to inject a variable into a query (we do this later for incremental loading), you use a named placeholder like `:since` and pass `params={"since": value}` — **never** build SQL with plain Python string formatting (`f"WHERE x = {value}"`). String formatting into SQL is how SQL injection vulnerabilities happen, even in a personal project — build the habit now.
- `extracted_at` — stamping every exported row with *when it was pulled* is cheap insurance. It tells you, later, "this row was captured at this moment," independent of when the underlying event happened.
- `df.to_csv(..., index=False)` — pandas writes CSV by default in **overwrite mode**. If you call this twice, the second call replaces the file entirely. That matters a lot later (see the idempotency bug in Part 2).

### Step 1.5 — Bronze layer: light enrichment, no real cleaning yet

Bronze is "raw, but tagged." A typical Bronze enrichment function just adds pipeline metadata columns:

```python
from pyspark.sql import DataFrame
from pyspark.sql.functions import lit, current_timestamp

def enrich_bronze_metadata(df: DataFrame) -> DataFrame:
    return (
        df.withColumn("_pipeline_layer", lit("bronze"))
          .withColumn("_ingested_at", current_timestamp())
    )
```

This is PySpark, not pandas, from here on — the export step used pandas because it's just moving a query result to a file; everything from Bronze onward uses PySpark because that's the engine that actually does the cleaning/aggregation work, and it's built to handle much larger data than pandas comfortably can.

### Step 1.6 — Silver layer: actual cleaning

This is where real transformation happens: dropping bad rows, standardizing types, deduplicating. A key trick worth understanding — deduplicating *within a batch* using a window function:

```python
from pyspark.sql import Window
from pyspark.sql.functions import row_number, col

def transform_to_silver(df: DataFrame) -> DataFrame:
    df = df.filter(col("amount").isNotNull())  # drop obviously broken rows

    window = Window.partitionBy("record_id").orderBy(col("created_at").desc())
    df = (
        df.withColumn("_row_num", row_number().over(window))
          .filter(col("_row_num") == 1)
          .drop("_row_num")
    )
    return df
```

What this window function does: for every group of rows sharing the same `record_id`, rank them by `created_at` descending, and keep only rank #1 (the most recent). This protects against the *same batch* containing the same row twice. Important limitation to understand: **this only dedupes within a single run's data — it does not check against rows already saved to disk from previous runs.** That's a completely separate problem, solved differently (see Part 2's idempotency bug).

### Step 1.7 — Gold layer: aggregation

Gold tables are just `groupBy` + aggregate functions, built for a specific question:

```python
from pyspark.sql.functions import sum as spark_sum, count, date_format

def build_monthly_spending(silver_df: DataFrame) -> DataFrame:
    return (
        silver_df
        .withColumn("month", date_format("created_at", "yyyy-MM"))
        .groupBy("month", "category")
        .agg(
            spark_sum("amount").alias("total_amount"),
            count("*").alias("transaction_count"),
        )
    )
```

Each Gold table answers one question. Don't try to build one giant "does everything" table — build several small, purpose-built ones (monthly spending, category breakdown, per-user summary, etc.).

### Step 1.8 — Wiring it together: `run_pipeline.py`

```python
from pyspark.sql import SparkSession

def create_spark(app_name="local-pipeline") -> SparkSession:
    return (
        SparkSession.builder.appName(app_name)
        .master("local[*]")
        .config("spark.sql.session.timeZone", "UTC")
        .getOrCreate()
    )

def write_parquet(df, path, mode="overwrite") -> None:
    path.parent.mkdir(parents=True, exist_ok=True)
    df.write.mode(mode).parquet(str(path))

def run_pipeline():
    spark = create_spark()
    bronze_df = spark.read.option("header", True).csv("output/bronze/your_table.csv")
    bronze_enriched = enrich_bronze_metadata(bronze_df)
    silver_df = transform_to_silver(bronze_enriched)
    write_parquet(silver_df, Path("output/silver/your_table"))
    gold_df = build_monthly_spending(silver_df)
    write_parquet(gold_df, Path("output/gold/monthly_spending"))
    spark.stop()

if __name__ == "__main__":
    run_pipeline()
```

**Why Parquet, not CSV, for Silver/Gold?** Parquet is a columnar binary format — much faster to read/write for analytical workloads, preserves data types properly (CSV turns everything into text), and is the standard format the rest of the data engineering world (Spark, Databricks, cloud data lakes) expects.

**Why `master("local[*]")`?** This tells Spark to run using all available CPU cores on your own machine, instead of connecting to a cluster somewhere. This is exactly what makes it "local" mode — the same code, pointed at a real cluster config later, is how you'd scale this to the cloud without rewriting the transform logic.

### Step 1.9 — Running it manually for the first time

```bash
cd data-pipeline/local
python export_operational.py
python run_pipeline.py
```

Or combined, if your `run_pipeline.py` supports a flag to do both:

```bash
python run_pipeline.py --export
```

At this point you have a genuinely working pipeline. It's just not *good* yet — every single run re-reads and re-processes 100% of your data from scratch, forever. That's Part 2.

---

## Part 2 — Making It Incremental and Idempotent

### Step 2.1 — The watermark concept

A **watermark** is just "the last point I successfully processed up to." Next run, only ask for rows *after* that point. It's the standard mechanism for incremental loading.

```python
# common/watermark.py
import json
from pathlib import Path

WATERMARK_FILE = Path(__file__).resolve().parents[1] / "local" / "output" / "_state" / "watermarks.json"

def get_watermark(dataset: str) -> str | None:
    if not WATERMARK_FILE.exists():
        return None
    data = json.loads(WATERMARK_FILE.read_text())
    return data.get(dataset)

def set_watermark(dataset: str, timestamp: str) -> None:
    WATERMARK_FILE.parent.mkdir(parents=True, exist_ok=True)
    data = json.loads(WATERMARK_FILE.read_text()) if WATERMARK_FILE.exists() else {}
    data[dataset] = timestamp
    WATERMARK_FILE.write_text(json.dumps(data, indent=2))
```

This is a simple JSON file on disk — good enough for a single-machine local pipeline. A real production system would usually use a database table for this instead (a "control table"), for the same reason it uses a database for everything else: multiple processes can read/write it safely at once, which a plain file can't guarantee.

### Step 2.2 — Using the watermark in your export query

```python
INCREMENTAL_QUERY = text("""
    SELECT id, user_id, amount, category, created_at
    FROM your_table
    WHERE created_at > :since
    ORDER BY created_at
""")

def export_table(output_path: Path) -> int:
    engine = create_engine(get_database_url())
    since = get_watermark("your_table")

    if since is None:
        query, params = EXPORT_QUERY, {}   # first run ever: get everything
    else:
        query, params = INCREMENTAL_QUERY, {"since": since}   # every run after: only new rows

    df = pd.read_sql(query, engine, params=params)

    # IMPORTANT: always write, even if df is empty. See Step 2.3 for why.
    output_path.mkdir(parents=True, exist_ok=True)
    df_to_write = df.copy()
    df_to_write["extracted_at"] = datetime.now(UTC)
    df_to_write.to_csv(output_path / "your_table.csv", index=False)

    if not df.empty:
        set_watermark("your_table", str(df["created_at"].max()))
    return len(df)
```

Note the `:since` placeholder and `params={"since": since}` — this is the safe way to inject a variable into SQL, mentioned in Step 1.4.

### Step 2.3 — The idempotency bug we actually hit (read this carefully)

Here's a real bug, found by actually thinking through the incremental logic, not by guessing:

Original code had a shortcut: **"if there's no new data, don't bother writing the Bronze file — just skip it."** That sounds harmless. It is not.

Here's the failure sequence:
1. Run 1: exports 5 new rows → writes `your_table.csv` containing those 5 rows.
2. Bronze → Silver step **appends** those 5 rows onto Silver (this is the whole point of incremental loading — Silver accumulates history across runs instead of being wiped every time).
3. Run 2 (nothing new happened in Postgres): the export step sees 0 new rows and — under the old logic — **skips writing**, leaving yesterday's `your_table.csv` (those same 5 rows) sitting on disk, untouched.
4. The pipeline step doesn't know the Bronze file is stale. It reads whatever's there and appends it to Silver **again** — the exact same 5 rows, now duplicated.

**The fix:** always overwrite the Bronze file every run, even when there are zero new rows — write an empty file. An empty-but-fresh file correctly tells the next stage "nothing new this run." A stale leftover file lies and says "here's new data" when it's actually old data.

```python
def _write_bronze_csv(df, destination, filename, *, always_write=False):
    destination.mkdir(parents=True, exist_ok=True)
    if df.empty and not always_write:
        return 0
    df = df.copy()
    df["extracted_at"] = datetime.now(UTC)
    df.to_csv(destination / filename, index=False)
    return len(df)

# call it with always_write=True for anything that feeds an append-based downstream step
```

**The lesson, generalized:** idempotency bugs are almost always about *stale state left on disk (or in a database) from a previous run*, not about the current run's logic being wrong in isolation. Whenever you build an incremental system, explicitly ask yourself: *"if this run finds zero new rows, what does the downstream step see, and is that accurate?"*

### Step 2.4 — Silver: append instead of overwrite

```python
silver_path = Path("output/silver/your_table")
write_mode = "append" if silver_path.exists() else "overwrite"
write_parquet(silver_increment_df, silver_path, mode=write_mode)
```

First run ever: the folder doesn't exist yet, so it's created fresh (`overwrite` on nothing is just "create"). Every run after: `append` adds the new rows onto the existing accumulated history instead of replacing it.

### Step 2.5 — Gold: read full Silver history, recompute from scratch

This is the one part that's *not* incremental, and that's a deliberate, reasonable choice for a first version:

```python
silver_full = spark.read.parquet(str(silver_path))   # ALL history, not just this run's increment
gold_df = build_monthly_spending(silver_full)
write_parquet(gold_df, gold_path, mode="overwrite")   # Gold is always fully overwritten
```

**Why not make Gold incremental too?** Because Gold tables are aggregates — "total spending per month" can change even for old months if new data or corrections were appended (unlikely here, but possible in general), so recomputing from the full Silver history and overwriting is the *simple and correct* choice. Making Gold truly incremental (updating only the affected aggregate rows) requires "upsert" logic — an operation like Delta Lake's `MERGE` — which is meaningfully harder and is usually a cloud/Databricks-era upgrade, not a first-version requirement. Know that this shortcut exists and why we took it; don't feel you have to solve it on day one.

### Step 2.6 — Testing your incremental + idempotent pipeline for real

Don't just read the code and assume it's right — prove it with a real test:

```bash
# 1. Run it once (first run — full export, since no watermark exists yet)
python run_pipeline.py --export

# 2. Run it again immediately, with nothing new in Postgres
python run_pipeline.py --export
# Expect: Silver's total row count is UNCHANGED between run 1 and run 2.
# If it grew, you have a duplication bug.

# 3. Insert exactly one new test row directly into Postgres (via psql or your DB tool)
# 4. Run it a third time
python run_pipeline.py --export
# Expect: Silver's total row count grew by EXACTLY 1, and Gold reflects it.
```

This three-step test (baseline → run with nothing new → run with exactly one new thing) is the standard way to prove both incrementality and idempotency at once. Keep this test in your back pocket for any pipeline you build.

---

## Part 3 — Orchestration with Apache Airflow

### Step 3.1 — Why orchestration, and what it actually is

Running `python export_operational.py && python run_pipeline.py` by hand works, but only while a human is there to type it. A real pipeline needs something that can:
1. Run on a schedule (e.g., every night at 2 AM), with nobody watching.
2. Run steps in the correct order, and only start step 2 if step 1 actually succeeded.
3. Retry automatically if something transient fails.
4. Keep a history: what ran, when, did it succeed, how long did it take.

Apache Airflow is software that does exactly this — nothing more mystical than that. Airbnb built it in 2014; it's now maintained by the Apache Foundation and is one of the most widely used tools for this job in the industry.

**Core concept: the DAG.** DAG = "Directed Acyclic Graph" — a fancy name for "a description of which steps run in what order." You write a small Python file declaring your tasks and their dependencies; Airflow's own engine handles the actual running, retrying, and history-keeping.

### Step 3.2 — Why Airflow needs its *own*, separate virtual environment

Airflow pins very specific versions of its own dependencies (via an official "constraints file") and will happily change package versions in whatever environment you install it into to satisfy those pins. If you install Airflow into the *same* venv your pipeline already uses, it can silently downgrade or upgrade a package your pipeline depends on, breaking a previously-working pipeline for a reason that has nothing to do with your own code.

**The fix: two completely separate virtual environments.**
- `.venv` — your pipeline's environment (PySpark, pandas, sqlalchemy, etc.) — unchanged from Part 1/2.
- `.venv-airflow` — Airflow's own environment, installed separately, that never touches your pipeline's packages.

Airflow doesn't need to *import* your pipeline code to orchestrate it — it just runs your existing scripts as ordinary subprocesses, the same as if you typed the command yourself in a terminal. So the two environments never need to interact directly.

### Step 3.3 — Checking your Python version compatibility (do this BEFORE installing)

Airflow is deliberately conservative about which Python versions it supports — it typically lags a year or more behind the newest Python release. Before installing anything, check what constraints file actually exists for your Python version and the Airflow version you want:

```bash
python3 --version
```

Then check (replace `X.Y` with your major.minor version, e.g. `3.12`, and `A.B.C` with the Airflow version you're considering, e.g. `2.10.4`):

```bash
curl -s -o /dev/null -w "%{http_code}\n" \
  "https://raw.githubusercontent.com/apache/airflow/constraints-A.B.C/constraints-X.Y.txt"
```

`200` means that combination is supported. `404` means it isn't — either use an older Python for this venv, or a newer Airflow version. (We hit exactly this: our default Python was a brand-new version unsupported by the Airflow release we first tried; we had to check several Airflow versions until we found one whose constraints file existed for our Python version. Check this up front and save yourself the trial and error.)

### Step 3.4 — Creating the second virtual environment

```bash
python3 -m venv .venv-airflow
.venv-airflow/bin/python --version    # confirm it's the Python version you expect
```

If you have multiple Python versions installed on your machine and need a *specific* one (not whatever `python3` defaults to), point directly at that interpreter's full path instead, e.g.:

```bash
/path/to/specific/python3.12 -m venv .venv-airflow
```

### Step 3.5 — Installing Airflow with its constraints file

```bash
AIRFLOW_VERSION=2.10.4   # whatever version you confirmed works with your Python in Step 3.3
PYTHON_VERSION="$(.venv-airflow/bin/python -c 'import sys; print(f"{sys.version_info.major}.{sys.version_info.minor}")')"
CONSTRAINT_URL="https://raw.githubusercontent.com/apache/airflow/constraints-${AIRFLOW_VERSION}/constraints-${PYTHON_VERSION}.txt"

.venv-airflow/bin/pip install "apache-airflow==${AIRFLOW_VERSION}" --constraint "${CONSTRAINT_URL}"
```

**Why the constraints file matters:** without it, pip picks the newest version of each of Airflow's ~90 dependencies independently, and "newest of everything, all at once" reliably breaks Airflow. The constraints file is Apache's own tested list of exact versions that actually work together.

This install pulls a lot of packages and can take a few minutes — that's normal, not a sign anything is wrong.

### Step 3.6 — A quick glossary before we go further

- **DAG** — a named workflow: a set of tasks and the order they must run in.
- **Task** — one unit of work inside a DAG (e.g., "run the export script").
- **Operator** — the *type* of task. `BashOperator` just runs a shell command — the simplest kind, and what we used, because it lets Airflow run your existing scripts exactly as-is, with zero rewriting.
- **Scheduler** — the process that decides *when* a DAG's next run should start, based on its schedule.
- **DAG Processor** — a separate process (in Airflow 3+) that scans your DAG folder, parses the Python files, and registers what it finds into Airflow's own database. Without this running, nothing you save into the DAGs folder is ever picked up.
- **Triggerer** — handles a more advanced feature (deferred/async tasks) — not something you need to think about for a simple pipeline, but it's part of the standard set of processes Airflow runs.
- **API Server** — serves the web UI (and API) you view in your browser.
- **AIRFLOW_HOME** — a folder where Airflow keeps its own internal database, config, and logs — separate from your actual pipeline code.

### Step 3.7 — Setting AIRFLOW_HOME and initializing Airflow's database

```bash
export AIRFLOW_HOME="$(pwd)/data-pipeline/airflow_home"
export AIRFLOW__CORE__LOAD_EXAMPLES=false
.venv-airflow/bin/airflow db migrate
```

- `AIRFLOW_HOME` — we deliberately point this *inside* our project folder (not the default `~/airflow` in your home directory) so everything related to this project stays together.
- `AIRFLOW__CORE__LOAD_EXAMPLES=false` — turns off Airflow's ~20 built-in tutorial DAGs, so your own DAG isn't buried in sample clutter.
- `airflow db migrate` — creates Airflow's own internal tracking database (by default, a simple SQLite file — fine for learning/local use) and builds its tables. One-time setup, similar in spirit to your pipeline's Postgres tables existing, except this database belongs to Airflow itself, not your data.

**Important:** these two `export` lines only last for your current terminal session. Close the terminal, and you'll need to run them again before using Airflow next time.

### Step 3.8 — Creating your DAG file

```bash
mkdir -p data-pipeline/airflow_home/dags
```

Airflow watches this exact folder (by default, `$AIRFLOW_HOME/dags`) and treats every Python file inside it as a potential workflow. It doesn't run the file like a script — it *imports* it and looks for a DAG object.

```python
# data-pipeline/airflow_home/dags/your_pipeline_dag.py
from datetime import datetime
from airflow.sdk import DAG
from airflow.providers.standard.operators.bash import BashOperator

PROJECT_ROOT = "/full/path/to/your-project"
DATA_PIPELINE_DIR = f"{PROJECT_ROOT}/data-pipeline"
VENV_PYTHON = f"{PROJECT_ROOT}/.venv/bin/python"   # your ORIGINAL pipeline venv, not .venv-airflow

with DAG(
    dag_id="your_pipeline",
    start_date=datetime(2026, 1, 1),
    schedule="0 2 * * *",     # cron syntax: every day at 2 AM
    catchup=False,
    tags=["your-project"],
) as dag:

    export_task = BashOperator(
        task_id="export_from_postgres",
        bash_command=f'cd "{DATA_PIPELINE_DIR}" && "{VENV_PYTHON}" -m local.export_operational',
    )

    run_pipeline_task = BashOperator(
        task_id="run_bronze_silver_gold",
        bash_command=f'cd "{DATA_PIPELINE_DIR}" && "{VENV_PYTHON}" -m local.run_pipeline',
    )

    export_task >> run_pipeline_task   # the arrow = "this must succeed before that starts"
```

Piece by piece:
- `from airflow.sdk import DAG` — in Airflow 3.x, this import lives under `airflow.sdk` (older tutorials you find online, written for Airflow 2.x, will show `from airflow import DAG` instead — check which major version you actually installed).
- `VENV_PYTHON` points at your **original pipeline venv**, not `.venv-airflow`. Airflow just launches this path as a subprocess — same as you typing it into a terminal — it doesn't need to "activate" anything.
- `schedule="0 2 * * *"` — standard cron syntax (minute, hour, day-of-month, month, day-of-week). `0 2 * * *` = minute 0, hour 2, every day = 2:00 AM daily.
- `catchup=False` — without this, Airflow tries to "backfill" every scheduled run between `start_date` and today the moment you turn the DAG on. `False` means "just run going forward."
- `export_task >> run_pipeline_task` — Airflow's dependency syntax. Literally reads as an arrow: task on the left must succeed before the task on the right starts.

### Step 3.9 — The import-path gotcha we actually hit (important, easy to repeat by accident)

Our first version of the DAG ran the scripts like this:

```python
bash_command=f'cd "{DATA_PIPELINE_DIR}/local" && "{VENV_PYTHON}" export_operational.py'
```

This failed with `ModuleNotFoundError: No module named 'common'`.

**Why:** `export_operational.py` does `from common.config import ...` — it expects `common` to be an importable top-level package. Python only finds a package if the folder *containing* it is on its import search path. Running `python export_operational.py` directly from inside `local/` puts `local/` on the path, not `data-pipeline/` (the folder that actually contains both `common/` and `local/`).

**The fix:** run it as a *module*, from `data-pipeline/` itself, using the `-m` flag:

```python
bash_command=f'cd "{DATA_PIPELINE_DIR}" && "{VENV_PYTHON}" -m local.export_operational'
```

`-m` tells Python "treat this as a module inside a package, and add my current folder to the import path" — which is what actually makes `common` and `local` resolve correctly as sibling packages. **General lesson:** whenever your project has this "sibling packages that import each other" layout, always run scripts with `-m` from the folder that contains both packages — never `cd` into the innermost folder and run the file directly.

### Step 3.10 — Running Airflow for real: `airflow standalone`

Airflow normally needs three-plus separate long-running processes (Scheduler, DAG Processor, API Server, Triggerer) running simultaneously. For local learning, Airflow ships a shortcut that starts all of them in one terminal:

```bash
source .venv-airflow/bin/activate    # IMPORTANT — see note below
export AIRFLOW_HOME="$(pwd)/data-pipeline/airflow_home"
export AIRFLOW__CORE__LOAD_EXAMPLES=false
airflow standalone
```

**Why `source .venv-airflow/bin/activate` first, instead of calling `.venv-airflow/bin/airflow standalone` directly:** `standalone` needs to launch several more copies of itself as background processes, and it does that by running the bare word `airflow` — trusting your terminal's PATH to know where that is. If you call it by full path without activating the environment first, those inner launches fail with `FileNotFoundError: 'airflow'`, because your PATH doesn't know where `airflow` lives. Activating first fixes this for the whole session.

This process **keeps running continuously** in that terminal — that's expected, not a hang. Leave it running; open a *second* terminal window for anything else.

On startup, it will print an admin password (once) and eventually a line like `Airflow is ready`, along with the web UI address — usually `http://localhost:8080`. Log in with username `admin` and that generated password (also saved in `$AIRFLOW_HOME/simple_auth_manager_passwords.json.generated` if you lose it).

### Step 3.11 — Using the web UI

1. Open `http://localhost:8080`, log in.
2. Click **Dags** in the sidebar — your DAG should be listed, initially **paused** (a safety default — new DAGs never run automatically the instant they're registered).
3. Click the toggle switch to **unpause** it.
4. Click the **▶ play icon** to manually trigger a run immediately, instead of waiting for the schedule.
5. Click into the run to see each task's state (queued → running → success/failed) update live.
6. If a task fails, click into it, go to its log, and scroll to the actual traceback near the bottom — that's where the real error is, underneath a lot of Airflow's own boilerplate logging.

### Step 3.12 — Two real problems we hit at this stage, and how we diagnosed them

**Problem A: Postgres itself wasn't running.**
Symptom in the task log: `Connection refused` when trying to reach `localhost:5432`. This is a *different* kind of error than a wrong password or wrong database name — "connection refused" specifically means nothing is even listening on that port. Diagnosis path:
```bash
brew services list | grep postgres
```
If it shows `error` (not just `stopped`), something failed to start previously. Check *why*:
```bash
tail -50 /opt/homebrew/var/log/postgresql@<version>.log
```

**Problem B: a stale lock file after an unclean shutdown.**
The log showed: `FATAL: lock file "postmaster.pid" already exists ... Is another postmaster (PID <N>) running?`. This happens when Postgres doesn't get a clean shutdown (e.g., the machine restarts or loses power) — it leaves behind a "someone is already running" marker file that never got cleaned up. Diagnosis: check if that PID is actually still Postgres:
```bash
ps -p <N>
```
If that PID now belongs to a completely unrelated process (or doesn't exist at all), the lock file is stale and safe to remove:
```bash
rm /opt/homebrew/var/postgresql@<version>/postmaster.pid
brew services restart postgresql@<version>
```
**General lesson:** "lock file already exists" after any unclean shutdown (crash, forced restart, power loss) is common and not something to panic about — always verify the referenced PID is actually still running that service before deleting the lock file, don't delete it blindly if the service might genuinely still be up.

Once both were fixed, re-triggering the DAG in the UI ran both tasks to **Success**, end to end, with zero manual terminal commands beyond starting `airflow standalone` itself.

---

## Part 4 — Full Command Reference (cheat sheet, in order)

```bash
# --- Part 1: pipeline setup ---
python3 -m venv .venv
source .venv/bin/activate
pip install pyspark pandas sqlalchemy python-dotenv pg8000

# --- Part 1: running the pipeline manually ---
cd data-pipeline/local
python export_operational.py
python run_pipeline.py

# --- Part 2: testing incremental + idempotent behavior ---
python run_pipeline.py --export      # run 1: baseline
python run_pipeline.py --export      # run 2: nothing new -> row count unchanged
# (insert 1 test row into Postgres here)
python run_pipeline.py --export      # run 3: row count grows by exactly 1

# --- Part 3: Airflow environment setup ---
python3 --version
curl -s -o /dev/null -w "%{http_code}\n" \
  "https://raw.githubusercontent.com/apache/airflow/constraints-<AIRFLOW_VER>/constraints-<PY_VER>.txt"
python3 -m venv .venv-airflow
AIRFLOW_VERSION=<version>
PYTHON_VERSION="$(.venv-airflow/bin/python -c 'import sys; print(f"{sys.version_info.major}.{sys.version_info.minor}")')"
CONSTRAINT_URL="https://raw.githubusercontent.com/apache/airflow/constraints-${AIRFLOW_VERSION}/constraints-${PYTHON_VERSION}.txt"
.venv-airflow/bin/pip install "apache-airflow==${AIRFLOW_VERSION}" --constraint "${CONSTRAINT_URL}"

# --- Part 3: Airflow initialization (repeat the two export lines every new terminal session) ---
export AIRFLOW_HOME="$(pwd)/data-pipeline/airflow_home"
export AIRFLOW__CORE__LOAD_EXAMPLES=false
.venv-airflow/bin/airflow db migrate
mkdir -p data-pipeline/airflow_home/dags
# (write your DAG .py file into that dags/ folder)

# --- Part 3: running Airflow ---
source .venv-airflow/bin/activate
export AIRFLOW_HOME="$(pwd)/data-pipeline/airflow_home"
export AIRFLOW__CORE__LOAD_EXAMPLES=false
airflow standalone
# open http://localhost:8080, log in, unpause + trigger the DAG in the browser

# --- troubleshooting: Postgres won't start ---
brew services list | grep postgres
tail -50 /opt/homebrew/var/log/postgresql@<version>.log
ps -p <PID from lock file error>
rm /opt/homebrew/var/postgresql@<version>/postmaster.pid   # only if that PID is confirmed stale
brew services restart postgresql@<version>
```

---

## Part 5 — Concepts Glossary (quick lookup)

| Term | Meaning |
|---|---|
| ETL | Extract, Transform, Load — the general pattern of moving data out of a source system, cleaning it, and loading it somewhere analysis-friendly. Predates the cloud by decades. |
| Medallion architecture | The Bronze (raw) → Silver (clean) → Gold (aggregated) layering pattern. |
| Idempotent | Safe to re-run any number of times without changing the result beyond the first successful run. |
| Incremental load | Only processing data that's new since the last successful run, instead of everything every time. |
| Watermark | A saved "last processed up to here" marker, used to make loading incremental. |
| Full reload | Reprocessing 100% of the source data every run — simple, but doesn't scale. |
| Parquet | A columnar binary file format, standard for analytical data (fast, typed, compressible) — contrast with CSV, which is plain text. |
| DAG | Directed Acyclic Graph — Airflow's term for "a workflow: a set of tasks with a defined order, no loops." |
| Operator | The type of a single Airflow task (e.g. `BashOperator` = "run a shell command"). |
| Scheduler | The Airflow process deciding when a DAG's next run should fire. |
| DAG Processor | The Airflow process that scans and registers DAG files from your dags folder. |
| Catchup | Whether Airflow tries to "replay" every missed scheduled run between `start_date` and now when a DAG is first turned on. Almost always want this `False` for a fresh DAG. |
| Constraints file | Apache's own list of exact, tested-together dependency versions for a specific Airflow + Python version combination. |
| `-m` flag (Python) | Runs a file as a module inside its package, adding the current folder to the import search path — needed whenever sibling packages import each other. |
| `postmaster.pid` | Postgres's own "I am currently running" lock file; can go stale after an unclean shutdown and needs manual removal if so. |

---

## Part 6 — What I'd Do Differently / Watch For, Doing This on a New Project

- Check your Python version against Airflow's supported versions **before** creating any venv or attempting an install — saves a lot of back-and-forth.
- Always create Airflow in its **own** venv, separate from your pipeline's venv, from the very start.
- When writing a script that imports sibling packages (`common`, `local`, etc.), always run it with `-m` from the parent folder — decide this up front rather than discovering the `ModuleNotFoundError` later.
- Before building anything incremental, ask explicitly: *"if a run finds zero new rows, what does every downstream step see — and is that actually correct?"* — that question alone would have caught our idempotency bug before it was ever written.
- Test incremental + idempotent behavior with the three-run test (baseline → nothing new → exactly one new thing) every time, not just once — it's cheap and catches real bugs.
- If Postgres (or any local service) won't start after a machine restart, check for a stale lock file before assuming something is seriously broken.

---

*End of notes. Next planned step from here: moving this same pattern — Bronze/Silver/Gold, incremental loading, Airflow orchestration — onto Azure (Data Lake Storage, Databricks, Delta Lake, Azure Data Factory).*
