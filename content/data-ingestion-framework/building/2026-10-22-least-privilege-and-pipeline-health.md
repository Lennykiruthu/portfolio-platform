---
title: "Least privilege and a pipeline health layer"
date: 2026-10-22
description: "An append-only app role, a read-only dashboard role and a failure sensor that backs up crashed runs."
tags: ["observability", "postgres", "dagster"]
---

> Written after the fact from the code.

## Goal

Make the pipeline's state visible, and make sure neither the ingestion code nor the dashboard can do more than it needs to.

## What I did

**Roles.** The ingestion code connects as an app role that can insert and select on `raw.records`, and write to the run metrics table. It has no update or delete on raw, so raw is append-only. Grafana uses a separate read-only role that can select from a short list of `ops` views and cannot touch `raw` at all. Both role passwords are re-synced from `.env` each time the schema is applied, so `.env` stays the single source of truth.

**Views.** The dashboard reads views for source status, success streaks, a 7-day success rate, a 90-day daily grid, recent failures and per-stream row counts. Views run with their owner's rights, so the per-stream summary can count rows in `raw` without the dashboard role ever reading a payload.

**Failures.** Each run records its own metrics, including a coarse error class that a view turns into a plain sentence, such as "The data provider asked us to slow down".

## What surprised me

A run that crashes outright never gets to write its own metrics. So a Dagster failure sensor inserts a `FAILURE` row as a backstop, but only when the run has no row at all. I first thought of matching on the asset key, but a run launched from an asset job carries no asset selection, so the sensor can't know which key the asset used. Matching on the whole run avoids that.

## Decision

The dashboard role stays read-only on views, which means Grafana can be exposed to the public later without exposing any raw data. Provisioning is done from files, so the dashboard and datasource are code in the repo.

## Next

A second source, chosen to force a new design problem, and CI so the tests run on every push.
