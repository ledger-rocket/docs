# Generate Reports

Fragment supports queries for generating common financial reports.

## Balance Sheet

A balance sheet reports the net worth of a business at the end of a reporting period.

```graphql
query GetBalanceSheet(
  $ledgerIk: SafeString!
  $balanceAtEndOf: LastMoment!
  $accountsFilter: LedgerAccountsFilterSet!
  $after: String
) {
  ledger(ledger: { ik: $ledgerIk }) {
    ledgerAccounts(filter: $accountsFilter, after: $after) {
      nodes {
        id
        name
        type
        balance(at: $balanceAtEndOf)
      }
      pageInfo {
        hasNextPage
        endCursor
      }
    }
  }
}
```

Parameters:

```json
{
  "ledgerIk": "ik-used-to-create-ledger",
  "accountsFilter": {
    "type": {
      "in": ["asset", "liability"]
    }
  },
  "balanceAtEndOf": "1969"
}
```

Generate a balance sheet by querying `balance` on all asset and liability accounts. A `LastMoment` parameter on the `at` field returns the balance at period end. The `in` operator performs OR logic, retrieving both account types.

## Income Statement

An income statement reports how a business's net worth changed over a reporting period.

```graphql
query GetIncomeStatement(
  $ledgerIk: SafeString!
  $balanceChangeDuring: Period!
  $accountsFilter: LedgerAccountsFilterSet!
  $after: String
) {
  ledger(ledger: { ik: $ledgerIk }) {
    ledgerAccounts(filter: $accountsFilter, after: $after) {
      nodes {
        path
        name
        type
        balanceChange(period: $balanceChangeDuring)
      }
      pageInfo {
        hasNextPage
        endCursor
      }
    }
  }
}
```

Parameters:

```json
{
  "ledgerIk": "ik-used-to-create-ledger",
  "accountsFilter": {
    "type": {
      "in": ["income", "expense"]
    }
  },
  "balanceChangeDuring": "1969"
}
```

Generate an income statement by querying `balanceChange` on income and expense accounts. A `Period` parameter retrieves the difference in account balance between the start and end of that period.

## Account Statement

An account statement reports how a specific account changed over a reporting period, including starting balance, ending balance, and all posted transactions.

```graphql
query GetAccountStatement(
  $accountMatch: LedgerAccountMatchInput!
  $startingBalanceAtEndOf: LastMoment!
  $endingBalanceAtEndOf: LastMoment!
  $linesFilter: LedgerLinesFilterSet!
  $after: String
) {
  ledgerAccount(ledgerAccount: $accountMatch) {
    path
    name
    type
    startingBalance: balance(at: $startingBalanceAtEndOf)
    endingBalance: balance(at: $endingBalanceAtEndOf)
    lines(filter: $linesFilter, after: $after) {
      nodes {
        id
        key
        posted
        description
        amount
        ledgerEntryId
      }
      pageInfo {
        hasNextPage
        endCursor
      }
    }
  }
}
```

Parameters:

```json
{
  "accountMatch": {
    "ledger": {
      "ik": "ik-used-to-create-ledger"
    },
    "path": "liabilities/customer-deposits/customer:123"
  },
  "linesFilter": {
    "posted": {
      "after": "1969-07-01T00:00:00.000Z",
      "before": "1969-07-30T23:59:59.999Z"
    }
  },
  "startingBalanceAtEndOf": "1969-06",
  "endingBalanceAtEndOf": "1969-07"
}
```

Generate an account statement by querying `balance` and `lines` on a specific account. Use GraphQL aliases to make multiple balance queries in one request. The `after` and `before` filters on lines are inclusive.

## Journal Export

A journal export lists all ledger entries posted during a reporting period.

```graphql
query GetJournalExport(
  $ledgerIk: SafeString!
  $entriesFilter: LedgerEntriesFilterSet!
  $after: String
) {
  ledger(ledger: { ik: $ledgerIk }) {
    ledgerEntries(filter: $entriesFilter, after: $after) {
      nodes {
        id
        type
        posted
        description
        lines {
          nodes {
            id
            description
            account {
              name
              path
              type
            }
            amount
          }
        }
      }
      pageInfo {
        hasNextPage
        endCursor
      }
    }
  }
}
```

Parameters:

```json
{
  "ledgerIk": "ik-used-to-create-ledger",
  "entriesFilter": {
    "posted": {
      "after": "1969-01-01T00:00:00.000Z",
      "before": "1969-03-31T23:59:59.999Z"
    }
  }
}
```

Generate a journal export by listing ledger entries and their associated lines with account details. Use pagination cursors to page through lengthy result sets.
