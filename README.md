# InfluxDB 3 Earthquake Sampler Plugin

A scheduled InfluxDB 3 Processing Engine plugin that ingests earthquake events and writes normalized points for dashboards and alerting.

## What this plugin does

On each scheduled run, the plugin:

- reads earthquake data from one of two source modes:
  - `source_type=http`:
    - USGS GeoJSON feeds (`feed=all_hour`, `significant_day`, `4.5_week`, etc.), or
    - a custom JSON endpoint (`source_url=...`) with parser `source_format=usgs_geojson|flat_json`
  - `source_type=influxdb_table`:
    - reads rows directly from an existing InfluxDB table (for example `usgs.quake`)
- normalizes each event into a common schema
- filters by minimum magnitude (`min_magnitude`)
- enforces per-run cap (`max_events`)
- deduplicates events using cached update markers (`skip_unchanged=true`)
- optionally copies an InfluxDB source table row-for-row without renaming columns or adding tags

## Files

- `earthquake_sampler.py` - plugin source code

## Key trigger arguments

- `source_type`: `http` (default) or `influxdb_table`
- `feed`: USGS feed key when using HTTP mode
- `source_url`: custom URL when using HTTP mode
- `source_format`: `usgs_geojson` or `flat_json`
- `source_table`: source table for `influxdb_table` mode (default `quake`)
- `source_query`: optional SQL override for table mode
- `lookback_minutes`: table query lookback window (default `15`)
- `measurement`: destination measurement (default `earthquakes`)
- `preserve_source_schema`: when `true` with `source_type=influxdb_table`, writes the source row's columns and types unchanged. Use this to make `earthquake_plugin_stats` match `quake` exactly.
- `min_magnitude`: minimum magnitude filter
- `max_events`: max events processed per run
- `skip_unchanged`: skip unchanged events based on cached marker
- `use_event_timestamp`: write event timestamp vs trigger time

## Install / deploy

### Option A: Generic InfluxDB 3 trigger create (from a node with `influxdb3` CLI)

1) Place plugin file where CLI can read it:

- `earthquake_sampler.py`

2) Create a scheduled trigger (example reads from existing `usgs.quake`):

```bash
influxdb3 create trigger \
  --database usgs \
  --path ./earthquake_sampler.py \
  --upload \
  --trigger-spec "every:2m" \
  --trigger-arguments "source_type=influxdb_table,source_table=quake,measurement=earthquake_plugin_stats,preserve_source_schema=true,lookback_minutes=15,max_events=500,min_magnitude=0.0,skip_unchanged=true" \
  earthquake_from_table
```

### Option B: Local Kubernetes install (namespace `influxdb3`, processor pod)

This matches a processor pod named `db3-influxdb3-enterprise-processor-0`.

1) Copy plugin to processor pod:

```bash
kubectl -n influxdb3 cp ./earthquake_sampler.py db3-influxdb3-enterprise-processor-0:/tmp/earthquake_sampler.py
```

2) Get admin token from secret and create trigger in DB `usgs`:

```bash
TOKEN=$(kubectl -n influxdb3 get secret influxdb3-admin-token -o jsonpath='{.data.TOKEN}' | base64 -d)

kubectl -n influxdb3 exec db3-influxdb3-enterprise-processor-0 -- sh -lc "
influxdb3 create trigger \
  --database usgs \
  --token '$TOKEN' \
  --host http://127.0.0.1:8181 \
  --node-spec 'nodes:db3-influxdb3-enterprise-processor-0' \
  --path /tmp/earthquake_sampler.py \
  --upload \
  --trigger-spec 'every:2m' \
  --trigger-arguments 'source_type=influxdb_table,source_table=quake,measurement=earthquake_plugin_stats,preserve_source_schema=true,lookback_minutes=15,max_events=500,min_magnitude=0.0,skip_unchanged=true' \
  earthquake_from_table
"
```

## Validate plugin output

```bash
# Trigger metadata
influxdb3 query --database usgs "SELECT trigger_name, trigger_specification, disabled FROM system.processing_engine_triggers"

# Plugin logs
influxdb3 query --database usgs "SELECT time, trigger_name, log_level, log_text FROM system.processing_engine_logs WHERE trigger_name='earthquake_from_table' ORDER BY time DESC LIMIT 20"

# Schema parity with the source table
influxdb3 query --database usgs "SHOW COLUMNS FROM quake"
influxdb3 query --database usgs "SHOW COLUMNS FROM earthquake_plugin_stats"

# Copied event data
influxdb3 query --database usgs "SELECT id, mag, place, latitude, longitude, depth, time FROM earthquake_plugin_stats ORDER BY time DESC LIMIT 20"
```

## Notes

- This repo is intentionally minimal: docs + plugin code.
- No extra Python dependencies are required in plugin code (stdlib only).
