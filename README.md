# Databricks Model Serving metrics collector

This project collects the supported Prometheus/OpenMetrics export from
Databricks Model Serving endpoints and stores timestamped history in Delta.
The setup notebook creates a continuous serverless Lakeflow Job and publishes
an AI/BI dashboard over the collected metrics.

The collector:

- Discovers non-foundation serving endpoints from the workspace API.
- Optionally restricts collection to an explicit endpoint include list.
- Polls metrics-capable endpoints on aligned one-minute boundaries.
- Skips Foundation Model API endpoints backed by `system.ai.*`.
- Uses source timestamps and Delta `MERGE` operations for idempotent retries.
- Stores raw exposition text, normalized samples, endpoint snapshots, rates,
  histogram percentiles, and dashboard-ready series.
- Reads configured inference tables directly from Unity Catalog. It does not
  copy inference payload tables or enable payload logging.

The public metrics export is a current-snapshot API. It does not expose the
historical range endpoint used internally by the Databricks UI. The continuous
collector retains each published snapshot so the dashboard can query history.

## Repository contents

- `model_serving_metrics_setup.ipynb` configures the continuous job and
  publishes the dashboard.
- `model_serving_metrics_collector.ipynb` performs continuous collection and
  maintains the Delta tables and views.
- `databricks.yml` syncs the notebooks through a Declarative Automation Bundle.
- `.env.example` contains local CLI configuration placeholders.

## Requirements

- A Databricks workspace with Unity Catalog.
- Databricks CLI authentication.
- Permission to list and read the selected Model Serving endpoints.
- `USE CATALOG`, `USE SCHEMA`, and `CREATE` privileges for the target catalog
  and schema.
- Permission to create and manage Lakeflow Jobs and AI/BI dashboards.
- `CAN USE` permission on a serverless or Pro SQL warehouse.
- Databricks Assistant enabled if the dashboard's built-in Genie experience is
  required.

## Configure the notebooks

Edit the first code cell in `model_serving_metrics_setup.ipynb`.

`INCLUDE_ENDPOINT_NAMES`

: Optional endpoint include list. Leave it empty to discover all
  non-foundation endpoints. The collector refreshes endpoint discovery hourly.

`STORAGE_CATALOG`

: Unity Catalog catalog for collected metrics. It defaults to the notebook's
  current catalog.

`STORAGE_SCHEMA`

: Schema created for tables and views. The default is
  `model_serving_observability`.

`JOB_NAME`

: Name of the continuous Lakeflow Job.

`DASHBOARD_NAME`

: Name of the AI/BI dashboard. Re-running setup updates the existing dashboard
  with this name.

`COLLECTOR_NOTEBOOK_NAME`

: Name of the collector notebook. Keep both notebooks in the same workspace
  folder when changing this value.

`COLLECTION_INTERVAL_SECONDS`

: Poll interval for metrics-capable endpoints. The default is 60 seconds.

`COLLECTION_PAUSE_STATUS`

: `UNPAUSED` starts or keeps the continuous job running. `PAUSED` creates or
  updates it without active collection.

`WAREHOUSE_ID`

: SQL warehouse used by the dashboard. Leave empty to select the first
  available warehouse.

`PARENT_PATH`

: Workspace folder for a newly created dashboard. Leave empty to use the setup
  notebook's folder.

`COLLECTOR_PERFORMANCE_TARGET`

: Serverless performance target. `STANDARD` minimizes cost. Use
  `PERFORMANCE_OPTIMIZED` only when faster startup is more important.

`COLLECTOR_ENVIRONMENT_KEY`

: Internal key for the job's serverless Python environment.

`COLLECTOR_ENVIRONMENT_VERSION`

: Databricks serverless environment version.

`COLLECTOR_DEPENDENCIES`

: Python requirements installed in the collector environment.

`DASHBOARD_LOOKBACK_HOURS`

: Default rolling window used by dashboard datasets.

## Deploy with the Databricks CLI

