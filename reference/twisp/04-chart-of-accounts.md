# Chart of Accounts

> Source: https://www.twisp.com/docs/accounting-core/chart-of-accounts

## Accounts Overview

Accounts are "a named store of value" that maintain both balances and activity records through ledger entries. In Twisp, accounts track activity across multiple layers and support multiple journals, with all changes stored as immutable, versioned documents.

For example, in a wallet product, each customer's wallet becomes an account, and the complete collection forms the chart of accounts.

## Organizing Accounts with Sets

Rather than using a single hierarchical tree structure, Twisp employs AccountSets -- "custom groups of accounts that aggregate balances and provide a unified interface into the entries for all accounts." A key advantage is that sets can contain other sets, enabling complex organizational structures.

**Important constraint:** "Entries can only be posted to accounts, not to account sets."

## Balances Roll Up Entries

All accounts have corresponding Balances that summarize posted entries. Balances consist of:

- Debit balance
- Credit balance
- Normal balance (the net figure)

For account sets: "The balance of account set X is always equal to the sum of all entries posted to accounts in X plus the balances of all account sets in X."

## Credit Normal and Debit Normal

Accounts differentiate between credit-normal and debit-normal types:

- **Credit-normal** accounts use `credits - debits` (used for revenue, liability, equity)
- **Debit-normal** accounts use `debits - credits` (used for assets and expenses)

The system imposes "no strict rules about which accounts can or should be credit or debit normal."

## Creating Accounts

Twisp supports charts of accounts for multiple use cases: Digital Banking, Acquiring/Marketplace, Lending, and Currency Exchange.

Common settlement/platform accounts include:

- Suspense
- ACH Settlement
- Bill Pay Settlement
- Charge-off
- Fraud Loss
- Card Disputes
- Interchange Revenue
- Collected Fee Revenue
- FBO (For Benefit Of)
- Cash
