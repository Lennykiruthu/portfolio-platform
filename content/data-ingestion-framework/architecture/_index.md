---
title: "Architecture"
description: "How the framework is put together: components, data model, run lifecycle and failure handling."
---

This page describes the system as it is today. For the reasoning behind individual choices, follow the links to the [decision records]({{< ref "/data-ingestion-framework/docs" >}}).

## Components

Four containers, all defined in one `docker-compose.yml`:

```
   FRED API
      |  HTTPS + API key
      v
+-----------------------------+      +--------------------------+
| dagster-daemon              |      | dagster-webserver        |
| schedules, sensors,         |      | UI on localhost:3001     |
| run queue                   |      +--------------------------+
+--------------+--------------+
               |  app role: insert + select on raw, write on ops
               v
+----------------------------------------------------+
| Postgres 16                                        |
|   raw.records   append-only, partitioned by source |
|   ops.*         run metrics, config, status views  |
+--------------+-------------------------------------+
               |  grafana_ro: select on seven ops views only
               v
        Grafana (pipeline health dashboard)
```

- **Postgres 16** holds three things: the raw data, the operational metadata, and Dagster's own run storage.
- **dagster-daemon** is the part that makes things happen. Schedules, sensors and the run queue all live in the daemon, so nothing fires without it.
- **dagster-webserver** serves the UI. Both it and the daemon load the same code and share a storage volume so run logs are visible in the UI.
- **Grafana** is provisioned from files (datasource and dashboard are both code), with a pinned version and a read-only database role.

All ports are bound to `127.0.0.1`, and containers have memory limits.

## Data model

There are two schemas, kept apart on purpose.

### `raw`: the data as the source sent it

One logical table, `raw.records`, list-partitioned by `_source`. Every row carries the same envelope:

| Column | Meaning |
| --- | --- |
| `_source`, `_stream` | The source (`fred`) and a sub-feed inside it (a FRED series id) |
| `_run_id` | The Dagster run that landed the row |
| `_ingested_at` | When we stored it |
| `_observed_at` | The period the record describes |
| `_request_url` | The request that produced it (never contains credentials) |
| `_record_key` | Natural key, for example `DFF:2026-10-20` |
| `_payload_hash` | SHA-256 of the record's canonical JSON (sorted keys, compact separators, UTF-8), so key order and whitespace never count as a change |
| `payload` | The record exactly as the source sent it, as JSONB |

A unique index on `(_source, _record_key, _payload_hash)` is the heart of the design. Identical content for the same key is stored once. A revised value hashes differently, so it becomes a new row and the old one stays. Adding a source is one call: `select raw.register_source('<name>')`. See [D9]({{< ref "/data-ingestion-framework/docs/d9-raw-single-partitioned-table" >}}).

### `ops`: how the pipeline is doing

| Object | Purpose |
| --- | --- |
| `ops.source_config` | What each source is, and its expected run interval |
| `ops.stream_config` | Readable names and provider cadence for each stream |
| `ops.run_metrics` | One row per run and asset: status, timings, rows fetched and inserted, bytes, HTTP status, error class, attempts |
| `ops.v_*` views | Seven views the dashboard reads (see [Operations]({{< ref "/data-ingestion-framework/docs/operations" >}})) |

## Run lifecycle

For the FRED asset, a scheduled or manual run does the following:

1. **Decide the window.** Read the latest observation already stored for the series, then request from that date minus a lookback (90 days by default). A first run, or `full_refresh`, starts from 2015-01-01. The overlap is deliberate: it re-requests recent history so revised values are noticed.
2. **Fetch** the observations from FRED.
3. **Parse** into records. This step is pure (no network, no database) and computes each record's content hash.
4. **Land** the batch in one statement with `ON CONFLICT DO NOTHING`. The statement returns how many rows were genuinely new.
5. **Record metrics** for the run in `ops.run_metrics`, including per-series counts.

Steps 1 to 4 repeat for each of the six series. Because the landing is idempotent, re-running or overlapping windows never creates duplicates.

## Failure model

Every failure is an `IngestionFailure`: one exception type with a category from a small closed vocabulary and a flag saying whether retrying can help.

| What happened | Category | Retried? |
| --- | --- | --- |
| Request timed out, or FRED could not be reached | `timeout` | Yes |
| HTTP 429 | `rate_limited` | Yes |
| HTTP 5xx | `http_5xx` | Yes |
| Any other HTTP 4xx (bad series id, bad key) | `http_4xx` | No, retrying cannot fix it |
| Response was not JSON | `parse_error` | No |
| A Postgres error | `db_error` | Yes, by Dagster's retry policy |
| Anything else | `unknown` | Yes, by Dagster's retry policy |

- **Retries.** Up to three, with exponential backoff (30 second base delay) and jitter. If several series fail in one run, the run is only retried when every one of those failures was retryable.
- **Fetch failures are isolated per series.** A timeout or bad response for one series does not stop the other five. After the loop, one summary failure is raised, carrying the category of the first failure. A database error is different: it is not caught per series and ends the run at once.
- **Metrics are always written.** Success or failure, the run's metrics are recorded when it ends. If writing the metrics itself fails, that is logged and the original error still surfaces, so a metrics problem can never hide the real one. On retries, the last attempt wins.
- **Crash backstop.** If the process is killed (out of memory, container restart, run timeout), none of our code runs, so a Dagster run-failure sensor writes the row instead. It only acts on runs tagged with a `source`, and it inserts a `FAILURE` row only when the run has no row at all, so it never overwrites a run's own result. It matches on the whole run rather than the asset key, because a run launched from an asset job carries no asset selection.

## Access control

| Role | Can do |
| --- | --- |
| App role (used by the Dagster code) | `INSERT` and `SELECT` on `raw.records` (no `UPDATE` or `DELETE`, so raw is append-only), read `ops.source_config`, write `ops.run_metrics` |
| `grafana_ro` | `SELECT` on the seven `ops` views, and nothing else. No access to `raw` at all |

The views run with their owner's rights, so `v_stream_status` can summarise `raw` (row counts, latest dates) without the dashboard role ever being able to read raw payloads. Both passwords are re-synced from `.env` on every schema run.

## Keeping the API key out of everything

FRED requires the key in the query string, which makes it easy to leak. Three things prevent that:

- The URL stored in `raw.records._request_url` is built without the key.
- `httpx` request logging is lowered to warnings, because at its default level it would print every URL.
- Request errors are re-raised with `from None`, because the original exception can embed the full URL, key included.

## Known limits

- **One source.** FRED is the only source so far. The framework is shaped for more, but that claim is untested until a second one lands.
- **Staleness measures runs, not data.** A source is `STALE` when its last successful run is older than twice its expected interval. That says the pipeline is running, not that the provider published something new. Per-stream data recency is available in `v_stream_status`.
- **`parse_error` is narrow.** Only a non-JSON response is classified that way. A JSON body with unexpected content would currently surface as `unknown`.
- **Schema-drift detection is not built.** `ops.run_metrics.schema_hash` is reserved for it and currently unused.
- **JSON only.** Original response bytes are not stored. Object storage is deferred until a source returns something that isn't JSON (see D9).
- **No point-in-time view yet.** Revisions are preserved as separate rows, so "what did this series say on date X" is answerable from the data, but there is no ready-made view for it.
- **No time-based retention.** Partitions are by source, so retention is per source, and only by dropping a partition.
