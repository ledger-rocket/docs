# Design Your Ledger

## Overview

A Ledger uses a Schema to define functionality for a specific product and use case. A Schema may be shared across multiple Ledgers. Updating a Schema will trigger migrations to update each Ledger. Use the Ledger designer in the Dashboard to model and store your Schema.

Ledgers track money using:

- **Ledger Accounts**: balances that represent the financial state of a business
- **Ledger Entries**: financial events that update Ledger Accounts

## Ledger Accounts

A Ledger Account has a balance. Changes to a Ledger Account's balance are called Ledger Lines.

### Four Account Types

Ledger accounts are split into two layers:

**State Layer:**

- **Assets**: what you own
- **Liabilities**: what you owe

**Change Layer:**

- **Income**: what you've earned
- **Expense**: what you've spent

### Account Categories

State Ledger Accounts track your product's financial relationships with your bank, payment systems and users. Balance Sheets report State Ledger Account balances.

Change Ledger Accounts track when and how your product makes a profit or loss. They produce Income Statements.

### Schema Structure

Within a Schema, the `chartOfAccounts` key contains a nested tree of Ledger Accounts, up to a maximum depth of 10.

Ledger Accounts sharing a parent require a unique `key`, though the same `key` can appear in different tree sections.

### Optional Properties

For some Ledger Accounts, you must set additional properties:

- `linkedAccount`: enables reconciliation with external systems
- `template`: allows multiple instances on demand
- `currencyMode`: configures the account as `single` or `multi`
- `consistencyConfig`: sets balances and Ledger Lines as strongly or eventually consistent

## Ledger Entries

A Ledger Entry is a single update to a Ledger. Define a Ledger Entry type in your Schema for every financial event in your product and bank.

### Balanced Entries

A Ledger Entry must be balanced, which means it follows the Accounting Equation:

```text
Assets - Liabilities = Income - Expenses
```

How you balance a Ledger Entry depends upon its net effect to the Ledger's balances.

### Zero Net Change Example

When the net change to State Ledger Accounts is zero, the financial event did not change net worth. An increase to an asset account is balanced by an increase in a liability account.

### Non-Zero Net Change Example

When the net change to State Ledger Accounts is non-zero, the financial event made a profit or loss. A difference in asset and liability accounts is balanced by an increase in an income account.
