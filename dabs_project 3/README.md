# Orders Medallion Pipeline — Databricks Asset Bundle

A self-contained Bronze → Silver → Gold pipeline built as a Databricks Asset Bundle (DAB),
designed to run on **Databricks Free Edition** using **Serverless compute**.

Unlike most tutorial projects, this one needs **no manual file upload** — the first task
pulls directly from a Databricks-provided sample dataset (`samples.tpch.orders`) and lands
it into your Unity Catalog Volume as CSV, simulating a real "raw file drop." Everything
downstream builds from there.

## Pipeline stages

| Task | Notebook | What it does |
|---|---|---|
| `land_to_volume` | `00_land_to_volume.ipynb` | Reads `samples.tpch.orders` (built into every Databricks workspace) and writes it out as raw CSV into your Volume — simulating an incoming file drop |
| `bronze` | `01_bronze.ipynb` | Reads the raw CSV from the Volume, adds an `ingestion_time` column, writes to `{env}_bronze.orders` as a Delta table |
| `silver` | `02_silver.ipynb` | Cleans bronze: drops null order keys, drops non-positive totals, standardizes `o_orderstatus` casing, dedupes on `o_orderkey`. Writes `{env}_silver.orders` |
| `gold` | `03_gold.ipynb` | Aggregates silver into a business-ready summary: order count and total revenue by year and status. Writes `{env}_gold.orders_summary` |

All four run as one Databricks **Job** with sequential dependencies (`land_to_volume` → `bronze` → `silver` → `gold`).

## Setup

### 1. Unzip and open in VS Code
```bash
unzip orders_medallion_pipeline.zip
cd orders_medallion_pipeline
code .
```

### 2. Confirm the Databricks CLI is installed and authenticated
```bash
databricks -v
databricks auth login --host "https://<your-workspace-url>"
databricks auth profiles
```

### 3. Update `databricks.yml` if needed
The `workspace.host` values are pre-filled with a placeholder — update both the `dev` and
`prod` target hosts if your workspace URL is different.

### 4. Update the Volume path if needed
Both `resources/orders_pipeline_job.yml` and the notebooks default to:
```
/Volumes/delta_lake_catalog/default/deltalake_volume
```
If your catalog/schema/volume names differ, update the `volume_path` value in
`resources/orders_pipeline_job.yml` (only needs changing in one place — it's passed as a
parameter into the notebooks from there).

### 5. Deploy the bundle
```bash
databricks bundle deploy -p "<your-profile-name>" --target dev
```

### 6. Run the pipeline
```bash
databricks bundle run orders_medallion_pipeline -p "<your-profile-name>" --target dev
```

### 7. Check the results
In the Databricks UI:
- **Workflows** → you should see `orders_medallion_pipeline` with 4 tasks and a run graph
- **Catalog** → look for `dev_bronze.orders`, `dev_silver.orders`, `dev_gold.orders_summary`

## Why there's no `artifacts:` block in `databricks.yml`
This project is notebooks-only — no Python package to build into a wheel. Including a wheel
build step (as the default DAB Python template does) causes a
`ValueError: Unable to determine which files to ship inside the wheel` error, since there's
nothing for Hatchling to package. If you later add real Python modules (not just notebooks),
that's when an `artifacts:` block becomes relevant again.

## Why no cluster is defined in the job
Each task in `resources/orders_pipeline_job.yml` deliberately has no `new_cluster`,
`existing_cluster_id`, or `job_cluster_key` specified. Leaving compute unspecified lets the
job run on **Serverless compute** — the default (and typically the only available option)
on Free Edition workspaces.

## Extending this project
- Add a `tests/` folder with `pytest` checks against the silver/gold tables (row counts, no
  nulls, no negative totals) — a natural next step once the pipeline runs cleanly.
- Swap `samples.tpch.orders` for `samples.nyctaxi.trips` or another built-in sample to
  practice the same pattern on different data shapes.
- Add a scheduled trigger to `resources/orders_pipeline_job.yml` (a `schedule:` block) to
  make Bronze→Silver→Gold run automatically on a cron cadence.
