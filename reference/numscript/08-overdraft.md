# Numscript Overdraft

> Source: <https://docs.formance.com/modules/numscript/reference/overdraft>

## Overview

The `overdraft` directive instructs the Numscript interpreter to permit account balances to
fall below zero, either with or without restrictions.

By default, accounts in Formance Ledger cannot go negative (except for the special `@world`
account). The overdraft directive overrides this behavior for specific accounts.

## Unbounded Overdrafts

Use the `allowing unbounded overdraft` clause to allow an account to go negative without
limits:

```numscript
send [USD/2 100] (
  source = @foo allowing unbounded overdraft
  destination = @bar
)
```

This will succeed even if `@foo` has a zero or insufficient balance.

## Bounded Overdrafts

Specify a maximum negative balance using the `allowing overdraft up to` clause:

```numscript
send [USD/2 100] (
  source = @foo allowing overdraft up to [USD/2 50]
  destination = @bar
)
```

This allows `@foo` to go negative by at most 50 USD/2. If the account has 60 USD/2, it can
send up to 110 USD/2 (60 balance + 50 overdraft allowance).

## The `overdraft()` Function (Experimental)

The `overdraft()` function returns the positive amount of overdraft when an account's balance
is negative, or 0 if the balance is positive.

### Example

```numscript
vars {
  monetary $acc_overdraft = overdraft(@account, USD/2)
}

send $acc_overdraft (
  source = @world
  destination = @account
)
```

### Behavior

- Balance of `[USD/2 -50]` returns `[USD/2 50]`
- Balance of `[USD/2 100]` returns `[USD/2 0]`

### Enabling the Feature

This is an experimental feature requiring two flags:

1. Experimental rewrite feature
2. `experimental-overdraft-function` flag

Enable via feature declaration:

```numscript
#![feature("experimental-overdraft-function")]

vars {
  monetary $acc_overdraft = overdraft(@account, USD/2)
}

send $acc_overdraft (
  source = @world
  destination = @account
)
```

## Backdated Transactions

When inserting backdated transactions with overdrafts, the ledger validates the final state
rather than intermediate states.
