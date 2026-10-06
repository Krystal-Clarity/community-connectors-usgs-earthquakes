# USGS Earthquakes Community Connector

## Quickstart

Replace each `<PLACEHOLDER>` below with a value from your Databricks workspace:

- `<CATALOG>`: Unity Catalog catalog where the pipeline and package volume can be created.
- `<SCHEMA>`: schema in that catalog for the pipeline output and uploaded packages.
- `<CONNECTION_NAME>`: a name for the Unity Catalog connection.
- `<PIPELINE_NAME>`: a name for the Lakeflow Spark Declarative Pipeline.

This connector reads public earthquake event data from the USGS FDSN Event Web
Service. It needs no API key or other credentials. The steps below build the
connector and framework wheels, create a `COMMUNITY` connection, and create a
pipeline that writes the `earthquakes` source table to a streaming table.

### Prerequisites

- A Databricks workspace with Unity Catalog and permission to create connections,
  volumes, and pipelines.
- Databricks CLI installed and authenticated to that workspace. Select the
  intended profile with `DATABRICKS_CONFIG_PROFILE` if it is not `DEFAULT`.
- Python 3.10 or later.

Run these commands from the root of this repository.

### 1. Install the Community Connector CLI and build the packages

```bash
python -m pip install -e tools/community_connector
python -m pip install build

python -m build --wheel . \
  --outdir dist/framework

python -m build --wheel \
  src/databricks/labs/community_connector/sources/usgs_earthquakes \
  --outdir dist/usgs_earthquakes
```

The framework wheel provides the shared connector runtime. The USGS wheel
contains this connector and declares its `requests` dependency. The CLI uploads
both wheels to a Unity Catalog Volume when it creates the pipeline.

### 2. Create a Unity Catalog connection

```bash
community-connector create_connection usgs_earthquakes <CONNECTION_NAME> \
  --options '{}' \
  --spec src/databricks/labs/community_connector/sources/usgs_earthquakes/connector_spec.yaml
```

The connection has no credentials because the USGS event API is public.

### 3. Define the pipeline spec

Copy the example spec into the repository root, then replace
`<CONNECTION_NAME>` with the connection name created in step 2:

```bash
cp src/databricks/labs/community_connector/sources/usgs_earthquakes/pipeline_spec.example.yaml pipeline_spec.yaml
```

`lookback_days` controls the initial query window. The connector uses USGS
`updatedafter` on later reads so revised events can be picked up. You can omit
`table_configuration` to use the connector defaults.

### 4. Create the pipeline

```bash
community-connector create_pipeline usgs_earthquakes <PIPELINE_NAME> \
  --pipeline-spec pipeline_spec.yaml \
  --catalog <CATALOG> \
  --schema <SCHEMA> \
  --package dist/framework/lakeflow_community_connectors-0.1.0-py3-none-any.whl \
  --package dist/usgs_earthquakes/lakeflow_community_connectors_usgs_earthquakes-0.1.0-py3-none-any.whl
```

The CLI creates the pipeline and uploads the two wheel dependencies to a
Unity Catalog Volume. It prints the pipeline URL when creation succeeds.

### 5. Start the pipeline and view the table

```bash
community-connector run_pipeline <PIPELINE_NAME>
```

Open the pipeline URL printed in the previous step to monitor the update. After
the first successful update, find the `earthquakes` streaming table in
`<CATALOG>.<SCHEMA>` in Catalog Explorer.

## Connector overview

The connector exposes one production-style table and four demo variants:

- `earthquakes`: CDC, the recommended table
- `earthquakes_snapshot`: snapshot ingestion
- `earthquakes_cdc`: CDC ingestion
- `earthquakes_cdc_with_deletes`: CDC with a delete flow; USGS does not publish
  deleted-event tombstones, so this flow returns no delete rows
- `earthquakes_append`: append-only ingestion

It reads USGS GeoJSON responses, flattens each Feature into a row, and uses the
event `updated` timestamp as the incremental cursor. On the first run, it
queries recent events using `starttime`. Later runs use USGS `updatedafter` to
read newly published and revised events.

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
