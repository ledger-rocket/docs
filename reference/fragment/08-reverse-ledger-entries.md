# Reverse Ledger Entries

To address errors, you can reverse a posted Ledger Entry in Fragment. Reversing creates a new entry that offsets the original.

## Reversing Entries

Call the `reverseLedgerEntry` mutation to reverse a posted Ledger Entry:

```graphql
mutation ReverseLedgerEntry($id: ID!) {
  reverseLedgerEntry(id: $id) {
    __typename
    ... on ReverseLedgerEntryResult {
      reversingLedgerEntry {
        ik
        id
        type
        posted
        created
        reverses {
          id
          created
        }
        reversalPosition
      }
      reversedLedgerEntry {
        ik
        id
        type
        posted
        created
        reversedBy {
          id
          created
        }
        reversalPosition
        reversedAt
      }
      isIkReplay
    }
    ... on Error {
      code
      message
    }
  }
}
```

**Variables:**

```json
{
  "id": "<ID of the entry to reverse>"
}
```

### Key Points

You cannot reverse using `ik` because the reversing entry, reversed entry, and any subsequent reposting share the same `ik`, ensuring idempotency. Attempting to reverse a reversing entry returns `isIkReplay: true`.

**The reversing entry includes:**

- Ledger lines with opposite amount values in the original currency
- The same posted timestamp as the original (ensuring net balance change is zero at that time)
- A unique created timestamp reflecting the reversal time
- Identical tags and groups from the original
- A `reverses` field pointing to the original entry
- A `reversalPosition` value one greater than the original (1-indexed)

**The reversed entry is updated with:**

- A `reversedBy` field referencing the reversing entry
- A `reversedAt` timestamp matching the reversing entry's creation time

After reversal, both entries become immutable and cannot be modified via `updateLedgerEntry`.

## Reposting Entries

Once reversed, an entry can be reposted using the same `ik`. Use `addLedgerEntry` or `reconcileTx`:

```graphql
mutation AddLedgerEntry(
  $ik: SafeString!
  $entry: LedgerEntryInput!
) {
  addLedgerEntry(ik: $ik, entry: $entry) {
    __typename
    ... on AddLedgerEntryResult {
      entry {
        type
        created
        posted
        reversalPosition
      }
      lines {
        amount
        key
        description
        account {
          path
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

The newly posted entry receives a `reversalPosition` value one greater than the previous reversal entry.

## Querying Reversals

Use the `ledgerEntryHistory` query to retrieve the full history of an entry with reversals. Direct lookups by IK for reversing or reversed entries return a `ledger_entry_not_found` error.

Ledger entries have a `reversalHistory` field that lists all entries sharing an `ik` along with their reversals.

## Hidden Filter

Once reversed, entries are hidden from your standard entry list. For debugging purposes, you can query hidden entries separately using the appropriate filter option.
