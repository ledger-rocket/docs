# System Architecture

> Source: https://www.twisp.com/docs/infrastructure/system-architecture

## Overview

"Twisp is a purpose-built system that provides a mission-critical, high performance, core ledger for constructing and running financial products."

## Integrating with Twisp

The platform streamlines financial data flows, structures ledger data, automates financial processes, and enables organizations to design seamless financial products.

## Your System Components

### Frontend

Customers interact with financial data through user interfaces. The platform enables direct GraphQL API access from frontend applications using fine-grained security policies and OIDC authentication.

### Infrastructure

The system backbone includes servers, streaming platforms, databases, and warehouses. Organizations can access financial data through Twisp data connectors supporting real-time streaming and batch access.

### Services

Self-contained software components provide specific functionality. Services integrate via GraphQL, gRPC, or REST interfaces after product logic is configured in the Accounting Core.

### Vendors

Third-party tools connect to the Accounting Core through plug-and-play integrations, simplifying adoption of existing technology investments.

## Interfaces

### API Options

- **GraphQL**: Clients request specific data needed, nothing more
- **gRPC**: High-performance framework using Protocol Buffers for distributed systems. Schemas available at https://buf.build/twisp/api
- **REST**: Scalable, stateless web services; includes OData for Salesforce integration

### Data Integration

**Real-Time:**

- Streaming to AWS Kinesis, Kafka, or webhooks
- Webhook-based HTTPS integration for instant notifications

**Batch:**

- SQL queries for reports and business intelligence tools
- Parquet format warehousing compatible with Redshift, Snowflake, BigQuery

### Vendor Integrations

Supported platforms include Fiserv, Lithic, Marqeta, and Stripe with webhook processing capabilities.

### Financial Protocols

- ISO 8583 (card networks)
- ISO 20022 (Swift and FedNOW)
- NACHA (ACH files)
- X9 (check clearance)
- BAI2 (financial institution files)

## Core Components

### Accounting Engine

Includes chart of accounts, transaction codes, and ledger entries for financial transaction processing.

### Ledger Database

Built on AWS DynamoDB with formally verified transaction layers providing "transactionally consistent, snapshot isolated, immutable storage" with complete historical records.

### Product Logic

Pre-built financial products include:

- **Brokerage** - trading lifecycle and clearinghouse reconciliation
- **Cards** - spend limits, wallets, organizational structures
- **Credit** - SCRA processes, interest calculations
- **Deposits** - FBO accounts, DDAs
- **Lending** - payments, amortization tables
- **Mortgages** - origination and servicing
- **Payments** - multi-currency, FX support
- **Wallets** - organizational spend management

### Security Policies

Fine-grained access control authorizes all operations. The system is credential-less, leveraging existing identity providers and cloud provider identities for authentication.
