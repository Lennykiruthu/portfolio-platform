<!-- ---
title: "Architecture"
description: "The full technical stack — data sources, transformation layer, orchestration, storage, and frontend."
weight: 1
--- -->

# Architecture

This page documents the current architecture of the platform. I update it as the system evolves.

**Last updated:** April 2026

---

## Stack Overview

| Layer | Technology | Notes |
|---|---|---|
| Data Sources | yFinance, FRED API | Equity prices + macro indicators |
| Storage | PostgreSQL | Single database, multiple schemas |
| Transformation | dbt | Staging → intermediate → mart pattern |
| Orchestration | Dagster | Scheduled jobs + event sensors |
| Auth / API | FastAPI | JWT-based auth, user management |
| Frontend | Streamlit | Transaction input + personal dashboards |
| Deployment | Docker | Compose-based, self-hosted |

---

## Data Sources

### yFinance
Yahoo Finance is the primary source for daily equity prices. When a user logs a transaction for a ticker the system hasn't seen before, it backfills the full available price history for that ticker into `bronze.raw_prices` before writing the transaction. This ensures the daily holdings computation has complete data.

### FRED (Federal Reserve Economic Data)
Macroeconomic indicators are pulled monthly from the FRED API into `bronze.raw_fred`. These are used downstream to classify macro regimes (e.g. expansionary vs. contractionary) which are layered onto portfolio performance charts as context.

---

## dbt Transformation Layer

The dbt project follows a standard three-layer medallion pattern inside a `default` schema group:

```
bronze (raw)  →  staging  →  intermediate  →  mart (gold)
```

### Staging models
- **`stg_raw_prices`** — Cleans and types the raw yFinance price data.
- **`stg_fred_macro`** — Cleans and types the raw FRED indicator data.
- **`stg_transactions`** — Cleans user-submitted transaction records.

### Intermediate models
- **`int_daily_holdings`** — Reconstructs each user's holdings on every calendar day based on their transaction history. This is the core computational model.
- **`fct_daily_returns`** — Joins daily holdings against prices to produce daily return figures per user, per asset.
- **`fct_macro_regimes`** — Classifies each date into a macro regime using the FRED indicators.

### Mart models (gold layer)
- **`gold_macro_context`** — Macro regime data formatted for dashboard consumption.
- **`gold_p_timeseries`** — Portfolio-level daily time series (value, returns, drawdown) per user.
- **`gold__performance`** — Aggregate performance metrics per user.
- **`gold_ticker_kpis`** — Per-ticker KPIs: total return, holding period, cost basis vs. current value.
- **`dim_assets`** — Asset dimension table: ticker metadata, sector, asset class.

The DAG flows from `raw_prices` and `raw_fred` (source nodes) through the staging and intermediate layers into the gold marts.
![alt](/images/dbt_lineage.png)

---

## Orchestration (Dagster)

Three jobs/sensors manage pipeline execution:

### `daily_market_data_ingestion`
- **Schedule:** 06:00 PM UTC, Monday–Friday
- Fetches updated prices from yFinance for all tickers in the system, then triggers a full dbt run to refresh all downstream models.

### `monthly_fred_macro_refresh`
- **Schedule:** 06:00 AM UTC, 1st of each month
- Pulls updated FRED indicators and refreshes the macro context models.

### `transaction_sensor`
- **Type:** Standard sensor (event-driven)
- Monitors for new transactions written to `bronze.transactions`. When triggered, it backfills price history for any new tickers and reruns the affected downstream models.

---

## Storage (PostgreSQL)

A single Postgres instance with a schema-per-layer approach:

- `bronze` — Raw ingested data (managed by Dagster/ingestion scripts)
- `default` — dbt-managed transformed models
- `public` — Application tables managed by FastAPI (users, sessions)

---

## Auth & API (FastAPI)

A FastAPI service handles:
- User registration and login
- JWT token issuance and validation
- Transaction write endpoints (with ticker validation before write)
- Serving portfolio data to the Streamlit frontend

---

## Frontend (Streamlit)

The Streamlit app is the user-facing interface. It has two main sections:

1. **Transaction Ledger** — Log BUY/SELL transactions with ticker, quantity, price, and date. New tickers are validated and backfilled automatically.
2. **Portfolio Analytics** — Personal dashboard showing portfolio performance, macro context overlays, and per-ticker KPIs. *(In active development as of April 2026.)*

Authentication is handled via the FastAPI service — Streamlit calls the API on login and stores the JWT in session state.

---

## Deployment (Docker)

The full stack runs as a Docker Compose application:

- `db` — PostgreSQL
- `api` — FastAPI service
- `dagster` — Dagster webserver + daemon
- `app` — Streamlit frontend

Everything is containerised and runs on a single host. Persistent volumes are used for the database. The plan is to move this to a cloud VM (AWS EC2 or DigitalOcean Droplet) for persistent public hosting.

---

## What's Missing / In Progress

- **Streamlit analytics dashboard** — Currently migrating from Apache Superset. The transaction ledger is done; the analytics tab is being built.
- **African market data sources** — The long-term data source expansion goal. No timeline yet.
- **Multi-user scaling** — The schema and models support multiple users already; the deployment doesn't yet handle scale gracefully.
- **CI/CD** — Currently manual deploys. Will add GitHub Actions.
