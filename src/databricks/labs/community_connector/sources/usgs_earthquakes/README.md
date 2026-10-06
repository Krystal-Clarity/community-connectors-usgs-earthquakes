# USGS Earthquakes Community Connector

## Quickstart

This quickstart assumes you have cloned the repository as described in the
blog. It follows the repository's connector package workflow: package the
source separately and deploy it with the shared framework.

Run the commands below from the **repository root**, not from this nested
source directory. If your terminal is elsewhere inside the clone, go to the
root with:

```bash
cd "$(git rev-parse --show-toplevel)"
```

Replace these placeholders with values from your Databricks workspace:

- `<PROFILE>`: Databricks CLI profile for your workspace.
- `<WORKSPACE_URL>`: workspace URL; only needed if you still need to authenticate.
- `<CATALOG>` and `<SCHEMA>`: existing Unity Catalog destination where you can
  create the pipeline output table and the CLI's package volume.
- `<CONNECTION_NAME>`: a name for the Unity Catalog connection.
- `<PIPELINE_NAME>`: a name for the Lakeflow Spark Declarative Pipeline.

The connector reads public earthquake data from the USGS FDSN Event Web
Service, so the connection needs no API key or other credentials. This
quickstart creates the `earthquakes` streaming table.

### Prerequisites

- Python 3.10 or later, available as `python`.
- Databricks CLI installed and authenticated to your workspace. If needed, set
  up authentication with:

  ```bash
  databricks auth login --host "<WORKSPACE_URL>" --profile "<PROFILE>"
  ```

- A Unity Catalog enabled workspace and an existing catalog/schema. You need
  permission to create the connection and pipeline, create a volume in the
  selected schema, and create the destination table. If you lack these
  permissions, ask your Databricks administrator to provision a destination
  schema and grant access.

Set the values used by the commands below. Use distinct connection and pipeline
names if you have already run this quickstart:

```bash
set -euo pipefail
export DATABRICKS_CONFIG_PROFILE="<PROFILE>"
export CATALOG="<CATALOG>"
export SCHEMA="<SCHEMA>"
export CONNECTION_NAME="<CONNECTION_NAME>"
export PIPELINE_NAME="<PIPELINE_NAME>"
```

### 1. Install the Community Connector CLI and build the packages

```bash
python -m pip install -e tools/community_connector
python -m pip install build
python -m build --wheel . --outdir dist/framework
python -m build --wheel \
  src/databricks/labs/community_connector/sources/usgs_earthquakes \
  --outdir dist/usgs_earthquakes

FRAMEWORK_WHEEL=$(find dist/framework -maxdepth 1 -name 'lakeflow_community_connectors-*.whl' -print -quit)
USGS_WHEEL=$(find dist/usgs_earthquakes -maxdepth 1 -name 'lakeflow_community_connectors_usgs_earthquakes-*.whl' -print -quit)
test -n "$FRAMEWORK_WHEEL"
test -n "$USGS_WHEEL"
```

The framework wheel provides the shared connector runtime. The USGS wheel
contains the connector and declares its `requests` dependency. The CLI uploads
both wheels to a Unity Catalog Volume when it creates the pipeline.

### 2. Create a Unity Catalog connection

```bash
community-connector create_connection usgs_earthquakes "$CONNECTION_NAME" \
  --options '{}' \
  --spec src/databricks/labs/community_connector/sources/usgs_earthquakes/connector_spec.yaml
```

### 3. Create the pipeline spec

The included example selects the `earthquakes` source table and sets its initial
lookback window to one day. Create a local spec and fill in your connection
name:

```bash
sed "s|<CONNECTION_NAME>|${CONNECTION_NAME}|g" \
  src/databricks/labs/community_connector/sources/usgs_earthquakes/pipeline_spec.example.yaml \
  > pipeline_spec.yaml
```

The connector uses USGS `updatedafter` on later reads so revised events can be
picked up. You can change the `table_configuration` options described below.

### 4. Create and start the pipeline

```bash
community-connector create_pipeline usgs_earthquakes "$PIPELINE_NAME" \
  --pipeline-spec pipeline_spec.yaml \
  --catalog "$CATALOG" \
  --schema "$SCHEMA" \
  --package "$FRAMEWORK_WHEEL" \
  --package "$USGS_WHEEL"

community-connector run_pipeline "$PIPELINE_NAME"
```

The CLI prints a pipeline URL. Open it to monitor the first update. Wait until
`community-connector show_pipeline "$PIPELINE_NAME"` reports the latest update
as `COMPLETED` before checking the table.

### 5. Verify the output

```bash
community-connector show_pipeline "$PIPELINE_NAME"
databricks tables get "$CATALOG.$SCHEMA.earthquakes" --profile "$DATABRICKS_CONFIG_PROFILE"
databricks experimental aitools tools query \
  "SELECT COUNT(*) AS event_count FROM $CATALOG.$SCHEMA.earthquakes" \
  --profile "$DATABRICKS_CONFIG_PROFILE"
```

The table should appear as a `STREAMING_TABLE` in Catalog Explorer at
`$CATALOG.$SCHEMA.earthquakes`, and the query should return the count of ingested
USGS events. Its columns include `id`, `mag`, `place`, `event_time`, and
`updated_at`.

## Connector overview

The connector exposes one production-style table and four demo variants. The
quickstart above deploys `earthquakes`; to use another variant, replace the
`source_table` in `pipeline_spec.yaml` with one of these names:

- `earthquakes`: CDC, the recommended table
- `earthquakes_snapshot`: snapshot ingestion
- `earthquakes_cdc`: CDC ingestion
- `earthquakes_cdc_with_deletes`: CDC with a delete flow; USGS does not publish
  deleted-event tombstones, so this flow returns no delete rows
- `earthquakes_append`: append-only ingestion

The connector reads USGS GeoJSON responses, flattens each Feature into a row,
and uses the event `updated` timestamp as the incremental cursor. On the first
run, it queries recent events using `starttime`. Later runs use USGS
`updatedafter` to read newly published and revised events.

Supported table options:

- `lookback_days`: initial query window in days; default `1`
- `limit`: maximum events per request; default `20000`
- `min_magnitude`: minimum earthquake magnitude; default `0.0`
- `event_type`: USGS event type, for example `earthquake`
- `min_latitude`, `max_latitude`, `min_longitude`, `max_longitude`: geographic
  bounding box

USGS references:

- [FDSN Event Web Service](https://earthquake.usgs.gov/fdsnws/event/1/)
- [GeoJSON feeds](https://earthquake.usgs.gov/earthquakes/feed/v1.0/geojson.php)
