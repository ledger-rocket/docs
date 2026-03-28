# Reconcile Payments

## Overview

Payment reconciliation in Fragment involves two primary steps:

1. **Sync payments** from external financial systems into Fragment using Links
2. **Record them** in your Ledger using the `reconcileTx` function

## Linked Ledger Accounts

To represent external accounts within your Ledger, you must create a Ledger Account and connect it to an External Account. Any Ledger Line posting to a linked account must correspond with a synced transaction in its matching External Account.

### Configuration Methods

**Hardcoded IDs:**
Set `linkedAccount` on a Ledger Account in your Schema with both `linkId` (from dashboard Link creation) and `externalId` (from your external financial system).

**Environment Variables:**
Use `${ENV_VAR}` syntax for different External Accounts per environment, which the CLI automatically replaces during `fragment store-schema` execution.

**Parameterized Templates:**
For dynamic scenarios like per-customer bank accounts, parameterize the `linkedAccount` field with `{{BANK_LINK_ID}}` syntax. These parameters become required when posting Ledger Entries.

## Reconciling Transactions

Follow the same two-step process as posting regular Ledger Entries:

1. Define the Ledger Entry structure in your Schema
2. Post using the API via `reconcileTx` mutation

Ledger Lines posting to linked accounts must specify the external transaction using `"tx": { "externalId": "{{bank_transaction_id}}" }`. This identifier must match exactly one synced transaction.

### Key Features

- **Idempotency:** The `Tx.externalId` ensures reconciliation operations are idempotent
- **Timestamps:** Entry timestamps derive from the transaction to maintain consistency with external systems
- **1:1 Matching:** Each Line posting to a linked account requires exact correspondence with one reconciled transaction

## Reconciling Multiple Transactions

Some payment types (like book transfers) generate multiple transactions at your bank. You can reconcile multiple transactions in a single call if they share:

- Identical `posted` timestamps
- Same Link origin

This approach consolidates related transactions while maintaining audit trails.

## Unreconciled Transactions

Transactions synced to Fragment but not yet reconciled represent potential reconciliation gaps. Query these using the `unreconciledTxs` field on `LedgerAccount`:

```graphql
ledgerAccount(ledgerAccount: {path: "assets/operating"}) {
  unreconciledTxs {
    nodes { id, description, amount, externalId }
  }
}
```

Note: Results are eventually consistent; recent transactions may not appear immediately after syncing.
