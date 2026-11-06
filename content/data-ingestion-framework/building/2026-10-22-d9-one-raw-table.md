---
title: "D9: why one raw table, partitioned by source"
date: 2026-10-22
description: "I wanted time partitions. Postgres would have quietly broken my deduplication."
tags: ["decisions", "postgres", "data-modeling"]
---

> Written after the fact from the code. The full decision record is [D9]({{< ref "/data-ingestion-framework/docs/d9-raw-single-partitioned-table" >}}).

## Goal

A raw layer that handles every future source with one insert path, and that can never store the same record twice.

## What I did

I settled on a single table, `raw.records`, list-partitioned by source, with a common envelope (source, stream, run id, ingest time, observed time, request URL, record key, content hash, payload). Adding a source is one function call that creates its partition.

## What surprised me

My first instinct was to partition by time, which is the usual advice for append-heavy tables. It doesn't work here. Postgres requires every unique index on a partitioned table to include the partition key. My deduplication key must *not* include the ingest time, because the same record seen next month has to conflict with the one I already stored. Partitioning by `_ingested_at` would have forced the time into the key, and idempotency would have broken without any error. The failure would have been silent duplicates.

Partitioning by `_source` is safe because source is already part of the key.

A smaller surprise: PostgreSQL 16 does not support identity columns on partitioned tables, so ids come from a plain sequence.

## Decision

List-partition by source. Revisit when one source passes roughly 20 GB or needs time-based retention. At that point the plan is a small unpartitioned index table for deduplication, with time sub-partitions on top.

## Next

How to define "the same record" when the provider stamps every row with today's date.
