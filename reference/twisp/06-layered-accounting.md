# Layered Accounting

> Source: https://www.twisp.com/docs/accounting-core/layered-accounting

## Overview

Twisp employs a three-layer accounting model to categorize transactions and provide clarity on account fund states.

## The Three Layer Model

The system distinguishes transactions across three important categories:

1. **Settled Layer** - "transactions that have fully settled"
2. **Pending Layer** - "holds and pending transactions which have been authorized but not yet settled"
3. **Encumbrance Layer** - "expected, planned, and scheduled future transactions"

### Benefits

This segmentation provides "data integrity needed for more accurate and useful balance calculations and funds flow modelling." The pending layer enables verification that accounts will have sufficient funds after authorized transactions clear. The encumbrance layer supports future transaction scheduling and budgeting features.

Without explicit layers, "chaos reigns." Many DIY ledgers lack layering concepts or apply them partially, making it difficult to assess ledger status or perform basic operations like reconciliation.

**Note:** Not all products require all three layers, but they remain available as product needs evolve.

## Layered Balances

"Each account can have a different aggregate balance in each layer depending on how transactions have been posted."

Balance calculation per layer:

```text
b(l) = sum of all entries on layer l
```

Where b(l) represents the layer balance.

Accounts maintain debit balances, credit balances, and normal balances across layers.

## Layers in Practice

### Initial State

A settled layer entry of $100.00 (debit) produces:

- Settled balance: $100.00
- Pending balance: $0.00
- Encumbrance balance: $0.00

### With Pending Transactions

When pending layer entries are added (modeling hold-settle patterns):

- Pending entries affect only pending balances
- Settled entries affect only settled balances
- Each layer aggregates independently

### Multiple Layer Entries

"Notice that the layered balances are only aggregating entries from their layer."

Example: With settled, pending, and encumbrance entries:

- Settled: $118.45 DR, $44.82 CR
- Pending: $20.50 DR, $20.50 CR
- Encumbrance: $44.82 DR, $40.00 CR

## Available Balance Calculation

The available balance rolls up individual layer balances into a single figure. It can be configured to sum:

- All layers
- Settled and pending layers only
- Settled layer only

The formula:

```text
a(l) = {
  b(S),                  if l = S
  b(S) + b(P),           if l = P
  b(S) + b(P) + b(E),    if l = E
}
```

Where a(l) is the available balance, b(l) is the layer balance, and S, P, E represent settled, pending, and encumbrance layers respectively.
