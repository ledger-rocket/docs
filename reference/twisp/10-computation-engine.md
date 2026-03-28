# Computation Engine

> Source: https://www.twisp.com/docs/infrastructure/computation-engine

## Overview

The computation infrastructure supports protocol workflows and an extensive financial calculation runtime via CEL (Common Expression Language).

Protocol workflows enable modeling of sophisticated state-machine-like computations on ledger data and calculations, with capability to communicate with external systems.

The workflow engine allows modeling sophisticated financial protocols and their interaction with ledgers, including ISO 8583, ACH, and external webhook processing.

## Workflows as State Machines

A state machine is an abstraction for designing algorithms and protocols. It reads inputs and transitions between states based on those inputs.

Twisp provides protocol workflows as a mechanism to define state machines. By combining immutable ledgers, calculation runtime, and protocol workflows, Twisp processes complex financial protocols and translates them into ledger entries with full historical lineage and integrity guarantees.

Protocol workflows can be triggered by:

- Events from ledger operations (create, read, update, delete)
- API calls to Twisp (including external webhooks)

Workflows execute within the same atomic transaction that triggered them, providing strong transactional and data integrity guarantees. Workflow output and calculation engine state persist to ledgers, enabling chains of transactionally guaranteed computation with full lineage. Since each transaction has dedicated compute and memory, noisy neighbor problems are minimal.

## Calculation Runtime

The calculation runtime is Twisp's primary mechanism for populating ledgers and deriving information from ledger data. It uses expression-based language and runtime processing schema-based input to produce schema-based output.

It operates with persisted ledger data while maintaining strong integrity guarantees through a powerful type system, strong type checking, and predictable performance.

Calculation expressions leverage CEL combined with a financial type system and function library, providing expressive mechanisms to derive and populate ledger data with strict typing.

The calculation engine supports:

- Calculating balances, fees, and interest across dimensions
- Extracting, aggregating, and bucketing ledger data
- Real-time analytics for business intelligence
- User-defined functionality via expressive syntax

## Function Library

Twisp includes a financial type system and extensive function library, surfaced within the calculation runtime as types and functions operating over those types.

The library includes sophisticated types such as full-featured money and currency types, with functions for complex calculations including interest, fees, rounding, and balances. It also provides parsers for well-known external formats like ISO8583 and ACH.
