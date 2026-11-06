---
title: "One failure type, and keeping the API key out of the logs"
date: 2026-10-22
description: "Classifying every failure, deciding what is worth retrying, and the three places an API key can leak."
tags: ["observability", "dagster"]
---

> Written after the fact from the code.

## Goal

When something fails, the system should say what kind of failure it was and whether trying again could help. And the FRED API key should never end up in a log, a stack trace or the database.

## What I did

**One exception type.** Everything that can go wrong during ingestion raises `IngestionFailure`, which carries a category from a small closed list (`timeout`, `rate_limited`, `http_5xx`, `http_4xx`, `parse_error`, `db_error`, `run_crashed`, `unknown`) and a flag saying whether retrying is worthwhile. The category is stored with the run, and a database view turns it into a plain sentence for the dashboard.

**Retry only what retrying can fix.** Timeouts, rate limits and server errors are retried with exponential backoff and jitter. A 4xx is not, because a bad series id or a bad key will fail the same way every time. Neither is a response that isn't JSON.

**The run's metrics are written in a `finally` block**, success or failure, and if writing the metrics itself fails it is logged without replacing the real error.

## What surprised me

FRED wants its API key in the query string, and that is a trap in three places:

1. `httpx` logs every request URL at INFO level, so I turned its logging down to warnings.
2. A request error can embed the full URL in its message. I re-raise with `from None` so the original exception, and the key inside it, doesn't travel with the new one.
3. I store the request URL alongside every row for traceability, so it has to be built separately, without the key.

## Decision

Failure handling is deliberately boring and uniform: one type, one vocabulary, one table. Anything unclassified falls through to `unknown` and still gets recorded.

## Next

A second source. Parts of this, such as the exact meaning of `parse_error`, are only proven against one provider so far.
