# Query Data

## Basics

### GraphQL

Fragment employs a GraphQL API where data is structured as a graph. Entities serve as nodes, and relationships between entities form edges. The API exposes several query entry points that retrieve either single entities or lists.

Expansions allow you to fetch related entities in nested queries, enabling retrieval of associated data in one request rather than multiple round-trips. For example, expanding from a Ledger Entry retrieves all contained Ledger Lines in a single call.

### Connection Types

Fragment uses connection types to return entity lists, consisting of nodes and a `pageInfo` object containing cursors for pagination.

```graphql
query ListLedgerAccounts($ledger: LedgerMatchInput!) {
  ledger(ledger: $ledger) {
    ledgerAccounts {
      nodes {
        name
        type
      }
      pageInfo {
        hasNextPage
        endCursor
        hasPreviousPage
        startCursor
      }
    }
  }
}
```

### Filtering

Connection types support `filter` arguments to narrow results. Multiple filter components combine with AND logic.

Example filter by account type:

```json
{
  "ledger": {"ik": "quickstart-ledger"},
  "filter": {"type": {"equalTo": "asset"}}
}
```

Combine multiple filters:

```json
{
  "ledger": {"ik": "quickstart-ledger"},
  "filter": {
    "type": {"equalTo": "asset"},
    "hasParentLedgerAccount": true
  }
}
```

### Pagination

List fields support cursor-based pagination:

- Results appear under `nodes` as arrays
- `pageInfo` provides cursors for next and previous pages
- Send cursors to `after` or `before` arguments
- `first` or `last` arguments set page size (default: 20, maximum: 200)
- Page size must remain consistent across pagination requests
- Results follow deterministic ordering, typically reverse chronological

Variables for first page:

```json
{
  "ledgerIk": "ik-used-to-create-ledger",
  "first": 2
}
```

For subsequent pages, use the `endCursor`:

```json
{
  "ledgerIk": "ik-used-to-create-ledger",
  "after": "<some-end-cursor>"
}
```

## Ledgers

### Lookup

Retrieve a Ledger using the IK from creation:

```graphql
query GetLedger($ledger: LedgerMatchInput!) {
  ledger(ledger: $ledger) {
    name
    created
    balanceUTCOffset
    ledgerAccounts {
      nodes {
        name
        type
      }
    }
    schema {
      key
    }
  }
}
```

Variables:

```json
{
  "ledger": {"ik": "quickstart-ledger"}
}
```

### List

Retrieve all Ledgers in your workspace:

```graphql
query ListLedgers {
  ledgers {
    nodes {
      name
      created
      balanceUTCOffset
      ledgerAccounts {
        nodes {
          name
          type
        }
      }
      schema {
        key
      }
    }
    pageInfo {
      hasNextPage
      endCursor
      hasPreviousPage
      startCursor
    }
  }
}
```

## Ledger Accounts

### Lookup

Retrieve a Ledger Account by its schema path:

```graphql
query GetLedgerAccount($ledgerAccount: LedgerAccountMatchInput!) {
  ledgerAccount(ledgerAccount: $ledgerAccount) {
    path
    name
    balance
    type
    lines {
      nodes {
        amount
        posted
      }
    }
  }
}
```

Variables:

```json
{
  "ledgerAccount": {
    "path": "assets/banks/user-cash",
    "ledger": {"ik": "quickstart-ledger"}
  }
}
```

### Lookup Multiple

Retrieve multiple accounts using the `in` filter:

```json
{
  "ledger": {"ik": "quickstart-ledger"},
  "filter": {
    "ledgerAccount": {
      "in": [
        {"path": "assets/banks/user-cash"},
        {"path": "income-root/income-revenue-root"}
      ]
    }
  }
}
```

### List

List all Ledger Accounts within a Ledger using the `ledgerAccounts` field on a `ledger` query.

### Filter by Type

Single type:

```json
{
  "filter": {"type": {"equalTo": "asset"}}
}
```

Multiple types:

```json
{
  "filter": {"type": {"in": ["asset", "liability"]}}
}
```

### Filter by Path

Use wildcard matching (`*`) for template variables:

```json
{
  "filter": {"path": {"matches": "liability-root/user:*/pending"}}
}
```

### Filter by Link

Filter Ledger Accounts by their linked External Accounts:

```json
{
  "filter": {
    "linkedAccount": {
      "in": [
        {"linkId": "<link id>", "externalId": "account-id-at-bank"},
        {"linkId": "<link id>", "externalId": "account2-id-at-bank"}
      ]
    }
  }
}
```

### Filter by Parent

Filter accounts by parent status:

```json
{
  "filter": {"hasParentLedgerAccount": false}
}
```

## Ledger Lines

### Lookup

Retrieve a Ledger Line by ID:

```graphql
query GetLedgerLine($ledgerLine: LedgerLineMatchInput!) {
  ledgerLine(ledgerLine: $ledgerLine) {
    amount
    currency {
      code
      customCurrencyId
    }
    account {
      name
      type
    }
  }
}
```

### List

List lines in a Ledger Account via the `lines` field on a `ledgerAccount` query.

### Filter by Posted

Filter lines by timestamp range (both `after` and `before` are inclusive):

```json
{
  "filter": {
    "posted": {
      "after": "1969-07-01T00:00:00.000Z",
      "before": "1969-07-30T23:59:59.999Z"
    }
  }
}
```

