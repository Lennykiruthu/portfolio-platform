---
title: "Scaffolding: Dagster, Postgres and Grafana in one compose file"
date: 2026-10-15
description: "Setting up the project skeleton and a four-container stack that fails loudly when misconfigured."
tags: ["dagster", "docker"]
---

> Written after the fact from the code.

## Goal

One command to bring up everything the framework needs: a database, the orchestrator and a dashboard, with sensible security defaults from day one rather than bolted on later.

## What I did

I scaffolded a Dagster project with `uv` and Python 3.13, then wrote a `docker-compose.yml` with four services: Postgres 16, the Dagster webserver, the Dagster daemon and Grafana.

A few choices in that file I would make again:

- **Fail fast on missing config.** Required variables use `${VAR:?set in .env}`, so Compose refuses to start with a clear message instead of booting a half-configured stack.
- **Bind to localhost.** Every port is published on `127.0.0.1` only. The webserver is on 3001 so it doesn't clash with `dg dev` on 3000.
- **Pin Grafana.** The version is a required variable, so an upgrade is always a decision.
- **Memory limits** on the Dagster and Grafana containers, because this is meant to run on a small server.

## What surprised me

The daemon is not optional. The webserver can show you everything, but schedules, sensors and the run queue all live in the daemon, so without it nothing ever fires. I made it a first-class service with a comment saying so, because it is exactly the thing I would forget in six months.

## Decision

Both the webserver and the daemon run my code, so they share one environment block and one storage volume. That way a run launched by the daemon writes logs the webserver can read.

## Next

The schema: what the raw data looks like once it lands.
