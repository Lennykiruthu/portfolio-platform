---
title: "Operations"
description: "How to read pipeline health, what the statuses mean and what to do when a run fails."
tags: ["observability"]
---

Everything the dashboard shows comes from views in the `ops` schema. The Grafana role can read those views and nothing else.

## The views

| View | Answers |
| --- | --- |
| `v_source_status` | Is each source healthy right now? |
| `v_source_streak` | How many successful runs since the last failure? |
| `v_success_rate_7d` | What share of runs succeeded in the last 7 days? |
| `v_daily_status` | One row per source per day for 90 days |
| `v_recent_failures` | The latest 50 failures, with a plain-language reason |
| `v_stream_status` | Per stream: rows held, latest observation, last ingest |
| `v_daily_new_rows` | New rows landed per day |

## Source status

| Status | Meaning |
| --- | --- |
| `HEALTHY` | The last run succeeded and the last success is recent enough |
| `FAILING` | The most recent run failed |
| `STALE` | The last success is older than twice the source's expected interval |
| `NO DATA` | The source has never run |

Staleness is about the pipeline, not the provider. A monthly series that hasn't changed will not make a source stale, because the check uses the time of the last successful run. To see how recent the data itself is, use `v_stream_status`.

## Daily status codes

`v_daily_status` uses three codes: `1` is a day with runs and no failures, `0` is a day with at least one failed run, and `2` is a day with no run at all.

## Failure reasons

Each failure carries a coarse error class, shown to the reader in plain language:

| Error class | Shown as | Retried automatically? |
| --- | --- | --- |
| `timeout` | The data provider did not respond in time | Yes |
| `rate_limited` | The data provider asked us to slow down | Yes |
| `http_5xx` | The data provider had a server problem | Yes |
| `http_4xx` | The data provider rejected our request | No |
| `parse_error` | The data arrived in an unexpected format | No |
| `db_error` | We could not save the data to our database | Yes |
| `run_crashed` | The pipeline run stopped unexpectedly | Not applicable |
| anything else | Unexpected error | Yes |

## When a run fails

1. **Check whether it retried.** Retryable failures are attempted up to three times with backoff, and the `attempts` column shows how many were used. A failure that retrying cannot fix (`http_4xx`, `parse_error`) shows one attempt.
2. **Look at the class.** `timeout`, `rate_limited` and `http_5xx` are usually the provider and often clear on their own. `http_4xx` points at our request, so check the API key and the series id first. `parse_error` means FRED returned something that wasn't JSON. `db_error` points at Postgres.
3. **Remember partial success.** If one series failed to fetch, the others still landed. `ops.run_metrics.details` holds per-stream counts. A database error is the exception: it ends the run immediately.
4. **Re-run safely.** Landing is idempotent, so running the asset again cannot create duplicates.
5. **`run_crashed`** means the process was killed (out of memory, a container restart or a run timeout) before it could write its own result, and the failure sensor recorded it. Check the run's logs in the Dagster UI.

## What a run row records

- `latency_ms` is the time spent waiting on the provider, summed across series. If no request completed, it falls back to the elapsed time of the run.
- `rows_fetched` and `rows_inserted` are totals for the run, so `rows_inserted` should be near zero on a quiet day.
- `attempts` counts the final attempt number. A retried run keeps one row, and the last attempt's result wins.

## Public dashboard option

Grafana can allow anonymous viewing (`GRAFANA_ANONYMOUS=true`, as a viewer only). Because the dashboard role can only read the `ops` views, a public dashboard exposes pipeline health and never the raw data.
