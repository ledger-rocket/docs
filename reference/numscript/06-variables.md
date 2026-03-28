# Numscript Variables

> Source: <https://docs.formance.com/modules/numscript/reference/variables>

## Overview

Variables enable flexible and reusable Numscript scripts by allowing dynamic value injection
at execution time, rather than relying on hardcoded values.

## Variable Declaration Syntax

Variables are declared in a `vars` block with their type and name:

```numscript
vars {
  monetary $price
  account  $trade
  portion  $commission
  asset    $pair
  number   $id
  string   $reference
}
```

## Supported Variable Types

### `monetary`

Represents a positive integer amount tied to an asset.

- Format example: `"USD/2 100"`
- Can be initialized using `balance($account, $asset)` to pull current balances
- Balance values must be non-negative or execution fails

### `account`

Represents account identifiers.

- Must start with a letter or underscore
- Can contain letters, digits, underscores, and colons

### `asset`

Represents asset identifiers.

- Can include uppercase letters, digits, and forward slashes
- Examples: `USD/2`, `EUR/2`, `BTC/8`, `COIN`

### `portion`

Represents fractional values of monetary amounts.

- Percentage format: `15%`
- Fractional format: `15/100`
- Computed values must be between 0 and 1 (inclusive)

### `number`

Represents numeric values.

### `string`

Represents text values.

## Variable Naming Rules

Variable names must:

- Start with `$` followed by at least one letter or underscore
- Contain only lowercase letters, digits, and underscores
- Examples: `$price`, `$user_id`, `$commission_rate`

## Injection at Execution

Variables are injected via the `POST /{ledger}/transactions` endpoint using the `script.vars`
field:

```json
{
  "script": {
    "plain": "vars { monetary $price ... } send $price ( ... )",
    "vars": {
      "price": "USD/2 100",
      "trade": "trades:108391999",
      "commission": "15%",
      "pair": "EUR/2",
      "id": "108391999",
      "reference": "USD/EUR:108391999"
    }
  }
}
```

## Balance Initialization

The `balance()` function retrieves account balances for variable initialization:

```numscript
vars {
  monetary $available = balance(@users:001, USD/2)
}

send $available (
  source = @users:001
  destination = @payouts:001
)
```

This is useful for dynamic transaction logic based on current account states.

## Metadata Initialization

Variables can be initialized from account metadata using the `meta()` function:

```numscript
vars {
  account  $coupon
  account  $wallet
  monetary $value = meta($coupon, "coupon_value")
}

send $value (
  source = $coupon
  destination = $wallet
)
```

The metadata value stored under `"coupon_value"` must include `type` and `value` properties
to be properly parsed.

## Using Variables in Account Addresses

Variables can be interpolated into account addresses:

```numscript
vars {
  string $user_id
}

send [USD/2 100] (
  source = @world
  destination = @users:$user_id
)
```

## Using Variables in Send Statements

```numscript
vars {
  monetary $price
  account  $seller
  portion  $commission
}

send $price (
  source = @world
  destination = {
    $commission to @platform
    remaining to $seller
  }
)
```
