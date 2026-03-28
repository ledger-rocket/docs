# Numscript Rounding

> Source: <https://docs.formance.com/modules/numscript/reference/rounding>

## Overview

Numscript lacks support for floating-point or decimal numbers. The system ensures that non-
integer values resulting from monetary computations are balanced by distributing remainders
across accounts deterministically.

## Core Rounding Mechanism

Numscript works by **flooring any computed amount** and subsequently **spreading the remaining
amount as fairly as possible starting from top to bottom**.

### Basic Example

Allocating `COIN 99` equally between two accounts at 50% each:

```numscript
send [COIN 99] (
  source = @world
  destination = {
    50% to @rider
    50% to @taxes
  }
)
```

- `@rider` receives `COIN 50`
- `@taxes` receives `COIN 49`

Reversing the destination order reverses who gets the extra unit.

### Five-Way Split

Splitting 99 units among five equal recipients (19.8 each):

```numscript
send [COIN 99] (
  source = @world
  destination = {
    1/5 to @a
    1/5 to @b
    1/5 to @c
    1/5 to @d
    1/5 to @e
  }
)
```

Result:

- `@a` through `@d` each receive 20 units
- `@e` receives 19 units

The 4 remainder units are distributed one per account from top to bottom.

## Penny Handling with Cryptocurrency

```numscript
// Split BTC 0.00000943 evenly between two accounts
send [BTC/8 943] (
  source = @users:1234
  destination = {
    50% to @users:4567
    50% to @users:7890
  }
)
```

Numscript floors to the nearest whole unit (BTC 0.00000471 each) and deterministically
allocates the remainder (1 unit) to the first destination: `@users:4567` gets 472,
`@users:7890` gets 471.

## Fixed Fees and Percentage Allocations

When combining fixed amounts with percentages, ordering significantly impacts outcomes due to
multi-pass resolution.

### The Challenge

A transaction with fixed fees and percentages may produce unexpected results:

```numscript
send [AUD/2 1999] (
  source = @world
  destination = {
    7/1999 to @payment_provider
    0.6% to @payment_provider
    0.5% to @franchise_fee
    remaining to @store
  }
)
```

`@payment_provider` receives 8 cents instead of the expected 7 cents.

### Multi-Pass Allocation Process

**First Pass**: Whole amounts are allocated:

- Fixed amounts and percentages are floored
- Remainders are held aside

**Second Pass**: Fragments are distributed top-to-bottom:

- Remaining cents are allocated sequentially from first to last destination
- This causes `@payment_provider` to gain an additional cent

### Recommended Solution

Place percentage allocations **before** fixed amounts to ensure fixed fees remain exact:

```numscript
send [AUD/2 1999] (
  source = @world
  destination = {
    0.6% to @payment_provider
    0.5% to @franchise_fee
    7/1999 to @payment_provider
    remaining to @store
  }
)
```

## Key Takeaways

1. Numscript uses **integer-only math** -- no floating point
2. Computed amounts are **floored** (rounded down)
3. Remainders are distributed **top to bottom** among destinations
4. Order of destinations **matters** for rounding behavior
5. Place **percentage allocations before fixed amounts** for predictable results
6. No money is ever created or destroyed in the rounding process
