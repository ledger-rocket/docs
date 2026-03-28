# Handle Currencies

## Overview

Fragment enables multi-currency ledger management through a flexible system supporting different currency modes, exposure tracking, and custom currency definitions.

## Track Exposure

Multi-currency ledgers often reflect temporary states where companies accept payment in one currency but intend converting it to another. Between accepting and converting the money, the exchange rate could change. To track potential gains or losses from rate fluctuations, use a Change Ledger Account with multiple currency balances.

### Example Scenario

A platform records when User 1 pays User 2 across different wallet currencies. The exposure account captures the exchange rate difference, allowing the system to track the difference in exchange rates when converting funds at potentially different rates.

## Ledger Accounts Configuration

### Multi-Currency Setup

Enable multi-currency by setting `defaultCurrencyMode` to `multi` without specifying a default currency:

```json
{
  "defaultCurrencyMode": "multi",
  "accounts": []
}
```

### Single-Currency Accounts

Within a multi-currency ledger, individual accounts can be configured for single currencies by specifying `currencyMode` and `currency` properties directly on the account.

Property inheritance applies -- child accounts inherit parent settings unless overridden.

## Ledger Entries

Multi-currency Ledger Entries require specifying the currency for each line. The Accounting Equation must balance per currency, necessitating minimum four Ledger Lines.

### Parameterized Entries

Currency codes can be parameterized for reusability: `"code": "{{currency}}"` allows a single entry type to handle multiple currencies dynamically.

### Account Templates

Ledger Account templates with parameterized currencies enable creating linked accounts across different currencies: `"path": "bank-accounts:{{currency}}"` supports flexible account instantiation.

### Entry Conditions

Multi-currency conditions must specify applicable currency: each condition includes a currency parameter indicating which balance it evaluates.

## Reading Balances

### Latest Balances

Query specific currency: use the `currency` argument in singular queries like `ownBalance`. Retrieve all currencies using plural forms: `ownBalances` returns a list of currency-amount pairs.

### Aggregated Balances

Aggregated balance queries support filtering by currency or querying all currencies simultaneously using `childBalances` and `balances`.

### Consistency Modes

The `consistencyMode` argument accepts three values: `strong`, `eventual`, or `use_account` (respecting schema defaults). Only `ownBalance` and `ownBalances` support strong consistency.

### Historical and Period-Based Queries

- **Point-in-time**: Use the `at` argument with temporal formats
- **Net change**: Query `ownBalanceChanges`, `childBalanceChanges`, or `balanceChanges` with a required `period` parameter
- **Balance history**: `balancesDuring` and `balanceChangesDuring` track values over time with configurable granularity

## Custom Currencies

Track non-standard values like reward points, stocks, or physical inventory through custom currency definitions.

### Creating Custom Currencies

Call the `createCustomCurrency` mutation with parameters including `customCurrencyId`, `precision`, `name`, and `customCode`. The custom currency identifier enables usage across accounts and ledger lines.

### Implementation

Reference custom currencies by setting `customCurrencyId` on currency fields throughout your ledger structure, allowing any value type to be tracked within the accounting framework.
