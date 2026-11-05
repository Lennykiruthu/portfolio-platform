---
title: "Data Ingestion Framework"
description: "A Dagster and Postgres framework for landing public data sources reliably — raw, traceable, and observable."
image: "thumbnail.svg"
weight: 1
---

This is a data ingestion framework I'm building in public. The goal is simple to state and surprisingly hard to do well: pull data from outside sources into Postgres in a way you can trust. Raw data is never overwritten, every row can be traced back to the run that landed it, and when something breaks, the system tells you.

It started as a fix for my own projects. My [Multi Asset Portfolio Analytics](/multi-asset-portfolio-analytics-platform/) platform pulls FRED and Yahoo Finance data with ad-hoc scripts, and I wanted one proper ingestion layer underneath it instead. It's early, it currently has one source, and this site documents every decision along the way, including the ones I get wrong.

---
## What It Does So Far

- **Raw landing layer** — Source data lands in a single partitioned raw table, untouched. The reasoning is written up as a decision record ([D9]).
- **Content hashing** — Each payload is hashed so the framework can tell new data from data it has already seen. [Confirm exact behaviour: skip duplicates / flag changes?]
- **Operational metadata** — An `ops` schema records runs and metrics, kept separate from the raw data itself.
- **Failure handling** — A Dagster failure sensor and a shared error model so failed runs are surfaced rather than silently lost.
- **Pipeline health dashboard** — A provisioned Grafana dashboard over the ops data, so the state of every source is visible at a glance.
- **Access control** — Database roles and grants separate what ingestion can write from what consumers can read.
- **First source: FRED** — Federal Reserve economic data, with its own source doc and parsing module. Every new source follows the same pattern.
- **Tests** — Unit tests for hashing and parsing, and integration tests for raw landing, the FRED asset and the failure sensor.

---
## Where It's Going

Each new source is chosen to force a new design problem, not to fill a catalogue:

- A paginated, rate-limited API (retries, backoff, resumable backfills)
- An HTML or Excel source (schema drift, quarantining bad rows)
- A large historical backfill (partitioning and runtime limits)
- A source that changes under me, simulated deliberately

Also on the list: CI with GitHub Actions, and a one-command quickstart.

---
## Sections

- **Build Log** — Dated notes on what I built, what broke, and what I decided.
- **Docs** — The current architecture, decisions and per-source documentation.