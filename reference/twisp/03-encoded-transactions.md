# Encoded Transactions (Transaction Codes)

> Source: https://www.twisp.com/docs/accounting-core/encoded-transactions

## Overview

Transactions are the fundamental mechanism for recording accounting events in the ledger. "In Twisp, the only way to write to a ledger is through a transaction." Each transaction creates at least two entries following double-entry accounting principles.

## Transactions and Entries

An entry represents one side of a transaction and includes an account, amount, and direction (DEBIT or CREDIT). Twisp assigns categorical types to entries for better organization and tracking.

The system enforces integrity through validity checks, ensuring debits and credits balance to prevent value creation or loss. This strict definition maintains consistency across the ledger record.

## Composable Double-Entry Accounting

Transaction codes (tran codes) function as templates that define how transactions interact with the ledger. Each code specifies entry patterns as predictable formulas, enabling engineers to implement standardized transaction types efficiently.

## Tran Codes in Practice

### Example: ACH Credit

Consider an ACH deposit scenario requiring:

- A credit entry to "ACH Settlement" account
- A debit entry to customer account
- Equal amounts for both entries

### Creating a Tran Code

The GraphQL mutation defines:

- `tranCodeId`: Unique identifier
- `code`: "ACH_CREDIT"
- `params`: Input variables (account, amount, effectiveDate)
- `entries`: Ledger entries using CEL expressions for dynamic values

Key feature: `params.amount` allows runtime value substitution when posting transactions.

### Posting a Transaction

With a defined tran code, posting becomes simple -- provide the code and parameters. A $12.87 deposit uses the template to automatically create two balanced entries.

### Example Tran Codes

| TranCode | Description | Entry Types |
|----------|-------------|------------|
| WIRE_TRANSFER | Bank-to-bank transfer | WIRE_OUTGOING_DR, WIRE_INCOMING_CR |
| CARD_HOLD_CANCEL | Hold cancellation | CARD_HOLD_CANCEL_DR, CARD_HOLD_CANCEL_CR |
| DEPOSIT | Bank deposit | DEPOSIT_DR, DEPOSIT_CR |
| BILL_PAYMENT | Bill payment | BILL_PAYMENT_DR, BILL_PAYMENT_CR |

## Timing and Sequencing

### Effective Dates

Two time values matter:

- **Created**: Wall-clock time transaction posts
- **Effective**: Accounting date the transaction applies to

These differ in scenarios like weekend ACH posts with Monday effective dates.

### Entry Sequences

Entries post in defined order within tran codes. The system records this as a `sequence` on each entry. Despite simultaneous database-level posting, sequence clearly shows entry order within transactions.

## Embedding Meaning and Context

### Correlation Identifiers

Related transactions share correlation IDs for grouped tracking. Without explicit assignment, a transaction uses its own ID. Related transactions reference this ID to show relationships.

Example: Card processing involves hold (transaction 1), release (transaction 2), and settlement (transaction 3) -- all sharing correlation ID 1.

### Transaction Metadata

The `metadata` field stores arbitrary JSON data without affecting accounting operations, providing flexibility for application-specific information.
