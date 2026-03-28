# Platform Comparison: LedgerRocket vs Formance vs Twisp

This document provides a structured comparison of three programmable ledger platforms:
**LedgerRocket**, **Formance**, and **Twisp**. Each takes a distinct architectural
approach to solving the same core problem -- providing developers with a reliable,
auditable financial ledger. This comparison is intended to inform product positioning,
documentation priorities, and feature roadmap decisions.

## High-Level Comparison

| Category | LedgerRocket | Formance | Twisp |
|---|---|---|---|
| **Architecture** | Stateless microservices (Go), hexagonal, REST, TigerBeetle | Monolithic Go service, REST, PostgreSQL | Cloud-native managed service, GraphQL/gRPC/REST, custom FLDB on DynamoDB |
| **Multi-Tenancy** | site_id isolation, sites as first-class resources | Ledger-level isolation, buckets for grouping | Tenant-level with OIDC/IAM auth |
| **Reference Data** | Rich dimensional model (Entity, Ledger, Account, AccountCode, Currency, Country) | Flat model, auto-created accounts, metadata KV pairs | Structured chart of accounts, AccountSets, Journals |
| **Transaction DSL** | JSON templates with CEL expressions | Numscript (purpose-built language) | GraphQL tran codes with CEL |
| **Accounting Model** | Double-entry, integer minor units, TigerBeetle transfers | Source/destination, unsigned volumes, conservation constraint | Double-entry, three layers (settled/pending/encumbrance) |
| **Governance** | Maker-checker workflow, audit slugs, content hashing | None built-in | None built-in for tran codes; workflows for card/ACH |
| **Balances** | Dedicated Balance Service, real-time + historical + chain analysis | Derived from volumes, single API | Layered balances, hierarchy rollups, velocity monitoring |
| **Storage** | TigerBeetle + PostgreSQL + S3 | PostgreSQL | Custom FLDB on DynamoDB (formally verified) |
| **API Style** | REST (OpenAPI 3.1), RFC 9457 errors, x-api-key | REST (OpenAPI), simple JSON errors | GraphQL (primary), gRPC, REST (OData), OIDC/IAM |
| **Developer Experience** | Dry-run, explain mode, enriched transfers, template validation | Numscript playground, VS Code extension, CLI, spec testing | Pre-built financial products, vendor integrations, protocol support |

## Detailed Comparison

### 1. Architecture

#### LedgerRocket

- Stateless microservices written in Go, following hexagonal (ports and adapters) architecture.
- Separate services for ledger management, event processing, balance queries, and template management.
- REST APIs defined with OpenAPI 3.1 specifications.
- TigerBeetle as the purpose-built financial database for transfers and balances.
- PostgreSQL for reference data (entities, ledgers, accounts, account codes).
- S3 for template storage.

#### Formance

- Monolithic Go service (Formance Ledger / formerly NumScript Ledger).
- Single REST API surface backed by PostgreSQL.
- Simpler deployment model -- one service, one database.
- Part of a broader Formance Stack (Payments, Wallets, Webhooks, etc.) but the ledger is a single service.

#### Twisp

- Cloud-native managed service -- customers do not deploy or operate infrastructure.
- Custom Financial Ledger Database (FLDB) built on AWS DynamoDB with formal verification.
- Multiple API styles: GraphQL as the primary interface, gRPC via Protocol Buffers (published to buf.build), and REST with OData conventions for Salesforce integration.
- Designed as a platform with pre-built financial product templates layered on top.

### 2. Multi-Tenancy

#### LedgerRocket

- `site_id` provides isolation across all services.
- Sites are first-class API resources -- created, configured, and managed via the API.
- All queries are scoped to a site, preventing cross-tenant data leakage at the data model level.

#### Formance

- Isolation at the ledger level -- each tenant can have its own ledger.
- Buckets group multiple ledgers together for organizational purposes.
- No first-class "tenant" or "site" concept in the ledger API itself.

#### Twisp

- Tenant-level isolation with OIDC/IAM authentication.
- Admin namespace for tenant CRUD operations.
- Multi-tenancy is a managed-service concern -- Twisp handles the infrastructure isolation.

### 3. Reference Data Model

#### LedgerRocket

- Rich dimensional model designed for complex financial reporting needs:
  - **Entity**: Represents a beneficial owner or custodian. Entities own ledgers.
  - **Ledger**: Scoped to an entity and a currency. All transfers occur within a single ledger.
  - **Account**: Dimensionally classified by EntityScope, CounterpartyType, FunctionalDomain, and ValuationRole. Accounts are immutable once created.
  - **AccountCode**: Carries IFRS/FSLI classification attributes for regulatory reporting.
  - **Linked accounts**: Nostro/vostro pairs for inter-entity relationships.
  - **Currency**: ISO currency codes with configurable scale (decimal places).
  - **Country**: ISO country codes for jurisdictional classification.
  - **Rollup accounts**: Aggregation accounts for reporting across account hierarchies.

#### Formance

