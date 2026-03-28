# Numscript Sources

> Source: <https://docs.formance.com/modules/numscript/reference/sources>

## Overview

There are several options when it comes to deciding where the money should come from. The
`send` statement provides multiple ways to specify monetary sources.

## Single Source

The simplest way to send a monetary value is from a single source:

```numscript
send [COIN 100] (
  source = @world
  destination = @users:001
)
```

This draws 100 COIN from the `@world` account.

## Ordered Sources

Multiple accounts can be specified sequentially. The system withdraws from each account in
order until the target amount is reached:

```numscript
send [COIN 100] (
  source = {
    @users:001:wallet
    @payments:001
  }
  destination = @orders:001
)
```

If `@users:001:wallet` has only 30 COIN, the remaining 70 is drawn from `@payments:001`.

### Maximum Limits on Ordered Sources

Ordered sources can be capped to a maximum monetary amount, preventing them from being drawn
more than the specified amount:

```numscript
send [COIN 100] (
  source = {
    max [COIN 10] from @users:001:wallet
    @payments:001
  }
  destination = @orders:001
)
```

At most 10 COIN will be drawn from `@users:001:wallet`, with the rest coming from
`@payments:001`.

## Portioned Sources

Fractions or percentages distribute the amount across multiple source accounts. The summed
total of fractions in a block must equal 1.

### Fraction Notation

```numscript
send [COIN 100] (
  source = {
    10/100 from @platform:marketing
    remaining from @users:001:wallet
  }
  destination = @orders:001
)
```

### Percentage Notation

```numscript
send [COIN 100] (
  source = {
    10% from @platform:marketing
    remaining from @users:001:wallet
  }
  destination = @orders:001
)
```

The `remaining` keyword automatically completes the total to 100%.

## Nested Sources

Complex source specifications combine ordered and portioned approaches:

```numscript
send [COIN 100] (
  source = {
    50% from {
      max [COIN 10] from @users:001:wallet
      @users:001:chest
    }
    remaining from @payments:001
  }
  destination = @orders:001
)
```

## Cascading Sources with Maximum Cap

```numscript
send [USD/2 29900] (
  source = {
    10% from {
      max [USD/2 2000] from @coupons:FALL24
      @users:1234
    }
    remaining from @users:1234
  }
  destination = @payments:4567
)
```

This declares both ordering and constraints in a single atomic transaction: the coupon
provides 10% off capped at $20, with the customer paying the rest.

## Source with Overdraft

Sources can be configured to allow overdrafts (see [08-overdraft.md](08-overdraft.md)):

```numscript
send [USD/2 100] (
  source = @foo allowing unbounded overdraft
  destination = @bar
)
```

## `oneof` Sources

Select one source from a set based on runtime conditions:

```numscript
send [COIN 100] (
  source = oneof {
    @users:001:wallet
    @users:001:savings
  }
  destination = @orders:001
)
```
