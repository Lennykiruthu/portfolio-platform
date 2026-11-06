---
title: "Source: FRED"
description: "The first source: which series, how often, how the window works and the hashing gotcha."
tags: ["fred", "dagster"]
---

Macro indicators from the Federal Reserve Economic Data service (FRED), published by the St. Louis Fed. This is the framework's first source and the template every later one follows.

<!-- CONFIRM: reconcile with docs/sources/fred.md in the repo; that file was not available when this page was written -->

## Series

| Series id | Name | Provider cadence |
| --- | --- | --- |
| `DFF` | Federal funds rate | Daily |
| `GS10` | 10-year Treasury yield | Monthly |
| `CPIAUCSL` | Consumer price index (CPI) | Monthly |
| `UNRATE` | Unemployment rate | Monthly |
| `USREC` | US recession indicator | Monthly |
| `T10YIE` | 10-year breakeven inflation | Daily |

The series id is the stream. The friendly name exists only for humans.

## Asset and schedule

- **Asset:** `raw/fred`, grouped under `raw`. It appends one row per observation to `raw.records`.
- **Job:** `fred_daily`, tagged with `source=fred` (the failure sensor reads that tag).
- **Schedule:** `0 6 * * *`, which is 06:00 UTC and 09:00 in Nairobi. That is early morning in the US, after FRED's overnight updates. It starts switched on.
- **Endpoint:** `https://api.stlouisfed.org/fred/series/observations`, requested as JSON with `series_id` and `observation_start`.
- **Timeouts:** 30 seconds overall, 10 seconds to connect. Requests send a descriptive `User-Agent`.
- **Credentials:** `FRED_API_KEY` from the environment. It is sent as a query parameter, but the stored request URL never includes it, and request logging and exception traces are set up so it doesn't leak. See [Architecture]({{< ref "/data-ingestion-framework/architecture" >}}).

## Record shape

- **Key:** `{series_id}:{date}`, for example `DFF:2026-10-20`
- **Observed at:** the observation date, at midnight UTC
- **Payload:** the observation exactly as FRED sent it

## How the window works

Each run asks for data from the latest stored date minus 90 days, not just "after the last date". Values inside that window are sometimes revised by the provider, and re-requesting them means the revision gets picked up. Unchanged rows are skipped by the dedupe index. Two options on the asset config:

- `lookback_days` (default 90) changes the trailing window.
- `full_refresh` re-requests all history from 2015-01-01, which catches old revisions such as seasonal adjustment changes.

## The hashing gotcha

FRED stamps every observation with `realtime_start` and `realtime_end` set to the day you asked. If the whole observation were hashed, every row would look changed every single day and deduplication would never work.

So the hash covers only what identifies the data: series, date and value. The full observation, realtime fields included, is still stored verbatim in `payload`. The realtime fields therefore hold the vintage date of the first time we saw that value.

The practical result: a repeated value is skipped, and a revised value becomes a new row beside the old one.

## Failure behaviour

| FRED response | Category | Retried? |
| --- | --- | --- |
| Timeout, or FRED unreachable | `timeout` | Yes |
| HTTP 429 | `rate_limited` | Yes |
| HTTP 5xx | `http_5xx` | Yes |
| Other HTTP 4xx | `http_4xx` | No |
| Body is not JSON | `parse_error` | No |

- Series are fetched independently. One failing series does not stop the other five.
- The asset retries up to three times with exponential backoff.
- After the loop, any failures are raised together as one run failure, using the category of the first failure.
- A 4xx is not retried because retrying cannot fix a bad series id or a bad API key.
