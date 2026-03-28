# Numscript Metadata

> Source: <https://docs.formance.com/modules/numscript/reference/metadata>

## Overview

Numscript transactions can interact with metadata, both on transactions and accounts.
Structured account metadata can be injected in Numscript variables during initialization.

## Reading Account Metadata

Metadata values can be injected into Numscript variables using the `meta()` function. This
retrieves structured data from account metadata keys.

### Type Structure

Metadata values require two properties: `type` and `value`.

### Available Types

| Type | Description | Example Value |
|------|-------------|---------------|
| `number` | Numeric values | `42` |
| `string` | Text values | `"hello"` |
| `asset` | Currency/asset identifiers | `"USD/2"` |
| `monetary` | Amount + asset objects | `{"amount": 100, "asset": "USD/2"}` |
| `account` | Account references | `"platform:merchant"` |
| `portion` | Percentage representations | `"15.5%"` |

### Example: Reading Metadata

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

The metadata stored on the coupon account under key `"coupon_value"` might look like:

```json
{
  "coupon_value": {
    "type": "monetary",
    "value": {
      "amount": 500,
      "asset": "USD/2"
    }
  }
}
```

## Writing Account Metadata

Metadata can be written to an account using the `set_account_meta()` statement:

```text
set_account_meta(@account, "key", value)
```

The statement takes a string-type key and a value which can be of any type, either as a
variable or a literal.

### Examples

```numscript
set_account_meta(@users:001, "status", "active")
set_account_meta(@users:001, "tier", 3)
set_account_meta($seller, "last_sale", [USD/2 100])
```

## Writing Transaction Metadata

Metadata can be written to the current transaction using the `set_tx_meta()` statement:

```text
set_tx_meta("key", value)
```

The statement takes a string-type key and a value which can be of any type, either as a
variable or a literal.

### Examples

```numscript
set_tx_meta("order_fee", [USD/2 100])
set_tx_meta("tax", 20/100)
set_tx_meta("collection_account", @platform:commission)
set_tx_meta("commission", $commission)
set_tx_meta("reference", "order-12345")
```

## Metadata Update Behavior

- Metadata updates are **additive and idempotent**
- New keys are added
- Existing keys update to new values
- Duplicate operations have no effect
- Metadata updates never remove existing keys (use empty strings or sentinel values instead)

## Practical Application

Metadata enables complex transaction logic. For example, per-merchant commission rates:

```numscript
vars {
  account  $merchant
  portion  $commission = meta($merchant, "commission_rate")
  monetary $amount
}

send $amount (
  source = @orders:current
  destination = {
    $commission to @platform:fees
    remaining to $merchant
  }
)

set_tx_meta("merchant", $merchant)
set_tx_meta("commission_rate", $commission)
```
