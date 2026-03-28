# Numscript Destinations

> Source: <https://docs.formance.com/modules/numscript/reference/destinations>

## Overview

Destinations define where funds are sent in financial transactions, with several options
available for routing monetary transfers across accounts.

## Single Destination

The simplest approach transfers all funds to one account:

```numscript
send [COIN 100] (
  source = @world
  destination = @users:001
)
```

## Allocation Destinations

Funds can be split onto multiple accounts using fractional or percentage-based distributions.
The sum must equal 1, with the `remaining` keyword handling the balance.

### Fractional Syntax

```numscript
send [COIN 100] (
  source = @world
  destination = {
    90/100 to @users:001
    remaining to @fees
  }
)
```

### Percentage Notation

```numscript
send [COIN 100] (
  source = @world
  destination = {
    90% to @users:001
    remaining to @fees
  }
)
```

## Kept Destinations

Specify that some portion should be kept (remain in the source account):

```numscript
destination = kept
```

Or with partial transfers:

```numscript
send [COIN 100] (
  source = @world
  destination = {
    50% to @users:001
    remaining kept
  }
)
```

## Ordered Destinations with Maximum Caps

Route funds to multiple destinations in sequence, with maximum caps for each destination:

```numscript
send [COIN 100] (
  source = @world
  destination = {
    max [COIN 20] to @users:001
    max [COIN 50] to @users:002
    remaining to @users:003
  }
)
```

## Nested Destinations

Destination blocks support nesting for complex distribution hierarchies:

```numscript
send [COIN 100] (
  source = @world
  destination = {
    80% to @users:001
    20% to {
      70% to @platform
      15% to @taxes
      remaining to @charity
    }
  }
)
```

## `oneof` Destinations

Select one destination from a set:

```numscript
send [COIN 100] (
  source = @world
  destination = oneof {
    max [COIN 50] to @users:001
    remaining to @users:002
  }
)
```

## Complex Split Example

```numscript
send [USD/2 599] (
  source = @rides:0234
  destination = {
    85% to @drivers:042
    remaining to {
      10% to @charity
      remaining to @platform:fees
    }
  }
)
```

Result: 510 to drivers, 9 to charity, 80 to platform fees.
