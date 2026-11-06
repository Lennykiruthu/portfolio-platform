---
title: "Documentation"
description: "Decisions, source references and operating notes for the ingestion framework."
---

Reference material for the framework. For the big picture, start with [Architecture]({{< ref "/data-ingestion-framework/architecture" >}}).

- **[D9: one raw table, partitioned by source]({{< ref "/data-ingestion-framework/docs/d9-raw-single-partitioned-table" >}})** is the storage decision and the constraint behind it.
- **[Source: FRED]({{< ref "/data-ingestion-framework/docs/source-fred" >}})** covers the series, schedule, windowing and the hashing gotcha.
- **[Operations]({{< ref "/data-ingestion-framework/docs/operations" >}})** explains how to read pipeline health and what to do when a run fails.