### Filter by Key

Filter lines by their schema keys:

```json
{
  "linesFilter": {
    "key": {"equalTo": "increase_user_balance"}
  }
}
```

## Ledger Entries

### Lookup

Retrieve a Ledger Entry by its IK (provided during posting):

```graphql
query GetLedgerEntry($ledgerEntry: LedgerEntryMatchInput!) {
  ledgerEntry(ledgerEntry: $ledgerEntry) {
    id
    ik
    ledger {
      id
      name
    }
    lines {
      nodes {
        amount
        currency {
          code
          customCurrencyId
        }
      }
    }
  }
}
```

Variables by IK:

```json
{
  "ledgerEntry": {"ik": "<ledger entry IK>"}
}
```

Or by ID:

```json
{
  "ledgerEntry": {"id": "<ledger entry ID>"}
}
```

### Lookup Multiple

Retrieve multiple entries using the `in` filter:

```json
{
  "filter": {
    "ledgerEntry": {
      "in": [
        {"ik": "fund-user-1-account"},
        {"ik": "fund-user-2-account"}
      ]
    }
  }
}
```

### List by Group

Retrieve paginated entries within a group using `ledgerEntryGroup.ledgerEntries`:

```json
{
  "ledgerEntryGroup": {"key": "withdrawal", "value": "12345"},
  "ledger": {"ik": "quickstart-ledger"}
}
```

### List Hidden

Include hidden entries (those reversed) by setting `showHidden: true` in the filter.

### Expand Lines

Retrieve all Ledger Lines within an entry via the `lines` connection field.

### Filter by Posted

By timestamp range:

```json
{
  "entriesFilter": {
    "posted": {
      "after": "1968-01-01",
      "before": "1969-01-01"
    }
  }
}
```

By specific date:

```json
{
  "entriesFilter": {
    "date": {"equalTo": "12-31-1968"}
  }
}
```

### Filter by Type

Single type:

```json
{
  "entriesFilter": {"type": {"equalTo": "withdrawal"}}
}
```

Multiple types:

```json
{
  "entriesFilter": {"type": {"in": ["withdrawal", "p2p_transfer"]}}
}
```

### Filter by typeVersion

```json
{
  "entriesFilter": {
    "type": {"equalTo": "withdrawal"},
    "typeVersion": {"equalTo": "2"}
  }
}
```

### Filter by Group

Find entries in specific group:

```json
{
  "entriesFilter": {
    "group": {
      "equalTo": {"key": "department", "value": "sales"}
    }
  }
}
```

Additional group filter operators: `in`, `notEqualTo`, `notIn`, `keyEqualTo`, `notKeyEqualTo`, `keyIn`, `notKeyIn`.

### Filter by Tag

Find entries with specific tag:

```json
{
  "entriesFilter": {
    "tag": {
      "equalTo": {"key": "user_id", "value": "user-1"}
    }
  }
}
```

Additional tag filter operators: `in`, `notEqualTo`, `notIn`, `contains`, `keyEqualTo`, `notKeyEqualTo`, `keyIn`, `notKeyIn`.

### Filter by Reversal

Find reversing entries:

```json
{
  "entriesFilter": {"isReversal": true}
}
```

For any entry, only one of `isReversal` or `isReversed` is true.

## Ledger Entry Groups

### Lookup

Retrieve a group by key and value:

```json
{
  "entryGroup": {"key": "withdrawal", "value": "12345"},
  "ledger": {"ik": "quickstart-ledger"}
}
```

### List

List all groups via `ledgerEntryGroups` on a `ledger` query. Results are paginated.

### Filter

Filter groups by `key`, `value`, `created`, and/or `balance`. The `balance` filter finds group values where account balance matches conditions -- useful for finding unpaid invoices represented by non-zero balances.

When filtering by balance, `key` is required and `created` cannot be used:

```json
{
  "filter": {
    "key": {"equalTo": "invoice"},
    "balance": {
      "account": {"path": {"equalTo": "liabilities/user-funds"}},
      "currency": {"equalTo": {"code": "USD"}},
      "ownBalance": {"ne": "0"}
    }
  }
}
```

## Links

### Lookup

Retrieve a Link by ID. The `__typename` indicates the Link type.

### External Accounts

List External Accounts represented by a Link via `externalAccounts` connection. Results are paginated.

## External Accounts

### Lookup

Retrieve an External Account by external ID and Link ID, or by Fragment ID.

### Txs

List Txs synced to an External Account via the `txs` connection. Results are paginated.

### Linked Accounts

List Ledger Accounts linked to an External Account via `ledgerAccounts` connection. Results are paginated.

## Txs

### Lookup

Retrieve a Tx by external IDs (externalId, externalAccountId, linkId) or by Fragment ID.

### Unreconciled

List unreconciled Txs for a Ledger Account via the `unreconciledTxs` connection on `ledgerAccount`. Results are paginated.

## Schemas

### Lookup

Retrieve a Schema by key. Use the `version` argument to query specific Schema versions; by default, the latest is returned. The `json` field returns the Schema JSON.

### List Versions

Query all Schema versions via the `versions` connection. Results are paginated.

### List Ledgers

List Ledgers created from a Schema via the `ledgers` connection.

### Migration Status

Query Ledger migration status when Schema updates via `version.migrations`. Returns migration status per Ledger.
