---
title: "The FRED hashing gotcha"
date: 2026-10-22
description: "Every FRED observation changes every day, unless you hash the right things."
tags: ["fred", "data-modeling"]
---

> Written after the fact from the code.

## Goal

Re-running the FRED asset, or overlapping its date windows, should insert nothing new unless a value actually changed.

## What I did

I hash each observation's content with SHA-256, over canonical JSON (sorted keys, compact separators) so key order and whitespace can never look like a change, and let a unique index on `(source, record key, hash)` reject repeats. The landing statement uses `ON CONFLICT DO NOTHING` and returns how many rows were genuinely new, so a run can report "fetched 90, new 0".

## What surprised me

FRED stamps every observation with `realtime_start` and `realtime_end` set to the day you asked. Hash the whole observation and every row looks different every day, which defeats the point entirely.

The fix is in the parser: the hash covers only series, date and value. The complete observation, realtime fields included, is still stored verbatim in `payload`, so nothing is lost. As a side effect, those fields now record the day I first saw each value.

## Decision

The parser is a set of pure functions with no network or database access, so this behaviour is easy to test on its own. Each run also re-requests a trailing 90-day window, because recent values get revised. A revised value hashes differently and lands as a new row next to the old one, which keeps the revision history instead of overwriting it.

## Next

Knowing when something goes wrong: roles, status views and failure handling.
