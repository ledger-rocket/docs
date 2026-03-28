# Twisp Introduction

> Source: https://www.twisp.com/docs/introduction

## What is Twisp?

Twisp is an accounting engine for building financial products. The platform provides a specialized financial ledger database with primitives designed for engineering systems that handle monetary transactions.

## Financial Ledgers Explained

Financial ledgers serve as foundational infrastructure for money-tracking systems. Examples:

- **Neobanks** manage debits and credits across numerous accounts with multiple financial instruments while generating consumer insights
- **Payments infrastructure** requires multi-tenant account servicing across multiple FBOs accessible via API

The platform addresses a critical challenge: "Developing, securing, operating, and scaling these financial ledgers on a general purpose database is a difficult and time consuming proposition."

## Core Features

Upon provisioning a Twisp ledger, users access:

1. **Transaction Ledger** - Implements double-entry accounting methodology
2. **Journals** - Tracks different currencies and time-period activities
3. **Accounts** - Provides chart-of-accounts structure for economic activity
4. **Transaction Codes** - Categorizes and tracks transaction types
5. **Layered Balances** - Maintains `available`, `settled`, and `encumbrance` balances with account hierarchies, dimensional tracking, and velocity monitoring
6. **Workflows** - Automates card authorization, ACH processing, and mobile check deposits
7. **High-Scale API** - Offers activity feeds, transaction enrichment, and reconciliation capabilities

---

## A Short History of Twisp

> Source: https://www.twisp.com/docs/introduction/history

### Traditional Database Challenges

Twisp identifies that monolithic relational databases -- the conventional choice for financial ledgers -- create operational difficulties. While distributed databases address scalability, their design patterns introduce complications for financial applications.

### Twisp's Solution

The company developed the Financial Ledger Database (FLDB) by merging "the operational and scaling characteristics of a distributed database with the correctness guarantees offered by relational databases."

### Core Features of FLDB

- Transactionally guaranteed operations
- Secure architecture
- Horizontal scalability
- Multi-tenant support

### Modern Cloud Integration

Twisp leverages contemporary infrastructure approaches -- serverless computing and infrastructure-as-code -- to create systems that are repeatable, autoscaling, fully managed, and usage-based.

Beyond the database layer, Twisp discovered demand for a complete accounting core. Many organizations have built custom ledger systems suffering from scale issues, unnecessary complexity, and flawed data models that complicate routine tasks like balance calculations.

---

## DIY Ledgers are a Problem

> Source: https://www.twisp.com/docs/introduction/no-diy-ledger

### Key Arguments

The Twisp team identified three critical lessons from years of fintech experience:

1. Designing and managing central accounting ledgers presents significant technical challenges
2. The apparent simplicity often misleads organizations into building proprietary solutions
3. Most DIY ledger implementations encounter substantial problems

### The Core Problem

Internal ledger systems accumulate technical debt like other long-lived software, but the consequences are especially severe. These systems are simultaneously:

- Critical business infrastructure
- Outside typical organizational expertise

This exemplifies "undifferentiated heavy lifting" -- work that drains resources without creating competitive advantage.

### Why Ledgers Should Be Outsourced

**Ledgers are fundamentally solved:** Double-entry accounting principles have existed for centuries, and database architectures supporting reliable, performant ledger systems have matured over decades. Rebuilding this infrastructure internally wastes valuable development capacity.

**The stakes are high:** Ledger failures directly impact user experience and can cause substantial financial damage due to their role in multiple business processes.

---

## Introducing Twisp (Blog Post)

> Source: https://www.twisp.com/blog/introducing-twisp

### The Problem with Ledgers

Organizations repeatedly rebuild ledger systems from scratch, wasting substantial engineering resources. "Ledger technology should be leveraged, not reinvented."

Homegrown ledger systems create several challenges:

- **Product-Ledger Entanglement**: Product features require constant modifications to core ledger code, blurring architectural boundaries and creating brittle applications
- **Developer Burden**: Engineers must understand accounting implications for every transaction, slowing product velocity
- **Organizational Scaling Issues**: Multiple internal ledgers create reconciliation nightmares, producing inconsistent accounting models across teams
- **Insufficient Alternatives**: Existing solutions either cost millions in licensing fees or bundle poorly-integrated ledger functionality

### Twisp's Core Principles

1. Operate like cloud infrastructure, not SaaS
2. Provide full autonomy and control of financial data
3. Be composable for any use case

### Key Capabilities

- Cloud-native architecture with security controls, horizontal scaling, and strong consistency guarantees
- Pre-built accounting templates for card processing, money movement, foreign exchange, and brokerage operations
- Financial workflow engine for fund flows, processor integrations, and reconciliation

### Pricing Model

Unlike competitors charging based on accounts or transaction volume, Twisp uses a usage-based model (reads, writes, storage) where inactive accounts incur minimal costs.
