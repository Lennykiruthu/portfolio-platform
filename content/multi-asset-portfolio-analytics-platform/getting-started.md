---
title: "Getting Started"
description: "How to create an account and start using the portfolio analytics platform."
weight: 2
---

# Getting Started

The platform is self-hosted and currently in early access. Here's how to get up and running.

---

## Creating an Account

Navigate to the platform URL and you'll land on the login screen. Click **Create Account** and fill in your email and password. There's no email verification step right now — your account is created immediately.

Once logged in, you'll see the **Portfolio Transaction Ledger** — the main interface for logging trades.

---

## Logging Your First Transaction

The sidebar on the left has everything you need:

1. **Select a ticker** — Use the dropdown to pick a ticker symbol. If it's a ticker the system hasn't seen before, it will validate it against Yahoo Finance and backfill the full price history before saving. This can take a few seconds.

2. **Choose transaction type** — BUY or SELL.

3. **Set quantity** — Number of shares or units. Supports fractional shares.

4. **Set price per unit** — The price you paid (or received) in USD.

5. **Set the transaction date** — Defaults to today. You can backdate transactions to reflect your actual purchase history.

The **Total BUY value** at the bottom updates in real time as you adjust quantity and price.

Hit **Submit** and your transaction is written to the ledger. The `transaction_sensor` in Dagster will pick it up and update your portfolio models on the next sensor tick.

---

## Viewing Your Portfolio

The **Portfolio Analytics** tab (next to Transactions) will show your portfolio dashboard once you have transactions and the models have run. *(This tab is currently being built — check back soon.)*

---

## Notes

- Prices are in USD. Multi-currency support is not yet implemented.
- The platform pulls daily prices on weekdays at 6PM UTC. Your portfolio metrics update after each run.
- Macro context (FRED data) refreshes on the 1st of each month.
- If you log a transaction and the analytics don't reflect it immediately, the pipeline may not have run yet. Give it until after 6PM UTC.
