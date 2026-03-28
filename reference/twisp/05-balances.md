# Balances

> Source: https://www.twisp.com/docs/accounting-core/balances

## Accounts and Balances

Balances represent auto-calculated sums of entries for a given account. Each balance record maintains a `drBalance` for DEBIT entries and a `crBalance` for CREDIT entries.

A `normalBalance` is calculated as the difference between credits and debits for credit normal accounts, or debits and credits for debit normal accounts.

### A Balance for Every Journal, Currency, and Layer

Accounts maintain separate balances for every journal, currency, and for each of three layers: `SETTLED`, `PENDING`, and `ENCUMBRANCE`.

In a simple single-journal, single-currency ledger using only `SETTLED` and `PENDING` layers, accounts will have two balances total. More complex ledgers with multiple journals and currencies generate additional balances. For example, a ledger with 2 journals, 2 currencies, and all 3 layers produces 12 balances per account (2 x 2 x 3).

## Balance Calculations

Balances in Twisp are derived from ledger entries through calculations performed at write time, not read time. This ensures "100% certainty that the balance reflects the current state of the ledger."

### Calculating Normal Balance

Consider a ledger with two accounts: **Cash** (debit normal) and **Revenue** (credit normal).

Given these entries:

| Entry ID | Account | Amount | Direction |
|----------|---------|--------|-----------|
| 1 | Revenue | $500 | CREDIT |
| 2 | Cash | $500 | DEBIT |
| 3 | Revenue | $400 | DEBIT |
| 4 | Cash | $400 | CREDIT |
| 5 | Revenue | $250 | CREDIT |
| 6 | Cash | $250 | DEBIT |

The resulting balances are:

| Account | CR Balance | DR Balance | Normal Balance |
|---------|-----------|-----------|----------------|
| Cash | $400 | $750 | $350 |
| Revenue | $750 | $400 | $350 |

The sum of all credits equals all debits, maintaining double-entry accounting principles.

## Debits and Credits (DR/CR)

Each journal entry debits or credits an account with a signed amount value. Both negative and positive debits or credits are possible.

This approach makes "the debit and credit balances for an account meaningful" by preserving accurate representations of account activity.

### Example: Correcting Transaction Errors

When a $1000 deposit should have been $1200, the immutable ledger requires voiding the original entry and reposting the correction:

| Entry ID | Account ID | Type | Debit | Credit |
|----------|-----------|------|-------|--------|
| 1 | f29f83 | DEPOSIT | - | $1000.00 |
| 2 | f29f83 | VOID_DEPOSIT | - | $(1000.00) |
| 3 | f29f83 | DEPOSIT | - | $1200.00 |

**Result:** Debit Balance $0.00, Credit Balance $1200.00, Normal Balance $1200.00

An alternative using signed debits and credits would produce misleading results:

| Entry ID | Account ID | Type | Debit | Credit |
|----------|-----------|------|-------|--------|
| 1 | f29f83 | DEPOSIT | - | $1000.00 |
| 2 | f29f83 | VOID_DEPOSIT | $1000.00 | - |
| 3 | f29f83 | DEPOSIT | - | $1200.00 |

**Result:** Debit Balance $1000.00, Credit Balance $2200.00, Normal Balance $1200.00

This incorrect method loses meaning by creating misleading debit and credit balances that don't reflect actual account movement.

**Note:** All examples use the "SETTLED" layer for simplicity.
