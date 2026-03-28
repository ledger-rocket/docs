# Configure Consistency

To maintain correctness at scale, Fragment's consistency mode enables you to establish safeguards without compromising throughput.

You can configure consistency within your Schema to make targeted tradeoffs between performance and data consistency guarantees.

## Ledger Configuration

Set `consistencyConfig` at the top level of your Schema to control how the Ledger Entries list query behaves:

- **`entries: eventual`**: For high-throughput Ledgers tolerating stale entry lists
- **`entries: strong`**: For lower-throughput Ledgers requiring strict consistency, such as reconciliation dashboards

By default, all Ledgers use `eventual` consistency.

```json
{
  "consistencyConfig": {
    "entries": "strong"
  },
  "chartOfAccounts": []
}
```

## Ledger Account Configuration

Configure balance consistency and line list behavior within individual Ledger Account definitions using `consistencyConfig`:

### Balance Updates

- **`ownBalanceUpdates: eventual`**: Suitable for reporting accounts needing high throughput
- **`ownBalanceUpdates: strong`**: Appropriate for transaction authorization requiring strict consistency

### Line Lists

- **`lines: eventual`**: For accounts tolerating stale transaction lists
- **`lines: strong`**: For end-user-facing transaction histories requiring current data

```json
{
  "accounts": [
    {
      "key": "user-balance",
      "template": true,
      "type": "asset",
      "consistencyConfig": {
        "ownBalanceUpdates": "strong",
        "lines": "eventual"
      }
    }
  ]
}
```

Set default consistency across all accounts using `chartOfAccounts.defaultConsistencyConfig`:

```json
{
  "chartOfAccounts": {
    "defaultConsistencyConfig": {
      "ownBalanceUpdates": "strong",
      "lines": "strong"
    },
    "accounts": []
  }
}
```

Child accounts inherit the parent's `consistencyConfig` setting.

## Balance Queries

Query strongly consistent `ownBalance` values by setting `consistencyMode: strong` during retrieval:

```graphql
query GetOwnBalances($ledgerAccount: LedgerAccountMatchInput!) {
  ledgerAccount(ledgerAccount: $ledgerAccount) {
    ownBalance(consistencyMode: strong)
    ownBalances(consistencyMode: strong) {
      nodes {
        amount
        currency { code }
      }
    }
  }
}
```

**Restrictions:**

- Only `ownBalance` supports strong consistency reads
- Querying strong consistency on eventually-consistent accounts returns errors
- Cannot combine `at` with `consistencyMode` for historical balances

## Entry Conditions

Entry conditions enforce correctness rules evaluated when posting Ledger Entries. Failed conditions prevent posting and return `conditional_request_failed` errors.

### Preconditions

Verify that a previously-read balance hasn't changed before posting:

```json
{
  "type": "pay-employee",
  "lines": [],
  "conditions": [
    {
      "account": { "path": "bank-account" },
      "precondition": {
        "ownBalance": { "eq": "{{current_balance}}" }
      }
    }
  ]
}
```

### Postconditions

Guarantee that writes never place accounts in invalid states:

```json
{
  "type": "pay-employee",
  "lines": [],
  "conditions": [
    {
      "account": { "path": "bank-account" },
      "postcondition": {
        "ownBalance": { "gte": "0" }
      }
    }
  ]
}
```

**Restrictions:**

- Apply only to `ownBalance` of directly-posted accounts
- Referenced accounts require `consistencyConfig.ownBalanceUpdates: strong`
- Linked accounts do not support conditions

## Repeated Conditions

When using repeated lines, apply conditions to variable account sets using the same `repeated` key:

```json
{
  "type": "batch_transfer",
  "lines": [
    {
      "key": "from",
      "account": { "path": "liabilities/users:{{from_user}}/available" },
      "amount": "-{{amount}}",
      "repeated": { "key": "transfers" }
    },
    {
      "key": "to",
      "account": { "path": "liabilities/users:{{to_user}}/available" },
      "amount": "{{amount}}",
      "repeated": { "key": "transfers" }
    }
  ],
  "conditions": [
    {
      "account": { "path": "liabilities/users:{{from_user}}/available" },
      "postcondition": { "ownBalance": { "gte": "0" } },
      "currency": { "code": "USD" },
      "repeated": { "key": "transfers" }
    }
  ]
}
```

Each array element generates both transfer lines and corresponding balance conditions.

## Consistent Groups

Configure accounts to have strongly-consistent `ownBalance` for specific group keys using `consistencyConfig.groups`:

```json
{
  "accounts": [
    {
      "key": "user-balance",
      "template": true,
      "type": "asset",
      "consistencyConfig": {
        "groups": [
          {
            "key": "invoice_id",
            "ownBalanceUpdates": "strong"
          }
        ]
      }
    }
  ]
}
```

## Query Consistent Groups

Retrieve strongly consistent group balances by setting `consistencyMode: strong` or `use_account`:

```graphql
query ListLedgerEntryGroupBalances(
  $ledger: LedgerMatchInput!
  $groupKey: SafeString!
  $groupValue: SafeString!
  $consistencyMode: ReadBalanceConsistencyMode
) {
  ledgerEntryGroup(ledgerEntryGroup: {
    ledger: $ledger,
    key: $groupKey,
    value: $groupValue,
  }) {
    balances {
      nodes {
        account { path }
        ownBalance(consistencyMode: $consistencyMode)
      }
    }
  }
}
```

By default, group balances are eventually consistent. Strong consistency reduces maximum throughput.

**Restrictions:**

- Group balances cannot be used in Entry conditions
- Querying strong consistency without configured group consistency returns errors