- Flat, convention-based model:
  - Accounts are auto-created on first use in a transaction.
  - Naming follows colon-delimited conventions (e.g., `payments:001`, `users:alice:main`).
  - Metadata as arbitrary key-value pairs on accounts and transactions.
  - No formal entity hierarchy, ledger-currency scoping, or account classification system.
  - `@world` is a special account representing external capital.

#### Twisp

- Structured but less dimensional than LedgerRocket:
  - **Chart of accounts** with formal account types.
  - **AccountSets** for hierarchical grouping and rollup.
  - **Journals** scoped to a currency and time period.
  - No entity/custodian concept or dimensional classification (EntityScope, FunctionalDomain, etc.).

### 4. Transaction Definition / DSL

#### LedgerRocket

- Templates are JSON documents with embedded CEL (Common Expression Language) expressions.
- Variables are evaluated sequentially and are integer-only (no floating-point in financial calculations).
- Validations are CEL predicates that must all pass before a transaction executes.
- Legs define individual transfers (debit account, credit account, amount).
- Metadata schemas use JSON Schema 2020-12 for structured, validated metadata on events.
- Accounting treatments reference external policy documents for regulatory compliance.

#### Formance

- **Numscript** is a purpose-built DSL with its own grammar (defined in ANTLR):
  - `send` and `save` statements for moving money.
  - Source/destination model with allocation percentages and kept amounts.
  - Built-in overdraft control (`allowing overdraft up to`, `allowing unbounded overdraft`).
  - Rounding rules for handling remainder cents.
  - Variables interpolated from transaction metadata.
  - Full toolchain: CLI (`check`, `run`, `test`), VS Code extension, web playground.

#### Twisp

- Transaction codes (tran codes) defined via GraphQL mutations.
- CEL expressions for dynamic value computation (similar to LedgerRocket).
- Entry-level granularity -- each entry specifies DEBIT or CREDIT explicitly.
- Three-layer model: entries can target settled, pending, or encumbrance layers.
- Correlation IDs link related transactions (e.g., authorization and settlement).

### 5. Accounting Model

#### LedgerRocket

- Traditional double-entry bookkeeping.
- Every transfer debits one account and credits another, always on the same ledger.
- Amounts stored as integer minor units (no floating-point).
- Transfers grouped by `event_id` for atomic multi-leg transactions.
- Rollup accounts aggregate balances for reporting purposes.

#### Formance

- Source/destination model rather than traditional debit/credit terminology.
- Balance = Inputs - Outputs (volumes tracked as unsigned integers).
- `@world` account serves as the external capital source (infinite liquidity).
- Conservation constraint: the sum of all balances across a ledger must equal zero.
- No negative balances by default (overdraft must be explicitly allowed in Numscript).

#### Twisp

- Traditional double-entry with three balance layers:
  - **Settled**: Completed, final transactions.
  - **Pending**: Authorized but not yet settled (e.g., card holds).
  - **Encumbrance**: Reserved amounts (e.g., budget commitments).
- Available balance is the sum of configured layers.
- Velocity controls enforce spend limits over time windows.
- Entry sequences within transactions provide ordering guarantees.

### 6. Transaction Lifecycle / Governance

#### LedgerRocket

- Full maker-checker workflow for templates:
  - States: DRAFT, PENDING_APPROVAL, LIVE, DEPRECATED.
  - Only LIVE templates can process events.
- Audit slugs provide a complete trail:
  - Maker (who created), checker (who approved), revision number, content hash, diff from previous version.
- Risk assessment metadata can be attached to templates.
- This governance model is a significant differentiator for regulated financial institutions.

#### Formance

- No built-in approval workflow.
- Templates (Numscript scripts) are stored and executed directly.
- Governance would need to be implemented externally (e.g., via CI/CD pipelines or a wrapper service).

#### Twisp

- No built-in approval workflow for tran codes.
- Workflows exist for automating specific financial processes (card authorization, ACH, deposits).
- These workflows are operational automation, not governance/approval mechanisms.

### 7. Balance Queries

#### LedgerRocket

- Dedicated Balance Service as a separate microservice.
- Real-time balances served from TigerBeetle with high throughput.
- Historical timeseries queries for balance-over-time analysis.
- Chain analysis: per-event-chain breakdown of balance changes (via Athena).
- Account code aggregation for reporting across account hierarchies.
- Immutable balance caching for point-in-time snapshots.

#### Formance

- Balances derived from volumes stored on the ledger.
- `GetBalances` API endpoint returns current balances.
- No separate balance service, historical timeseries, or chain analysis capabilities.

#### Twisp

- Layered balances reflecting settled, pending, and encumbrance states.
- Account hierarchies (via AccountSets) roll up balances automatically.
- Velocity monitoring tracks balance changes over configurable time windows.
- No dedicated balance service -- balances are computed from the core ledger.

### 8. Storage Layer

#### LedgerRocket

- **TigerBeetle**: Purpose-built financial database optimized for high-throughput transfers and balance queries. Provides strict serializability and deterministic execution.
- **PostgreSQL**: Stores reference data (entities, ledgers, accounts, account codes, currencies, countries) via the Ledger Service.
- **S3**: Template storage for versioned template documents.

