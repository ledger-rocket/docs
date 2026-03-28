# The Twisp Financial Ledger Database

> Source: https://www.twisp.com/docs/infrastructure/ledger-database

## Overview

Twisp developed a Financial Ledger Database (FLDB) by merging "the operational and scaling characteristics of a distributed database with the correctness guarantees offered by relational databases." The system is transactionally guaranteed, secure, and highly scalable -- specifically engineered for financial applications requiring multi-tenant workloads. Twisp employs infrastructure-as-code methodology for resource creation and access management.

## Comparison to Other Databases

The FLDB enforces strict user-defined schemas, indexes, and referential integrity like traditional relational databases. Unlike purely tabular systems, it supports nested sub-documents and collections (lists, JSON) while maintaining schema requirements.

The platform combines "the flexibility of document databases like MongoDB while maintaining the schematic integrity and operational model of a table-driven relational database like PostgreSQL."

## Database Features

### Append-Only Immutability

Financial data lineage is critical. The system uses immutability as its core mechanism -- all new data is recorded while previous data remains unchanged. This creates an append-only log with cryptographic protections, ensuring data is tamper-proof.

Every record includes a `history` node exposing the chain of state changes.

### Strong Data Integrity

A transactional MVCC write-ahead log (WAL) tracks all system data with cryptographically linked blocks. The system uses digest hash values and Merkle audit proofs to "cryptographically verify the integrity of any data in the system."

### Index-First Design

All data reads must occur through indexes, eliminating table scans and promoting predictable access patterns. Index features include:

- **Strongly consistent** -- Index operations guarantee data integrity
- **Strong partition and sorting controls** -- Compound keys and multi-field sorting supported
- **Compatible with all fields** -- Any schema element can create index keys
- **Filterable with CEL expressions** -- Partial indexes for conditional records
- **Unlimited definitions** -- Ledgers support unlimited indexes
- **Online, zero-downtime migrations** -- Indexes re-partition without downtime

### Index-Based Relations

Ledgers support standard relationship types via properly constructed indexes:

- One-to-one
- One-to-many
- Many-to-many

The system enforces full referential integrity across ledgers, preventing parent record deletion without removing child records first.

### Referential Integrity

Strict schemas enforce consistent data structure. Foreign key (FK) constraints maintain relationships between data elements, preventing changes that would violate relationships.

### Type-Safe Schemas

All records conform to declarative schemas with typed fields. Supported types:

| Type | Description |
|------|-------------|
| `int` | 64-bit signed integer |
| `uint` | 64-bit unsigned integer |
| `double` | 64-bit IEEE floating-point |
| `bool` | Boolean value |
| `string` | Unicode strings |
| `timestamp` | ISO-8601 with nanosecond precision |
| `bytes` | Base64-encoded sequences |
| `uuid` | RFC4122 identifier |
| `json` | Arbitrary JSON documents |
| `[<type>]` | Ordered scalar lists |

Sub-documents support nested structures while maintaining type safety and can be used in indexes.

### Zero-Downtime Migrations

Schema changes execute online without downtime. New elements enter `DELETE_ONLY` or `WRITE_ONLY` states during migration. Supported operations include:

- Add/Drop elements
- Add/Drop indexes
- Add/Drop schema elements
- Add/Drop references
- Add/Drop joins
- Add calculations (replayed based on position)

### Transaction Layer

The system provides MVCC interactive transactions with strictly serializable isolation levels for maximum consistency.
