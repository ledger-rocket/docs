# Sync Payments

## Overview

Payments made by your product need to be recorded into your Ledger. This section describes the first step in a two-step process: syncing payments from external systems into Fragment using Links, followed by reconciliation.

## Types of Links

The system supports two categories:

**Native Links** are built-in integrations that synchronize automatically through periodic syncing, webhooks, and just-in-time updates during reconciliation.

**Custom Links** provide APIs enabling you to build sync processes with any external financial system.

Each external financial system account has a corresponding External Account, which contains transactions (Txs). After setting up a Link, you create Ledger Accounts linked to each External Account.

## Native Link Integrations

### Increase Integration

To establish a Native Link with Increase:

1. Create a Link in the dashboard and select Increase
2. Choose between sandbox or production environment
3. Approve the connection when redirected to Increase

### Stripe Integration

Setting up Stripe involves these steps:

1. Create a Link and select Stripe
2. Select your test or live environment
3. Approve the connection via Stripe redirect

**Fee Handling**: Stripe Balance Transactions generate two transaction entries -- one for the gross amount and another for fees. These are identified as `{{stripe_tx_id}}_gross` and `{{stripe_tx_id}}_fee`, allowing independent accounting.

**Stripe Connect Support**: Enable this by creating a Restricted Access Key with specific permissions for Balance, Balance Transaction Sources, Balance Transfers, Connect Resources, and Webhook Resources.

### Unit Integration

To create a Native Link with Unit:

1. Generate an API Key at Unit with `accounts` and `transactions` scopes
2. Create a Link in the dashboard selecting Unit
3. Enter your Unit Org ID and API Key

Note: Pre-approval from Unit is required to enable this Native Link.

## Custom Links

Custom Links let you build integrations with any external system using APIs rather than automatic syncing.

### Onboarding

Create a Link via the dashboard or using the `createCustomLink` mutation with parameters for name and integration key.

### Account Synchronization

Call `syncCustomAccounts` to sync accounts. Use stable, unique `externalId` values ensuring idempotent operations. Updating with a different name for existing accounts updates the External Account name.

### Transaction Synchronization

Use `syncCustomTxs` to sync settled transactions. Only sync settled transactions, not pending or declined ones. The `externalId` must be unique within the account scope, typically the transaction ID from the external system. You can sync transactions from multiple accounts in one call if they belong to the same Custom Link.

### Updating Transactions

While `amount` and `posted` timestamp are immutable, you can update `description` by resyncing with the same `externalId`. To modify immutable fields, delete and resync the transaction.

### Deleting Transactions

Call `deleteCustomTxs` with Fragment IDs (not external IDs). Deleted transactions can be resynced with updated values. Constraints include: cannot delete reconciled transactions, maximum 100 per call, and up to 10 delete-resync cycles per transaction.
