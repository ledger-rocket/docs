# Group Ledger Entries

## Overview

A Ledger Entry Group is a collection of related Ledger Entries that occur at different points in time. Each Group tracks the net change to each Ledger Account balance it affects.

Use Ledger Entry Groups to tie together Ledger Entries that are part of the same funds flow, such as a deposit, settlement or invoice. To store metadata, use tags instead.

## Configuring Groups

Groups for a Ledger Entry Type are defined as a list of key/value pairs in the Schema:

```json
{
  "type": "user_initiates_withdrawal",
  "description": "{{user_id}} initiates withdrawal",
  "lines": [
    {
      "account": {
        "path": "liabilities/users:{{user_id}}/available"
      },
      "key": "decrease_user_balance",
      "amount": "-{{withdraw_amount}}"
    }
  ],
  "groups": [
    {
      "key": "withdrawal",
      "value": "{{withdrawal_id}}"
    }
  ]
}
```

### Limitations

- You can specify up to 10 Groups for any Ledger Entry type
- Parameters can be used in the `value` of a Group, but not the `key`
- Group keys must be unique per Ledger Entry, and you can only provide one value for each key

## Querying Balances

Use the `ledgerEntryGroup.balances` expansion to get the net change per Ledger Account balance from all Ledger Entries in a Group. Group balances are eventually consistent.

### Basic Balance Query

```graphql
query GetLedgerEntryGroupBalances(
  $ledger: LedgerMatchInput!
  $entryGroup: EntryGroupMatchInput!,
) {
  ledger(ledger: $ledger) {
    ledgerEntryGroup(ledgerEntryGroup: $entryGroup) {
      balances {
        nodes {
          account {
            path
          }
          ownBalance
        }
      }
    }
  }
}
```

Variables:

```json
{
  "entryGroup": {
    "key": "withdrawal",
    "value": "some-withdrawal-id"
  },
  "ledger": {
    "ik": "quickstart-ledger"
  }
}
```

### Consistent Balances

Ledger Entry Groups support strongly-consistent reads for the ownBalance field. Configure this in your Schema for consistent retrieval of group balance information.

### Filtering Balances

Balances in a Group may be filtered by account, currency, and ownBalance:

```json
{
  "filter": {
    "currency": {
      "equalTo": { "code": "USD" }
    },
    "ownBalance": {
      "gte": "-1000",
      "lte": "1000"
    },
    "account": {
      "path": {
        "equalTo": "liability-root/user:user-id/pending"
      }
    }
  }
}
```

### Filter by Template

Group balances support filtering account paths using `*` in place of a template variable:

```json
{
  "filter": {
    "account": {
      "path": {
        "matches": "liability-root/user:*/pending"
      }
    }
  }
}
```

## Updating Entry Groups

In addition to Groups defined in your Schema, you can add a posted Ledger Entry to additional Groups using the `updateLedgerEntry` mutation.

```graphql
mutation UpdateLedgerEntryGroups(
  $ledgerEntry: LedgerEntryMatchInput!
  $update: UpdateLedgerEntryInput!
) {
  updateLedgerEntry(
    ledgerEntry: $ledgerEntry,
    update: $update
  ) {
    __typename
    ... on UpdateLedgerEntryResult {
      entry {
        type
        ik
        groups {
          key
          value
        }
      }
    }
    ... on Error {
      code
      message
    }
  }
}
```

### Update Behavior

This is an additive operation:

- If you specify a new Group, the Ledger Entry will be added to that Group
- If you don't specify an existing Group, the Ledger Entry will remain in that Group
- You may not modify an existing Group key or remove a Ledger Entry from a Group
- You can only update a Ledger Entry a maximum of 10 times
