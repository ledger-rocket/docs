# Numscript Save Statement

> Source: <https://docs.formance.com/modules/numscript/reference/save>

## Overview

The `save` directive prevents accounts from falling below a specified minimum balance
threshold. This minimum is deducted from the account's available balance for transaction
purposes.

## Basic Syntax

```numscript
save [ASSET/SCALE AMOUNT] from @account
```

## Basic Usage

```numscript
save [USD/2 100] from @merchants:1234

send [USD/2 500] (
  source = @merchants:1234
  destination = @payouts:T1891G
)
```

Even if `@merchants:1234` has a balance of [USD/2 500], this transaction fails because the
post-transaction balance would drop below the protected threshold of [USD/2 100].

## Multiple Source Accounts

When additional funding sources are available, the protected account behaves as if its balance
is reduced by the save amount:

```numscript
save [USD/2 100] from @merchants:1234

send [USD/2 500] (
  source = {
    @merchants:1234
    @world
  }
  destination = @payouts:T1891G
)
```

The system distributes funds as:

- [USD/2 400] from `@merchants:1234` (500 balance - 100 saved = 400 available)
- [USD/2 100] from `@world`

## Insufficient Funds Error

When the requested amount exceeds the available balance (after applying `save`), and no
fallback sources exist, the transaction fails with an `INSUFFICIENT_FUND` error:

```json
{
  "errorCode": "INSUFFICIENT_FUND",
  "errorMessage": "running numscript: script execution failed: account(s) @my_account had/have insufficient funds"
}
```

Example: With [GBP/2 120] balance and [GBP/2 100] saved, only [GBP/2 20] is available.

## Send All Balance with Save

### Balance Exceeds Save Amount

When balance surpasses the protected threshold, the wildcard `*` sends the difference:

```numscript
// Account balance: [GBP/2 120]
save [GBP/2 100] from @my_account

send [GBP/2 *] (
  source = @my_account
  destination = @world
)
```

Result: [GBP/2 20] transfers (120 - 100).

### Balance Less Than or Equal to Save Amount

If the balance is less than or equal to the saved amount, a transaction with amount 0 is
created:

```numscript
// Account balance: [GBP/2 80]
save [GBP/2 100] from @my_account

send [GBP/2 *] (
  source = @my_account
  destination = @world
)
```

A zero-amount transaction is still recorded in the ledger for audit purposes.

## Multi-Account Distribution with Save

When using multiple sources with `save`, funds are taken from the saved account up to its
available limit, then completed from other sources:

```numscript
// @account_a balance: [USD/2 150]
save [USD/2 100] from @account_a

send [USD/2 80] (
  source = {
    @account_a
    @account_b
  }
  destination = @destination
)
```

Distribution:

- `@account_a` contributes [USD/2 50] (available: 150 - 100 = 50)
- `@account_b` contributes [USD/2 30] (remaining needed)
