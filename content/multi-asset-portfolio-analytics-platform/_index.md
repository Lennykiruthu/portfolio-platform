<!-- ---
title: "The Platform"
description: "A self-hosted portfolio analytics platform — what it is, how it works, and where it's going."
weight: 1
--- -->

# Multi Asset Portfolio Analytics

This is a self-hosted portfolio analytics platform I built from scratch. It lets anyone who's created an account track investment transactions, compute daily portfolio performance, and layer in macroeconomic context from the Federal Reserve.

It's not a SaaS product yet and is lacking many data sources to be a transformative tool to your investing yet, however It's a real system running on real infrastructure that I built to scratch my own itch — and to prove to myself I could ship something end-to-end.

---
## What It Does So Far

- **Transaction ledger** — Log BUY and SELL transactions. New tickers are validated against Yahoo Finance and their full price history is backfilled automatically before the transaction is written.
- **Transaction orchestration** - Whenever a user inputs a new transaction a dagster sensor picks up on it and triggers the full dbt medallion arhcitecture transformation preparing the asset for dashboarding, ad-hoc analysis and predictive modelling.
- **Orchestration** — Currently the orchestration architecture collects yfiance data for present assets from Monday to Friday and monthly for Federal Reserve data.
- **Macroeconomic context** — Federal Reserve economic data (FRED) is used to classify macro regimes, which are layered onto portfolio performance charts.
- **Dashboarding** — Time series performance, KPIs, and holding summaries broken down by ticker.

---
## Sections

- [Architecture](/multi-asset-portfolio-analytics-platform/architecture/) — The full stack, the DAG, and why I made the decisions I did.
- [Getting Started](/multi-asset-portfolio-analytics-platform/getting-started/) — How to create an account and start logging transactions.
