# Numscript Introduction

> Source: <https://docs.formance.com/modules/numscript/introduction>

## Overview

Numscript is a Domain-Specific Language (DSL) designed to help you model complex financial
transactions, replacing complex and error-prone custom code with easy-to-read, declarative
scripts.

Numscript is the DSL used to express financial transactions within the Formance ledger.

## Financial Transaction Definition

Formance defines a financial transaction as a series of discrete value movements between
abstract accounts. Each movement represents a transfer of value from one account to another,
with an associated amount and asset denomination.

- **Assets** can represent any kind of value: traditional currencies like USD or JPY, custom
  tokens, or commodities.
- **Accounts** can represent anything: a bank account, a voucher, a virtual wallet, or an
  order that has yet to be paid out.

## Design Principles

### Readability

The intent of a Numscript program should always be clear and easy to understand. Numscript
programs should be readable by both developers and non-technical financial users, providing
a shared, executable definition of money movements.

### Correctness

Monetary computations in Numscript should always yield correct results, avoiding common
currency rounding errors and accidental money creation or destruction. Execution is atomic,
ensuring that either all modeled transactions are committed or none.

Numscript uses only integer math with built-in rounding rules combined with Unambiguous
Monetary Notation to ensure deterministic rounding without error.

### Finiteness

Numscript programs should always terminate in a predictable and consistent way. Programs are
deterministic, always terminating with a predictable output. This ensures that the behavior
of Numscript programs can be reliably predicted and controlled.

## Example: Marketplace Payment Split

```numscript
// Capture payment of $5.99
send [USD/2 599] (
  source = @world
  destination = @payments:001
)

// Move payment to ride clearing account
send [USD/2 599] (
  source = @payments:001
  destination = @rides:0234
)

// Split $5.99 among driver, charity, and platform
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

This script produces five postings distributing 599 USD/2 across accounts:

- 510 to `@drivers:042`
- 9 to `@charity`
- 80 to `@platform:fees`

## Key Characteristics

- **Declarative**: Focuses on *what* should happen, not *how* to make it happen
- **Atomic execution**: All postings complete together or none at all
- **Immutable records**: Every transaction is permanently logged
- **Integer-only math**: Eliminates floating-point ambiguity
- **Open source**: Available at <https://github.com/formancehq/numscript>
- **Portable**: Written in Go, with a standalone interpreter and WASM port

## Tooling

- **CLI**: `numscript check`, `numscript run`, `numscript test`
- **VS Code extension**: Available on the Marketplace
- **Online Playground**: <https://playground.numscript.org/>
- **Language Server**: Diagnostics, hover info, go-to-definition, document symbols
