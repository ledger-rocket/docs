# Numscript Send Statement

> Source: <https://docs.formance.com/modules/numscript/reference/send>

## Overview

In Numscript, a posting models the movement of an amount of an asset from one account to
another. Postings are contained within transactions to guarantee atomic application of all
changes.

## Basic Syntax

```numscript
send [ASSET/SCALE AMOUNT] (
  source = @source_account
  destination = @destination_account
)
```

### Example

```numscript
send [COIN 100] (
  source = @world
  destination = @users:001
)
```

### Component Breakdown

- **Asset Notation**: Square brackets contain the asset using Unambiguous Monetary Notation (UMN)
  - Asset type (e.g., `COIN`, `USD`)
  - Optional scaling value (e.g., `/2` for cents)
  - Amount (e.g., `100`)
- **Source**: The account(s) from which funds are drawn
- **Destination**: The account(s) to which funds are sent

## Wildcard Balance (`*`)

Rather than specifying an exact amount, use an asterisk (`*`) to transfer all available
assets of a given type from source accounts:

```numscript
send [USD/2 *] (
  source = @order:1234
  destination = {
    10% to @platform:fees
    remaining to @merchant:5678
  }
)
```

10% of the balance is moved to one account, and the remaining 90% is moved to another.

## Multiple Send Statements

A program can contain multiple `send` statements, resulting in a single transaction with
multiple postings:

```numscript
send [USD/2 599] (
  source = @world
  destination = @payments:001
)

send [USD/2 599] (
  source = @payments:001
  destination = @rides:0234
)

send [USD/2 599] (
  source = @rides:0234
  destination = {
    85% to @drivers:042
    remaining to @platform:fees
  }
)
```

## Using Variables

```numscript
vars {
  monetary $price
}

send $price (
  source = @world
  destination = @users:001
)
```

## Transactions Are Not Idempotent

Transactions described in Numscript are not idempotent by default. Executing the same
transaction twice tells Formance Ledger to make two distinct transfers.
