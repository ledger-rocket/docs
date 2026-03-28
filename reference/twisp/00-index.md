# Twisp Documentation Index

Twisp is an accounting engine for building financial products. It provides a specialized financial ledger database with primitives designed for engineering systems that handle monetary transactions.

- **Website**: https://www.twisp.com
- **Documentation**: https://www.twisp.com/docs
- **gRPC Schemas**: https://buf.build/twisp/api
- **GitHub**: https://github.com/twisp
- **Go SDK**: https://github.com/twisp/twisp-sdk-go
- **Auth Go**: https://github.com/twisp/auth-go

## Files in this Directory

### Introduction and Overview

- [01-introduction.md](01-introduction.md) - What is Twisp, history, why not DIY ledgers, blog post

### Accounting Core

- [02-ledgers.md](02-ledgers.md) - Ledger architecture: journals, accounts, entries, transactions, balances
- [03-encoded-transactions.md](03-encoded-transactions.md) - Transaction codes (tran codes), composable double-entry accounting, correlations
- [04-chart-of-accounts.md](04-chart-of-accounts.md) - Accounts, account sets, credit/debit normal types
- [05-balances.md](05-balances.md) - Balance calculations, DR/CR, write-time computation
- [06-layered-accounting.md](06-layered-accounting.md) - Three-layer model (Settled, Pending, Encumbrance), available balance

### Infrastructure

- [07-system-architecture.md](07-system-architecture.md) - APIs (GraphQL, gRPC, REST), data integration, vendor integrations, financial protocols
- [08-security-and-auth.md](08-security-and-auth.md) - OIDC authentication, policies, tenants, clients, encryption
- [09-ledger-database.md](09-ledger-database.md) - Financial Ledger Database (FLDB), immutability, indexes, schemas, migrations
- [10-computation-engine.md](10-computation-engine.md) - Workflows as state machines, CEL calculation runtime, function library

### API Reference

- [11-graphql-api-reference.md](11-graphql-api-reference.md) - All GraphQL mutations and types

### Tutorials

- [12-tutorial-101.md](12-tutorial-101.md) - Twisp 101: Zuzu neobank example (accounts, tran codes, transactions)
- [13-tutorials-foundations.md](13-tutorials-foundations.md) - Setting up accounts, posting transactions, account sets, custom indexes
- [14-tutorial-admin-management.md](14-tutorial-admin-management.md) - Managing tenants, users, and groups
- [20-tutorial-ach-setup.md](20-tutorial-ach-setup.md) - Setting up ACH processing

### Guides

- [15-guide-building-financial-systems.md](15-guide-building-financial-systems.md) - End-to-end methodology for building on Twisp
- [16-guide-neobanking.md](16-guide-neobanking.md) - Modeling banking: deposits, credit, cards, ACH, clearing
- [17-guide-fbo-clearing.md](17-guide-fbo-clearing.md) - FBO clearing and payment facilitation
- [18-guide-position-accounting.md](18-guide-position-accounting.md) - Securities trading and position-based accounting

### Processors

- [19-processor-ach.md](19-processor-ach.md) - ACH processor architecture, workflows, file operations, webhook decisioning
