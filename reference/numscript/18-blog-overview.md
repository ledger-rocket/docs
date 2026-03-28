# What is Numscript and Why is it Awesome?

> Source: <https://www.formance.com/blog/engineering/numscript>

## Overview

Numscript is Formance's domain-specific language (DSL) for programmable ledgers. It enables
teams to describe financial transactions in a declarative way that's both human-readable and
machine-safe.

## Core Concept

Rather than using imperative code to choreograph individual debits and credits, developers
express transaction intent: "Move X from A to B under these conditions." The ledger then
executes this intent atomically and immutably, ensuring every posting succeeds or fails as
one complete unit.

## Key Characteristics

- **Declarative approach**: Focuses on *what* should happen, not *how* to make it happen
- **Atomic execution**: All postings complete together or none at all
- **Immutable records**: Every transaction is permanently logged
- **Cross-team clarity**: Accessible to engineers, finance teams, and auditors alike
- **Open source**: Fully documented, available on GitHub with standalone interpreter

## Why Numscript Matters

### Safety First

Traditional systems orchestrate money movement through chained API calls, balance checks, and
conditional logic. Each additional integration introduces failure points: timeouts, duplicate
retries, balance inconsistencies. By consolidating complex transaction flows into single
declarative scripts, Numscript eliminates race conditions and partial posting issues that
plague conventional systems.

### Unified Language

"Move X from A to B under these conditions" becomes intelligible to developers, accountants,
and compliance officers without translation layers. Everyone references the same source of
truth, reducing miscommunication and accelerating reviews.

### Deterministic Rounding

Numscript uses integer arithmetic exclusively, combined with Unambiguous Monetary Notation.
This prevents floating-point errors when handling splits and fractional amounts -- a critical
requirement in financial systems.

## Comparison: Traditional vs. Numscript

| Aspect | Traditional | Numscript |
|--------|-------------|-----------|
| Transaction model | Procedural chains | Declarative, atomic |
| Failure handling | Manual rollbacks | Automatic all-or-nothing |
| Rounding errors | Floating-point issues | Deterministic integer math |
| Team communication | Multiple interpretations | Single source of truth |
| Audit trail | Scattered logs | Immutable ledger record |

## Use Cases

Numscript shines for apps that require complex money-moving code:

- **E-commerce**: Complex payment flows with multi-party settlements
- **Marketplaces**: Payment splitting between sellers, platform, and service providers
- **Ride-sharing**: Driver payouts with commission splits and charity donations
- **Company currencies**: Internal token/credit systems
- **Fintech platforms**: Multi-currency operations with compliance requirements

## Technical Foundation

- **Written in Go**: Portable, compiled language
- **Integer-only math**: Eliminates floating-point ambiguity
- **Open source**: Available at <https://github.com/formancehq/numscript>
- **Fully documented**: Complete specification and guides
- **Playground**: Interactive testing at <https://playground.numscript.org/>
- **WASM port**: Available at <https://github.com/PagoPlus/numscript-wasm>

## Getting Started

1. Try the [Numscript Playground](https://playground.numscript.org/)
2. Install the CLI: `go install github.com/formancehq/numscript/cmd/numscript@latest`
3. Read the [official docs](https://docs.formance.com/modules/numscript/introduction)
4. Explore [examples](17-examples.md)
