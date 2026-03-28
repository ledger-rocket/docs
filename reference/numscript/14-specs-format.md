# Numscript Specs Format

> Source: <https://docs.formance.com/modules/numscript/specs-format>

## Overview

The Numscript specs format provides a standardized JSON-based approach for writing unit tests
that validate Numscript execution, assertions, and expected outcomes given specific inputs.

## Schema

A JSON schema is available for editor integration and autocomplete support:

```text
https://raw.githubusercontent.com/formancehq/numscript/refs/heads/main/specs.schema.json
```

## File Convention

For a Numscript file `my-script.num`, the specs file is `my-script.num.specs.json`.

## Core Structure

```json
{
  "$schema": "https://raw.githubusercontent.com/formancehq/numscript/refs/heads/main/specs.schema.json",
  "balances": {},
  "variables": {},
  "metadata": {},
  "featureFlags": [],
  "testCases": []
}
```

### Root-Level Fields

| Field | Description |
|-------|-------------|
| `balances` | Initial account balances by asset |
| `variables` | Named variable definitions |
| `metadata` | Account metadata key-value pairs |
| `featureFlags` | Array of feature flag identifiers |
| `testCases` | Array of test case objects |

## Preconditions

Preconditions define the input state for test execution. They can be specified at the root
level or within individual test cases. Inner-level definitions override root-level values
while preserving unmodified properties.

### Variables Format

```json
{
  "variables": {
    "amount": "USD/2 100"
  }
}
```

### Balances Format

```json
{
  "balances": {
    "alice": { "USD/2": 200 },
    "bob": { "USD/2": -42 }
  }
}
```

### Metadata Format

```json
{
  "metadata": {
    "alice": {
      "id": "1234"
    }
  }
}
```

## Test Cases

Each test case has an `it` description field and a set of assertions:

```json
{
  "testCases": [
    {
      "it": "should transfer 100 USD from alice to bob",
      "expect.postings": [
        {
          "source": "alice",
          "destination": "bob",
          "asset": "USD/2",
          "amount": 100
        }
      ]
    }
  ]
}
```

## Assertions

Only explicitly defined assertions execute.

### Error Assertions

**`expect.error.missingFunds`**: Confirms script failure due to insufficient funds (default:
false).

**`expect.error.negativeAmount`**: Confirms script failure from negative send amounts
(default: false).

### Posting Assertions

**`expect.postings`**: Validates exact posting operations emitted:

```json
{
  "expect.postings": [
    {
      "source": "world",
      "destination": "user:001",
      "asset": "EUR/2",
      "amount": 100
    }
  ]
}
```

### Metadata Assertions

**`expect.txMetadata`**: Validates transaction metadata set during execution:

```json
{
  "expect.txMetadata": {
    "senderAccount": "user:5829"
  }
}
```

**`expect.metadata`**: Validates account metadata post-execution:

```json
{
  "expect.metadata": {
    "alice": {
      "id": "1234"
    }
  }
}
```

### Balance Assertions

**`expect.endBalances`**: Validates complete final balances across all accounts and assets.

**`expect.endBalances.includes`**: Validates a subset of final balances, ignoring other
accounts or currencies.

### Movement Assertions

**`expect.movements`**: Validates flows between accounts by mapping
source -> destination -> asset -> amount:

```json
{
  "expect.movements": {
    "alice": {
      "bob": { "EUR/2": 100 }
    }
  }
}
```

## Test Control Modifiers

### Focus Mode

```json
{
  "focus": true,
  "it": "only run this test"
}
```

When applied to any test, only focused tests execute. Produces error status codes to prevent
accidental commits.

### Skip Modifier

```json
{
  "skip": true,
  "it": "skip this test"
}
```

Excludes a test from execution entirely.

## Running Tests

```bash
numscript test src/domain/numscript
```
