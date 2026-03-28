# Formance Accounting Model

> Sources:
> - <https://docs.formance.com/modules/ledger/accounting-model/introduction>
> - <https://docs.formance.com/modules/ledger/accounting-model/source-destination-accounting-model>
> - <https://docs.formance.com/modules/ledger/accounting-model/transactions>
> - <https://docs.formance.com/modules/ledger/accounting-model/accounts>
> - <https://docs.formance.com/modules/ledger/accounting-model/constraints>

## Overview

The Formance Ledger accounting framework represents financial movements within a business.
Rather than using traditional double-entry bookkeeping, Formance implements a
**source-destination model**.

## Source/Destination Model

### Key Benefits

1. **Balance verification**: You make sure your pay-in is correct, then the ledger ensures
   you don't use more than you have available
2. **Inventory management**: Functions as an inventory system for managing cash flows
3. **Fund integrity**: Prevents creation of funds without genuine backing

### Account Structure

Accounts serve as containers holding multiple asset types, each tracked independently (USD,
EUR, BTC, etc.).

Each asset maintains two components:

- **Source** (input): Funds added to the account
- **Destination** (output): Funds removed from the account

These are append-only: you increase source to add funds, increase destination to remove them.

### Balance Calculation

```text
Balance = Destination - Source
```

Or equivalently in API terms:

```text
Balance = Input - Output
```

### Volumes vs. Balances

- **Volumes**: Total input/output amounts processed as unsigned integers (never negative)
- **Balances**: Available account amounts that can be positive, negative, or zero

The ledger records transaction volumes rather than balances directly; balances emerge from
computing input minus output differences.

## Transactions

A transaction represents a set of modifications applied to a set of accounts at a specific
time. All modifications within a transaction occur atomically.

### Postings

A posting models the movement of an amount of an asset from one account to another:

```json
{
  "source": "alice",
  "destination": "bob",
  "asset": "COIN",
  "amount": 100
}
```

### Multi-Posting Transactions

Formance wraps postings within transactions to enable atomic commitment:

```json
{
  "postings": [
    {
      "source": "alice",
      "destination": "teller",
      "amount": 100,
      "asset": "COIN"
    },
    {
      "source": "teller",
      "destination": "alice",
      "amount": 5,
      "asset": "GEM"
    }
  ]
}
```

### Design Philosophy

Formance employs **single I/O postings with multi-posting transactions**:

- Multi-posting transactions leverage atomicity for handling complex financial flows
- Single I/O postings remain mentally intuitive and maintain auditability
- Complex multi-I/O scenarios are modeled using transient accounts within multi-posting
  transactions

## Accounts

An account serves as a container for assets that groups financial resources according to
business needs.

### Automatic Creation

Accounts do not need to be created prior to being used. Submitting a transaction involving
an account will automatically create it.

### Naming Conventions

Account addresses must follow the pattern: `^[a-zA-Z_0-9]+(:[a-zA-Z_0-9]+){0,}$`

Best practices:

- Use colons for structured organization
- Examples: `payments:001:authorizations:001`, `sales:001:contract`
- Use Ledger Schema for stricter naming enforcement

### The `@world` Account

A special account that can have a negative balance. Used as the entry point for introducing
capital into the ledger system.

### Account Metadata

Accounts support key-value metadata:

- Updates are additive and idempotent
- New keys are added; existing keys update
- Metadata updates never remove existing keys

## Constraints

Three constraints maintain ledger integrity:

### 1. Zero Ledger-wide Balance (Conservation)

The sum of all account sources must equal the sum of all destinations, preventing artificial
cash creation.

### 2. Balanced Transactions

Within any transaction, source additions must equal destination additions, preventing cash
destruction.

### 3. No Negative Balances

Accounts cannot go negative unless explicitly permitted (via overdraft directives or the
`@world` account), preventing arbitrary cash generation.

## Introducing Cash

### World Account Method

The designated `@world` account is permitted negative balances:

```numscript
send [USD/2 10000] (
  source = @world
  destination = @users:001
)
```

### Counter-Part Accounts Method

Alternative accounts representing payment methods absorb transactions:

```numscript
send [USD/2 5000] (
  source = @world
  destination = @payments:stripe
)

send [USD/2 5000] (
  source = @payments:stripe
  destination = @users:001
)
```