#### Formance

- **PostgreSQL**: Single database for all data (transactions, accounts, metadata, balances).
- Simpler operational model but potentially limited at extreme scale for transfer-heavy workloads.

#### Twisp

- **Custom FLDB on AWS DynamoDB**: Purpose-built Financial Ledger Database with formal verification (TLA+ or similar).
- Managed service -- customers have no direct access to or responsibility for the storage layer.
- Designed for cloud-native scalability on AWS infrastructure.

### 9. API Style

#### LedgerRocket

- REST APIs defined with OpenAPI 3.1 specifications.
- RFC 9457 Problem Details for structured error responses.
- `x-api-key` header authentication.
- Consistent resource-oriented design across all services.

#### Formance

- REST APIs with OpenAPI specifications.
- Simple JSON error responses (no structured error standard).
- Part of a broader API surface (Formance Stack) with consistent patterns.

#### Twisp

- **GraphQL** as the primary API -- rich query capabilities, schema introspection.
- **gRPC** via Protocol Buffers published to buf.build for high-performance integrations.
- **REST with OData** conventions for Salesforce and similar enterprise integrations.
- OIDC/IAM authentication for enterprise identity management.

### 10. Developer Experience

#### LedgerRocket

- **Dry-run validation** (`events:validate`): Test event processing without committing transfers.
- **Execution trace** (`events:validate:explain`): Step-by-step trace showing how templates evaluate, which validations pass/fail, and what transfers would be created.
- **Enriched transfers**: API responses include resolved account names, account codes, and entity information alongside raw IDs.
- **Template validation endpoint**: Validate template syntax and CEL expressions before saving.

#### Formance

- **Numscript playground**: Web-based editor for writing and testing Numscript.
- **VS Code extension**: Syntax highlighting, autocomplete, and inline validation for Numscript.
- **CLI toolchain**: `check` (validate), `run` (execute), `test` (spec-based testing).
- **Spec-based testing**: Define expected outcomes and run automated tests against Numscript scripts.

#### Twisp

- **Pre-built financial products**: Ready-to-use templates for cards, deposits, lending, trust/custody, crypto, and more.
- **Vendor integrations**: Out-of-the-box connectors for Fiserv, Lithic, Marqeta, Stripe, Plaid, and others.
- **Financial protocol support**: ISO 8583 (card networks), ISO 20022 (SWIFT/payments), NACHA (ACH), BAI2 (bank statements).
- Significantly higher starting point for teams building common financial products but less flexibility for custom models.

## Summary: Strengths and Trade-Offs

### LedgerRocket

**Strengths:**

- Richest reference data model of the three -- dimensional account classification with IFRS/FSLI attributes supports complex regulatory reporting without external enrichment.
- Built-in maker-checker governance with full audit trail (slugs, content hashes, diffs) addresses compliance requirements that the other platforms leave to external tooling.
- TigerBeetle as the transfer engine provides purpose-built financial database performance with strict serializability.
- Dedicated Balance Service with historical timeseries and chain analysis goes beyond simple "current balance" queries.
- Dry-run and explain modes provide developer confidence without risking production data.

**Trade-offs:**

- More services to deploy and operate compared to Formance's single-service model.
- JSON+CEL templates are less expressive than Numscript for complex allocation/rounding logic but more accessible to developers already familiar with JSON and expression languages.
- No pre-built financial product templates or vendor integrations (unlike Twisp).

### Formance

**Strengths:**

- Numscript is the most expressive transaction DSL -- allocation percentages, rounding rules, overdraft control, and the save statement handle complex money movement patterns natively.
- Developer toolchain (playground, VS Code extension, CLI with spec testing) lowers the barrier to entry.
- Single-service deployment is operationally simple.
- Open source with active community.

**Trade-offs:**

- Flat account model with convention-based naming provides no structural enforcement -- account classification and hierarchy must be managed externally.
- No built-in governance or approval workflow -- regulated institutions must build this themselves.
- PostgreSQL-only storage may limit throughput for very high-volume transfer workloads.
- Source/destination model is unfamiliar to accountants trained in debit/credit terminology.

### Twisp

**Strengths:**

- Three-layer balance model (settled/pending/encumbrance) natively supports card authorization, ACH, and other two-phase financial flows without workarounds.
- Pre-built financial products and vendor integrations dramatically reduce time-to-market for common use cases.
- Financial protocol support (ISO 8583, ISO 20022, NACHA, BAI2) handles real-world message formats.
- Managed service eliminates operational burden.
- Formally verified storage engine (FLDB) provides mathematical guarantees of correctness.

**Trade-offs:**

- Managed service only -- no self-hosted option for organizations with data residency or sovereignty requirements.
- GraphQL-primary API adds complexity for teams without GraphQL experience.
- Less flexible for highly custom accounting models -- the pre-built product approach optimizes for common patterns.
- No maker-checker governance or audit trail for tran code changes.
- Vendor lock-in to AWS infrastructure via DynamoDB-based FLDB.
