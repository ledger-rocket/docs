# Post Ledger Entries

## Overview

Posting a Ledger Entry involves two steps: defining its structure in your Schema, then posting it via the API.

## Schema Definition

Ledger Entries are configured under the `ledgerEntries.types` key in your Schema. Each entry type consists of:

- **Type**: Unique identifier for the entry
- **Description**: Human-readable explanation (supports template variables)
- **Lines**: Array of debit/credit entries

### Parameterization

Line amounts support handlebar syntax for dynamic values: `{{variable_name}}`. Basic arithmetic operations (+/-) are supported:

```text
"amount": "{{funding_amount}} - {{fee_amount}}"
```

Entries must balance according to the Accounting Equation or the schema validation fails.

## API Posting

Call the `addLedgerEntry` mutation with:

- **ledger.ik**: Ledger identifier
- **type**: Entry type name
- **typeVersion**: Optional version (defaults to 1)
- **parameters**: Variable values for the entry
- **posted**: Timestamp when the transaction occurred

## Key Concepts

### Minor Units

All monetary amounts are integers representing the smallest currency unit, encoded as strings. For example, USD $2.50 becomes `"250"`.

### Idempotency

Provide an `ik` (idempotency key) parameter to prevent duplicate postings when retrying failed requests.

### Timestamps

Entries have two timestamps:

- **posted**: When the business event occurred (you provide this)
- **created**: When posted to the API (auto-generated)

You may post entries with past or future timestamps; balances update retroactively from the posted date forward.

### Net Amounts

Configure how lines aggregate before posting via the `postLinesAs` field:

- `net_amounts`: Combines lines for the same account/currency/transaction
- `skip_zero_lines`: Posts as-is, skips zero amounts unless all are zero
- `raw_lines`: Posts exactly as provided

## Linking External Accounts

For reconciliation with external systems, include the `tx` field with an `externalId` on Ledger Lines. Use `reconcileTx` mutation instead of `addLedgerEntry` for linked entries.

## Multi-Currency Support

For accounts with `currencyMode: multi`, specify currency on each line:

```json
{
  "currency": {
    "code": "USD"
  }
}
```

## Repeated Lines

Post variable numbers of lines using the `repeated` configuration. Specify `repeated.key` to reference a parameter array. Each array element expands into separate lines.

Restrictions:

- Only one repeated key per entry type
- Each group must have minimum 2 balancing lines
- All array elements must contain identical keys

## Entry Conditions

Define rules in your Schema to enforce correctness. Conditions check preconditions or postconditions on account balances. If unmet, the entry fails with `BadRequestError`.

## Tags

Store up to 10 arbitrary key-value pairs on entries. Define tags in Schema or add them at posting time.

## Groups

Associate related entries using groups configured in your Schema or added later via `updateLedgerEntry`.

## Updating Entries

**Limited updates**: Use `updateLedgerEntry` to modify tags and groups only.

**Corrections**: Reverse the entry with `reverseLedgerEntry`, then post a corrected version to change amounts, accounts, or timestamps.

## Runtime Entries

Omit the `lines` field in your Schema definition to support runtime-defined entries. Provide the line structure when posting via API instead.
