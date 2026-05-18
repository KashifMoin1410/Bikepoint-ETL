# Bikepoint ETL

End-to-end Databricks pipeline for [TfL BikePoint](https://api.tfl.gov.uk/BikePoint/) station data. Raw API snapshots are ingested into Bronze, cleaned and typed in Silver, aggregated into business-ready Gold tables, validated with data-quality checks, and visualised in a Lakeview dashboard.

## Architecture

Medallion-style layering on Delta Lake in the `workspace.bikepoint` schema:

```text
TfL BikePoint API
        │
        ▼
┌─────────────────┐
│ Bronze (raw)    │  Append-only snapshots, JSON preserved
└────────┬────────┘
         ▼
┌─────────────────┐
│ Silver (clean)  │  Incremental MERGE, parsed & typed
└────────┬────────┘
         ▼
┌─────────────────┐
│ Gold (business) │  Full overwrite, station-level metrics
└────────┬────────┘
         ▼
┌─────────────────┐
│ Data quality    │  Pass/fail checks across all layers
└─────────────────┘
```

| Layer   | Table(s) | Write mode |
|---------|----------|------------|
| Bronze  | `bronze_bikepoint_raw` | Append |
| Silver  | `silver_bikepoint` | Incremental MERGE |
| Gold    | `gold_bikepoint_station_current_status`, `gold_bikepoint_station_aggregated_metrics` | Full overwrite |

**Source:** `https://api.tfl.gov.uk/BikePoint/`

## Project structure

```text
Bikepoint-ETL/
├── README.md
├── .gitignore
├── notebooks/
│   ├── bronze/
│   │   └── 01_bronze_ingestion.ipynb
│   ├── silver/
│   │   └── 02_silver_transformation.ipynb
│   ├── gold/
│   │   └── 03_gold_aggregation.ipynb
│   └── quality/
│       └── 04_data_quality.ipynb
└── dashboards/
    └── bikepoint_analysis.lvdash.json
```

## Running the pipeline

The same four notebooks run in sequence for every execution:

```text
01_bronze_ingestion → 02_silver_transformation → 03_gold_aggregation → 04_data_quality
```

| Step | Layer | Action |
|------|-------|--------|
| 1 | Bronze | Fetch API snapshot and append to `bronze_bikepoint_raw` |
| 2 | Silver | Parse JSON, dedupe, and MERGE into `silver_bikepoint` |
| 3 | Gold | Build current-status and aggregated-metrics tables |
| 4 | Data quality | Run all checks; fail the run if any check fails |

### Manual execution

Run each notebook in order from the workspace (Repos or imported paths under `notebooks/`).

### Scheduled execution (Databricks Job)

Production runs are handled by a Databricks Job that chains the four notebooks as sequential tasks.

| Setting | Value |
|---------|--------|
| **Job ID** | `917460087260830` |
| **Run as** | `kashifmoin.1704@gmail.com` |
| **Schedule** | Every 6 hours |
| **Compute** | Serverless |
| **Environment** | `bronze_ingestion_environment` (version 5) |
| **Performance optimized** | On |
| **Maximum concurrent runs** | 1 |
| **Queue** | Enabled |
| **Duration warning** | 15 minutes |
| **Duration timeout** | 30 minutes |
| **Notifications** | `kashifmoin.1704@gmail.com` — on failure and duration warning |
| **Git** | Not configured |

To trigger a run manually from the UI: open the job in **Workflows → Jobs**, then click **Run now**.

## Dashboard

Import `dashboards/bikepoint_analysis.lvdash.json` into Databricks Lakeview. It queries the Gold tables for network KPIs, risk distribution, and station-level analysis.

## Prerequisites

- Databricks workspace with Unity Catalog (or equivalent) access to catalog `workspace`
- Network access to the TfL BikePoint API from the cluster
- Schema: `workspace.bikepoint` (created by the Bronze notebook if missing)

## Getting started

1. Clone this repository and sync it to your Databricks workspace (Repos or workspace import).
2. Either run the notebooks manually (see [Manual execution](#manual-execution)) or use the existing Databricks Job (see [Scheduled execution](#scheduled-execution-databricks-job)).
3. Import `dashboards/bikepoint_analysis.lvdash.json` into Lakeview for operational reporting.

For a first-time setup, run Bronze once manually to create the `workspace.bikepoint` schema and seed the raw table before relying on the scheduled job.

## Design notes

- **Bronze** is append-only so history can be rebuilt and Silver/Gold reprocessed without re-calling the API.
- **Silver** uses an anti-join on `bikepoint_id + pipeline_run_id` to process only new Bronze rows.
- **Gold** is fully rebuilt each run because tables are small and depend on full Silver history.
- **Data quality** runs every check before raising, so failures surface in one summary.