Copy the local environment template and set an authenticated CLI profile:

```bash
cp .env.example .env
databricks auth login --profile my-workspace
```

Edit `.env`:

```dotenv
DATABRICKS_CONFIG_PROFILE=my-workspace
BUNDLE_TARGET=dev
```

Load the values, validate, and deploy:

```bash
set -a
source .env
set +a

databricks bundle validate \
  --target "$BUNDLE_TARGET" \
  --profile "$DATABRICKS_CONFIG_PROFILE"

databricks bundle deploy \
  --target "$BUNDLE_TARGET" \
  --profile "$DATABRICKS_CONFIG_PROFILE"
```

Run the setup notebook as a one-time serverless task:

```bash
BUNDLE_FILE_PATH="$(
  databricks bundle summary \
    --target "$BUNDLE_TARGET" \
    --profile "$DATABRICKS_CONFIG_PROFILE" \
    --output json |
  jq -r '.workspace.file_path'
)"

SETUP_NOTEBOOK_PATH="${BUNDLE_FILE_PATH}/model_serving_metrics_setup"

databricks jobs submit \
  --profile "$DATABRICKS_CONFIG_PROFILE" \
  --timeout 30m \
  --json "$(
    jq -n \
      --arg path "$SETUP_NOTEBOOK_PATH" \
      '{
        run_name: "Configure Model Serving metrics",
        tasks: [{
          task_key: "configure_metrics",
          notebook_task: {
            notebook_path: $path,
            source: "WORKSPACE"
          }
        }]
      }'
  )"
```

The setup notebook is idempotent. Re-running it updates the named job and
dashboard instead of creating duplicates.

## Setup validation and output

Setup prints one JSON status record per step directly in notebook output. The
records include configured values, resolved endpoints, skipped foundation
endpoints, inference-table state, workspace paths, SQL warehouse selection, job
state, query validation, and dashboard publication.

An empty `INCLUDE_ENDPOINT_NAMES` value remains an empty job parameter. This is
intentional. The continuous collector owns hourly endpoint discovery. A
non-empty include list is passed unchanged after setup verifies that every name
exists, is not a Foundation Model API endpoint, and supports metrics export.

Setup stops immediately when it detects:

- Unknown or foundation endpoint names in the include list.
- Missing collector notebooks or dashboard parent folders.
- Missing catalogs, schema creation failures, or unavailable SQL warehouses.
- Duplicate jobs or dashboards with the configured name.
- Invalid collection controls or empty runtime dependencies.
- Collector startup failures.
- Invalid dashboard dataset queries.

## Deploy by uploading notebooks

1. In the Databricks workspace, create a folder in Workspace.
2. Import `model_serving_metrics_setup.ipynb`.
3. Import `model_serving_metrics_collector.ipynb` into the same folder.
4. Open the setup notebook.
5. Edit its first code cell.
6. Select **Run all**.

The setup notebook creates or updates the continuous job and dashboard. The
collector notebook should normally run only through that job.

## Delta objects

The collector creates these objects under the configured catalog and schema:

- `model_serving_endpoint_snapshots`
- `model_serving_metrics_raw`
- `model_serving_metric_samples`
- `model_serving_counter_rates`
- `model_serving_histogram_percentiles`
- `model_serving_dashboard_series`

The first three are Delta tables. The remaining objects are views.

## Operational behavior

The collector runs continuously and uses the API-provided sample timestamp as
the event time. Multiple polls that return the same source timestamp update the
same Delta rows. This prevents duplicate samples after retries or overlapping
polls.

At startup, the collector probes all included or discovered non-foundation
endpoints. It remembers endpoints that successfully expose metrics and polls
them every minute. It refreshes workspace endpoint discovery hourly.

Some endpoint types do not export every metric family. Charts remain empty
until their source metric is emitted. Inference-table dashboard pages are added
only when the endpoint API reports an enabled inference table and the payload
table exists.

Continuous serverless collection incurs ongoing compute cost. Pause the job
when minute-level history is not required.
