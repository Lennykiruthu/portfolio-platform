---
title: "The Platform"
description: "A self-hosted portfolio analytics platform — what it is, how it works, and where it's going."
weight: 1
---

# The Platform

This is a self-hosted portfolio analytics platform I built from scratch. It lets me (and anyone I give access to) track investment transactions, compute daily portfolio performance, and layer in macroeconomic context from the Federal Reserve.

It's not a SaaS product yet. It's a real system running on real infrastructure that I built to scratch my own itch — and to prove to myself I could ship something end-to-end.

## What it does

- **Transaction ledger** — Log BUY and SELL transactions. New tickers are validated against Yahoo Finance and their full price history is backfilled automatically before the transaction is written.
- **Daily portfolio performance** — Computed nightly for every user, based on their actual holdings at each point in time.
- **Macroeconomic context** — Federal Reserve economic data (FRED) is ingested monthly and used to classify macro regimes, which are layered onto portfolio performance charts.
- **Per-asset analytics** — Time series performance, KPIs, and holding summaries broken down by ticker.

## Sections

- [Architecture](/platform/architecture/) — The full stack, the DAG, and why I made the decisions I did.
- [Getting Started](/platform/getting-started/) — How to create an account and start logging transactions.
