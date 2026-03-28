# Migrate Data

## Overview

As your product requirements evolve, you'll need to update your Schema and migrate existing data. Fragment's Schema is immutable and versioned, enforcing backwards compatibility checks once deployed to a Ledger. Migrations enable you to safely update your Schema and Ledger data while maintaining compatibility, achieving zero downtime, and preserving immutable, auditable records.

By deploying a new Schema version, you can:

- Create new Entry Type versions
- Disable and archive old Entry type versions
- Migrate entries posted with archived Entry Type versions
- Disable and archive old Ledger Accounts
- Migrate entries posted to archived Ledger Accounts

## Updating Entry Types

When creating a new Entry Type, it's automatically marked as V1. Once deployed, Entry Type versions become immutable and cannot be deleted from your Schema. The Schema JSON includes `type` and `typeVersion` fields that must be unique together. All deployed Entry Type versions remain on the Schema to ensure backwards compatibility.

When posting entries, you can specify `typeVersion` to select which version to use. If unspecified, version 1 is the default.

## Disabling Entry Types

Once you've created a new Entry Type version, prevent postings of the previous version by setting its `status` field to `"disabled"` in your Schema.

When an Entry Type is disabled:

- Posting attempts result in a `BadRequestError`
- Existing entries remain unchanged and queryable
- The type can later be re-enabled or archived
- You can still reverse or migrate disabled entry type entries

## Archiving Entry Types

Set an Entry Type's `status` to `"archived"` to trigger migration. Archiving requires the type to be `disabled` for 45 seconds first. This creates a `LedgerEntryDataMigration` for each Ledger using the Schema, though it doesn't automatically migrate entries.

You can un-archive by setting `status` back to `"active"`, which deactivates the associated migration.

## Migrating Entries

When you deploy a Schema with a newly archived Entry Type, a `LedgerEntryDataMigration` is asynchronously created for each Ledger. To migrate:

1. Enumerate `ledgerEntries` on the relevant migration
2. Call `migrateLedgerEntry` for each entry with the new Entry Type version
3. Migration completes when the `ledgerEntries` array is empty

The `migrateLedgerEntry` mutation reverses the old entry and reposts it using the new version, maintaining complete auditability.

## Updating Accounts

### Consistency Configuration

Accounts can migrate between strong and eventual consistency by updating the `consistencyConfig` field. Once deployed, a `LedgerMigration` is created for each Schema Ledger to apply the update.

### Currency Mode

Single-currency Accounts can be upgraded to multi-currency by updating the `currencyMode` field. A `LedgerMigration` is created for each Ledger upon deployment.

Note: Multi-currency Accounts cannot be downgraded to single-currency. Provide the `currency` argument when reading balance fields of now multi-currency Accounts.

## Disabling Accounts

Set an Account's `status` to `"disabled"` to prevent new entries from being posted to it. This is useful when migrating to new account structures or preparing archived data migration.

When disabled:

- Posting attempts result in a `BadRequestError`
- All Entry Types posting to it must also be disabled
- Existing entries and balances remain unchanged and queryable
- Entries can still post to active child accounts
- The account can be re-enabled or archived later

## Archiving Accounts

Once disabled, you can archive an Account by setting `status` to `"archived"`. Archiving requires 45 seconds of `disabled` status first. This creates a `LedgerAccountDataMigration` for each Ledger, though it doesn't automatically migrate entries.

## Migrating Accounts

Fragment automatically creates a `LedgerAccountDataMigration` when archiving. Consider whether you can tolerate downtime on account balance reads -- this determines your migration strategy.

### Offline Migrations

For offline migrations, account balance is temporarily inaccurate:

1. Update Schema to disable old Account and all related Entry Types; create new versions
2. Update code to use new Entry Type versions
3. Update Schema to archive old Account and Entry Types
4. Query and migrate all entries using new Entry Type versions

### Online Migrations

For online migrations, account balance remains accurate throughout:

1. Create new Account and control Account; add dual-write Entry Type versions posting to both old and new accounts, plus a control account
2. Update code to use dual-write versions; migrate existing old entries to dual-write versions
3. Update code to use new Account balance and final Entry Type version posting only to new account
4. Disable and archive old Account and dual-write Entry Types; migrate remaining entries to final versions
