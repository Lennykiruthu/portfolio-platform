---
title: "Getting Started"
description: "Bring up the full stack locally and land your first FRED data."
weight: 1
---

This gets the whole stack running on your machine: Postgres, Dagster (webserver and daemon) and Grafana, then lands the first FRED data.

## Prerequisites

- Docker with Docker Compose
- A free FRED API key from the St. Louis Fed

## 1. Configure

Copy the example env file and fill in every value. Compose refuses to start if a required variable is missing.

```bash
cp .env.example .env
```

| Variable | Purpose |
| --- | --- |
| `PG_ADMIN_USER`, `PG_ADMIN_PASSWORD`, `PG_DB` | Postgres superuser and the database name |
| `APP_DB_USER`, `APP_DB_PASSWORD` | The least-privileged role the ingestion code connects as |
| `DAGSTER_PG_USER`, `DAGSTER_PG_PASSWORD`, `DAGSTER_PG_DB` | Dagster's own storage in the same Postgres |
| `GRAFANA_RO_PASSWORD` | Password for the read-only `grafana_ro` role |
| `GRAFANA_VERSION` | An exact Grafana version (it is pinned on purpose) |
| `GRAFANA_ADMIN_PASSWORD` | Grafana admin login (`GRAFANA_ADMIN_USER` defaults to `admin`) |
| `FRED_API_KEY` | Your FRED key |

`.env` holds real credentials and should never be committed. Only `.env.example` belongs in git.

## 2. Start the stack

```bash
docker compose up -d --build
```

The database creates the roles, schemas and views on its first start only. Postgres runs the files in `sql/` once, when the data volume is first created. `docker compose down -v` deletes the volume, and with it your data.

| Service | Address |
| --- | --- |
| Dagster UI | http://localhost:3001 |
| Grafana (pipeline health is the home dashboard) | http://localhost:3002 |
| Postgres | `127.0.0.1:5432` |

Everything binds to localhost. Nothing is exposed to the network by default.

## 3. Land the first data

The FRED schedule runs daily at 06:00 UTC and starts switched on, so it will fire on its own. To run it now, open the Dagster UI, find the asset `raw/fred` and click **Materialize**.

## 4. Check it worked

```bash
docker exec -it dif-postgres psql -U <PG_ADMIN_USER> -d <PG_DB> \
  -c "select _stream, count(*) from raw.records group by 1 order by 1;"
```

You should see one row per FRED series with its observation count. Then materialize again: the second run should insert zero or very few new rows, because identical content is skipped and only genuinely revised values are added. See [how deduplication works]({{< ref "/data-ingestion-framework/docs/source-fred" >}}).

Open Grafana to see the run appear in the pipeline health dashboard.

## Local development

<!-- CONFIRM: commands below assume uv + the dg CLI as suggested by uv.lock, .dg/ and the compose comment about `dg dev` -->

The compose file maps the Dagster webserver to port 3001 so it does not clash with `dg dev` on 3000, which is how you run Dagster outside Docker while developing.

```bash
uv sync
dg dev
uv run pytest
```

Unit tests cover the pure functions (hashing and parsing). Integration tests cover raw landing, the FRED asset and the failure sensor.
