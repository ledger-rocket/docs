# Variables

Variables compute intermediate values from event data. They are evaluated sequentially before validations and legs, and their results are available to all subsequent expressions.

## Variable Declaration

Variables are declared in the `variables` array. Each variable has three fields:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | Yes | Identifier used to reference this variable in later variables, validations, and legs. Max 100 characters, `lower_snake_case`. |
| `value` | string | Yes | CEL expression that computes the variable's value, **or** a static literal value. Max 1000 characters. |
| `description` | string | Yes | Human-readable explanation of what this variable represents. Max 500 characters. |

## Sequential Evaluation Order

Variables are evaluated **in declaration order**. A variable at index `N` can reference any variable declared at indices `0` through `N-1`, but not variables declared after it.

```json
"variables": [
    {
        "name": "fee_amount",
        "value": "event.amount * 3 / 100",
        "description": "3% processing fee."
    },
    {
        "name": "net_amount",
        "value": "event.amount - fee_amount",
        "description": "Amount after fee deduction. References fee_amount above."
    }
]
```

In this example, `net_amount` can reference `fee_amount` because it is declared after it. Reversing the order would cause a compilation error.

## Static Variables

Variables can hold static values -- fixed account IDs or constants -- that do not depend on the incoming event. This is useful for template-defined accounts that are always the same regardless of the event:

```json
"variables": [
    {
        "name": "suspense_account",
        "value": "50050000-0000-4001-a000-000000000004",
        "description": "Static: Suspense (10103) on house ledger."
    },
    {
        "name": "cash_bal_suspense",
        "value": "50050000-0000-4002-a000-000000011903",
        "description": "Static: Cash Balances - Suspense (11903) on allocation ledger."
    }
]
```

This example is from **template 5215** (`accountants_reconciliation_bank_payment`). The suspense accounts are always the same for this site, so they are hardcoded as static variables rather than passed on each event.

## Computed Variables

Variables can contain CEL expressions that compute values from event fields, metadata, or previously-declared variables:

```json
"variables": [
    {
        "name": "fee_bps",
        "value": "250",
        "description": "Fee rate in basis points (2.50%)."
    },
    {
        "name": "fee_amount",
        "value": "event.amount * fee_bps / 10000",
        "description": "Fee computed from event amount at the configured basis point rate."
    },
    {
        "name": "net_amount",
        "value": "event.amount - fee_amount",
        "description": "Remainder after fee deduction."
    }
]
```

## Type Constraints

Variable values are typed based on the CEL expression result:

- **Integer expressions** produce `int` values. Use these for amounts, counts, and basis points.
- **String expressions** produce `string` values. Use these for account IDs, references, and labels.
- **Boolean expressions** produce `bool` values. Use these for conditional flags.

Leg fields reference variables by name. The type must match what the leg field expects:
- `amount` fields expect `int` values
- `debit_account` and `credit_account` fields expect `string` values (account UUIDs)
- `condition` fields expect `bool` values

## Real-World Examples

### Template 5215: Static Account Variables

```json
{
    "template_id": 5215,
    "name": "accountants_reconciliation_bank_payment",
    "variables": [
        {
            "name": "suspense_account",
            "value": "50050000-0000-4001-a000-000000000004",
            "description": "Static: Suspense (10103) on house ledger."
        },
        {
            "name": "cash_bal_suspense",
            "value": "50050000-0000-4002-a000-000000011903",
            "description": "Static: Cash Balances - Suspense (11903) on allocation ledger."
        }
    ]
}
```

These static variables define accounts that never change per-event. The legs then reference these variables by name (`accounts.suspense_account.account_id` and `accounts.cash_bal_suspense.account_id`) to build transfers.

### Common Patterns

**Percentage fee calculation (basis points):**

```json
{
    "name": "platform_fee",
    "value": "event.amount * 300 / 10000",
    "description": "3% platform fee in minor units."
}
```

**Remainder after deduction:**

```json
{
    "name": "seller_payout",
    "value": "event.amount - platform_fee",
    "description": "Amount paid to seller after platform fee."
}
```

**Boolean flag from metadata:**

```json
{
    "name": "is_expedited",
    "value": "event.extra_metadata.priority == \"expedited\"",
    "description": "True when the event requests expedited processing."
}
```

**Conditional amount:**

```json
{
    "name": "expedite_fee",
    "value": "is_expedited ? 1500 : 0",
    "description": "Flat $15.00 expedite fee when priority is expedited, zero otherwise."
}
```
