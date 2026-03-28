# Ledgers in Twisp

> Source: https://www.twisp.com/docs/accounting-core/ledgers-in-twisp

## What is a Ledger?

Financial ledgers form the backbone of money-tracking systems. Double-entry accounting has served as the standard mechanism for recording financial transactions for over 500 years, with ledgers at its core.

Key uses include:

- Banks managing debits and credits across multiple accounts with various financial instruments
- Payment infrastructure companies requiring robust multi-tenant account servicing
- Accounting software needing strong transactional guarantees and continuous data availability

## Ledgers in Twisp

Twisp reimagines financial ledger technology by merging "the operational and scaling characteristics of a distributed database with the correctness guarantees offered by relational databases."

The platform's ledger balances flexibility for any financial product with structural reliability for trustworthy financial data storage.

## Architecture

Twisp accounts can provision multiple accounting core instances, each powered by a single ledger composed of five elements:

### Journals

Journals organize transactions into separate "books." Twisp provides a default journal with code `DEFAULT`. Users can create separate journals for different currencies or product-specific purposes.

### Accounts

A chart of accounts models all economic activity within the ledger, forming the basis for balance sheets, P&L reports, and understanding entity balances.

### Layered Entries

Entries represent one side of a ledger transaction. Twisp enforces double-entry accounting, requiring entries within a Transaction context, creating at least two ledger entries per transaction.

Entries contain: account, amount, direction (CREDIT/DEBIT), type (e.g., "TRANSFER_DR"), and layer assignment (SETTLED, PENDING, ENCUMBRANCE).

### Transactions

Transactions record all accounting events. In Twisp, all ledger writes occur through transactions using double-entry principles, creating two or more entries.

Transaction codes (tran codes) encode repeatable transaction patterns as predictable formulas.

### Balances

Balances are auto-calculated entry sums for given accounts. Each account maintains balances across all three layers with separate debit and credit amounts.

## Architected from Principles

The ledger adheres to established accounting principles:

- **Enforce double-entry principles**: Ensure money is never created or destroyed -- every transaction has debit and credit sides.
- **Preserve history**: Ledgers remain immutable and append-only, maintaining complete change history.
- **Contextualize transactions**: Use correlations, layers, and metadata to preserve context across transaction lifecycles.
- **Design for composability**: Encode higher-order accounting logic through transaction codes and automations.
- **Rely on industry standards**: Apply well-established patterns like layers and typed entries.
