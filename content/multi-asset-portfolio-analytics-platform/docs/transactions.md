---
title: "Logging Transactions"
description: "How to record BUY and SELL transactions, what each field means, and how to correct mistakes."
weight: 1
---

# Logging Transactions

The Transaction Ledger is the primary input interface. Everything else in the platform — your portfolio performance, your holdings breakdown, the analytics — is computed from your transaction history.

---

## Fields

### Ticker
The stock or asset symbol. Must be a valid ticker recognisable by Yahoo Finance (e.g. `AAPL`, `MSFT`, `BTC-USD`, `GC=F` for gold futures).

If you enter a ticker the system hasn't seen before, it will:
1. Validate it against Yahoo Finance
2. Backfill the full available price history into the database
3. Then write your transaction

This means the first transaction for a new ticker takes a few extra seconds. Subsequent transactions for that ticker are instant.

### Transaction Type
**BUY** or **SELL**.

- BUY increases your position.
- SELL decreases it. You can't SELL more than you currently hold — the system will reject that.

### Quantity (shares / units)
The number of shares or units transacted. Fractional shares are supported (e.g. `0.5` for half a Bitcoin).

### Price per unit (USD)
The price at which you transacted, in USD. This is the price *you paid or received*, not the market price at time of submission. Enter your actual cost basis.

### Transaction Date
The date the transaction occurred. Defaults to today's date. You can backdate transactions freely — this is useful when onboarding and entering your historical trade history.

---

## The Transaction Ledger table

The main panel shows all your transactions sorted by date, most recent first. Each row shows ticker, type, quantity, price, date, and total value.

---

## Correcting Mistakes

If you entered a transaction incorrectly, use the **Delete a transaction** section at the bottom of the ledger panel. Enter the transaction ID (shown in the ledger table) and confirm deletion.

Deletion is intended for corrections only — there's no audit trail suppression, so your models will recompute correctly after a delete.

After deleting, the pipeline will recompute your holdings on the next sensor tick or scheduled run.

---

## What happens after you submit

1. The transaction is written to `bronze.transactions`.
2. The `transaction_sensor` in Dagster detects the new record.
3. If the ticker is new, price history is backfilled.
4. The dbt models downstream of `stg_transactions` are rerun: `int_daily_holdings`, `fct_daily_returns`, and all gold marts.
5. Your analytics dashboard reflects the updated data.

Depending on when you submit, this can take anywhere from a few seconds (sensor tick) to a few minutes if there's a backfill happening.
