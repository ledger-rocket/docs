# Read Balances

Fragment supports querying Ledger Account balances in multiple ways, including latest balances, historical data, and balance changes over specific periods.

## Latest Balances

Use the `ledgerAccount` query to retrieve a Ledger Account's current balance:

```graphql
query GetBalance($ledgerAccount: LedgerAccountMatchInput!) {
  ledgerAccount(ledgerAccount: $ledgerAccount) {
    balance
  }
}
```

## Aggregated Balances

Ledger Accounts maintain three distinct balance types:

- **ownBalance**: Sum of Ledger Lines posted directly to the account, excluding child accounts
- **childBalance**: Sum of Ledger Lines posted to child accounts
- **balance**: Combined total of own balance plus child balance

This hierarchical structure allows tracking balances at different account levels within your ledger structure.

## Consistent Balance Reads

Balance reads are eventually consistent by default. To achieve strong consistency, configure the Ledger Account's `consistencyConfig` in your Schema with `ownBalanceUpdates: "strong"`.

When querying, set the `consistencyMode` parameter on balance queries to:

- `strong` for strongly consistent reads
- `eventual` for eventually consistent reads
- `use_account` to apply the Schema's configured setting

Only `ownBalance` supports strong consistency mode.

## Point-in-Time Balances

Query historical balances using the `at` argument with various time granularities:

```graphql
query GetHistoricalBalances($ledgerAccount: LedgerAccountMatchInput!) {
  ledgerAccount(ledgerAccount: $ledgerAccount) {
    end_of_year: balance(at: "1969")
    end_of_month: balance(at: "1969-07")
    end_of_day: balance(at: "1969-07-21")
    end_of_hour: balance(at: "1969-07-21T02")
  }
}
```

The `at` parameter works at hour-level granularity.

## Net Change Queries

Track balance changes over reporting periods using balance change fields:

- **ownBalanceChange**: Change in own balance
- **childBalanceChange**: Change in child balance
- **balanceChange**: Total balance change

Specify a `period` parameter supporting years, quarters, months, days, or hours:

```graphql
query GetBalanceChanges($ledgerAccount: LedgerAccountMatchInput!) {
  ledgerAccount(ledgerAccount: $ledgerAccount) {
    currentYear: balanceChange(period: "1969")
    lastYear: balanceChange(period: "1968")
    lastYearQ4: balanceChange(period: "1968-Q4")
  }
}
```

## Balance History Over Periods

Query balances or balance changes across time ranges using `balancesDuring` and `balanceChangesDuring` queries, which require:

- **startTime**: UTC date string marking period start
- **duration**: Length of period (positive or negative for lookback)
- **granularity**: Time unit (hourly, daily, or monthly)

Granularity must be finer than the start time precision. Duration limits vary: monthly (240 periods/20 years), daily (731 periods/2 years), hourly (744 periods/31 days).

Responses include `startTime`, `endTime`, `granularity`, and `nodes` containing historical data points.

## Multi-Currency Balances

For accounts with `currencyMode: multi`, either:

1. Use the `currency` argument to query specific currency balances
2. Use plural field names (like `balances`) to retrieve all currency balances

```graphql
query GetMultiCurrencyBalances($ledgerAccount: LedgerAccountMatchInput!) {
  ledgerAccount(ledgerAccount: $ledgerAccount) {
    latestUSDBalance: balance(currency: { code: USD })
    balances {
      nodes {
        currency { code }
        amount
      }
    }
  }
}
```

## Timezone Offset Configuration

The Ledger's `balanceUTCOffset` affects how periods and times are interpreted. Specified during Ledger creation as a UTC offset string (plus/minus HH:00 format), supported ranges span from -11:00 to +12:00.

When querying balances:

- Time strings are interpreted in the Ledger's configured timezone
- Period queries return net changes between local midnight boundaries
- Daylight savings is ignored for consistent daily calculations

Default offset is "+00:00" (UTC).
