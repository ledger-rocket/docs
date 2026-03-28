# LedgerRocket Documentation Design

A comprehensive plan for the documentation LedgerRocket should publish, informed by
what Formance and Twisp publish and tailored to our microservices architecture.

## Documentation Structure

### 1. Getting Started

#### 1.1 Platform Overview
**What**: What LedgerRocket is, who it's for, architectural philosophy.
**Why**: First thing prospects and new developers read.
**Status**: Partial -- `docs/shared/platform.md` covers architecture and domain concepts. Needs a public-facing rewrite.

#### 1.2 Quickstart Guide
**What**: End-to-end walkthrough: create a site, entity, ledger, accounts, define a template, submit an event, query balances. Working curl/SDK examples.
**Why**: Formance has "Your First Transaction" tutorial; Twisp has "Twisp 101". We need an equivalent.
**Status**: Needs to be written from scratch.

#### 1.3 Authentication
**What**: API key setup, header format (`x-api-key`), key rotation, environment-specific keys.
**Why**: First practical blocker for any integration.
**Status**: Documented in OpenAPI spec descriptions. Needs standalone doc.

#### 1.4 Architecture Overview
**What**: Service map, data flow, technology choices (TigerBeetle, PostgreSQL, S3), hexagonal architecture pattern.
**Why**: Engineers need to understand the system before building on it.
**Status**: `docs/shared/platform.md` covers this well. Needs public-facing polish.

### 2. Core Concepts

#### 2.1 Multi-Tenancy (Sites)
**What**: What a site is, how isolation works, creating and managing sites.
**Why**: Fundamental to every API call. Twisp has tenant docs; Formance has buckets/ledgers.
**Status**: Covered in platform.md. Needs expansion with examples.

#### 2.2 Entities
**What**: Beneficial owners vs custodians, entity types (individual, company), KYC status, nostro/vostro patterns.
**Why**: Unique to LedgerRocket -- competitors don't model entity ownership this richly.
**Status**: Brief coverage in platform.md. Needs dedicated doc with examples.

#### 2.3 Ledgers
**What**: Entity+currency scoping, why all transfers must be same-ledger, ledger creation patterns.
**Why**: Critical architectural constraint. Formance has looser ledger model; Twisp uses journals.
**Status**: Brief coverage in platform.md. Needs expansion.

#### 2.4 Accounts
**What**: Dimensional model (EntityScope, CounterpartyType, FunctionalDomain, ValuationRole), linked accounts, rollup accounts, immutability.
**Why**: Our richest differentiator vs competitors. Formance auto-creates accounts with no structure; Twisp uses AccountSets.
**Status**: Partially documented. Needs comprehensive guide with dimensional model examples.

#### 2.5 Chart of Accounts (Account Codes)
**What**: Account types (ASSET/LIABILITY/EQUITY/REVENUE/EXPENSE/SUPPLEMENTAL), IFRS/FSLI attributes, rollup flags.
**Why**: Regulatory reporting capability. Neither competitor offers IFRS-level classification.
**Status**: Exists in API spec. Needs conceptual guide.

#### 2.6 Events
**What**: Event structure, event_id (UUIDv7), accounts map (purpose -> account_id), template_ids, amount, extra_metadata. Idempotency. Event lifecycle.
**Why**: The primary API interaction for posting financial transactions.
**Status**: Covered in API spec and platform.md. Needs standalone tutorial.

#### 2.7 Templates
**What**: JSON+CEL template structure, variables, validations, legs, metadata schemas, accounting treatments.
**Why**: Core of our transaction definition model. Comparable to Numscript and tran codes.
**Status**: Covered in platform.md. Needs comprehensive authoring guide (see section 3).

#### 2.8 Transfers
**What**: Double-entry transfers, same-ledger constraint, enriched transfers (with account names/codes), transfer queries.
**Why**: The output of event processing -- what gets written to TigerBeetle.
**Status**: Covered in API spec. Needs conceptual doc explaining the model.

#### 2.9 Balances
**What**: Real-time balances, historical timeseries, chain analysis, account code aggregation, credits_posted/debits_posted model.
**Why**: Core read path for any financial product.
**Status**: Balance Service API spec exists. Needs conceptual guide.

### 3. Template Authoring Guide

#### 3.1 Template Structure
**What**: JSON schema breakdown -- top-level fields, how variables/validations/legs relate.
**Why**: This is our equivalent of Formance's Numscript reference. Must be equally comprehensive.
**Status**: Needs to be written. Can reference existing test fixtures for examples.

#### 3.2 CEL Expressions
**What**: Available CEL functions, integer-only arithmetic, accessing event fields (amount, accounts, extra_metadata), type system.
**Why**: The expression language is the core authoring skill.
**Status**: Needs to be written.

#### 3.3 Variables
**What**: Sequential evaluation, referencing prior variables, built-in selectors, type constraints.
**Why**: Variables are how templates compute derived values.
**Status**: Needs to be written.

#### 3.4 Validations
**What**: CEL predicates, error messages, common validation patterns (amount > 0, required metadata fields).
**Why**: Prevents bad events from creating transfers.
**Status**: Needs to be written.

#### 3.5 Legs (Transfer Definitions)
**What**: Debit/credit account selection, amount expressions, allowed selectors, no-expression constraint in legs.
**Why**: The actual double-entry output definition.
**Status**: Needs to be written.

#### 3.6 Metadata Schemas
**What**: JSON Schema 2020-12 for extra_metadata, how validation works, schema examples.
**Why**: Ensures event payloads conform to expected structure.
**Status**: Needs to be written.

#### 3.7 Accounting Treatments
**What**: Treatment types, policy references, how treatments map to regulatory requirements.
**Why**: Differentiator -- neither competitor has this.
**Status**: Needs to be written.

#### 3.8 Template Lifecycle
**What**: Maker-checker workflow (DRAFT -> PENDING_APPROVAL -> LIVE -> DEPRECATED), audit slugs, revision tracking, content hashing, rollback.
**Why**: Governance capability unique to LedgerRocket.
**Status**: Covered in ARE API spec. Needs narrative guide.

#### 3.9 Testing Templates
**What**: Dry-run validation (`events:validate`), execution trace (`events:validate:explain`), what the trace output shows.
**Why**: Critical for template development iteration.
**Status**: Covered in Event Service API spec. Needs tutorial.

### 4. API Reference

#### 4.1 Ledger Service API
**What**: Auto-generated from OpenAPI 3.1 spec. Interactive docs (Scalar).
**Status**: OpenAPI spec exists at `/v2/ledger/docs/`. Needs hosted documentation portal.

#### 4.2 Event Service API
**What**: Auto-generated from OpenAPI 3.1 spec.
**Status**: OpenAPI spec exists at `/v2/event-service/docs/`. Needs hosted documentation portal.

#### 4.3 Balance Service API
**What**: Auto-generated from OpenAPI 3.1 spec.
**Status**: OpenAPI spec exists at `/v2/balances/docs/`. Needs hosted documentation portal.

#### 4.4 Accounting Rule Engine API
**What**: Auto-generated from OpenAPI 3.1 spec.
**Status**: OpenAPI spec exists at `/v2/accounting-rule-engine/docs/`. Needs hosted documentation portal.

#### 4.5 Common Patterns
**What**: Pagination, filtering (equals/not-equals/in/not-in operators), search, error handling (RFC 9457), authentication.
**Why**: Cross-cutting concerns that apply to all services.
**Status**: Needs to be written. Patterns exist in API specs but aren't documented as a guide.

### 5. Guides (Use-Case Driven)

#### 5.1 Setting Up a Multi-Entity Ledger Structure
**What**: Step-by-step: create site, entities (company + customers), ledgers (per entity+currency), accounts with dimensional classification.
**Why**: The foundational setup every customer needs. Comparable to Twisp's "Foundations" tutorial.
**Status**: Needs to be written.

#### 5.2 Modeling Payment Flows
**What**: Card authorization/settlement, ACH credit/debit, wire transfers. Template examples for each.
**Why**: Most common use case. Twisp has extensive neobanking guide; Formance has marketplace examples.
**Status**: Needs to be written. Can draw from existing template test fixtures.

#### 5.3 Revenue Recognition
**What**: Accounting treatments for different revenue recognition patterns, template examples with policy references.
**Why**: Unique differentiator -- regulatory compliance built into templates.
**Status**: Needs to be written.

#### 5.4 Building a Neobank on LedgerRocket
**What**: End-to-end guide: account structure, deposit/withdrawal flows, card transactions, balance queries, reporting.
**Why**: Comparable to Twisp's "Guide: Neobanking" -- shows the full platform capability.
**Status**: Needs to be written.

#### 5.5 FBO Account Management
**What**: For-Benefit-Of patterns, custodian vs beneficial owner, nostro/vostro accounts.
**Why**: Critical for payment platforms. Twisp has an FBO guide; Formance doesn't address this.
**Status**: Needs to be written.

#### 5.6 Rollup Accounts and Reporting
**What**: How rollup accounts aggregate, manual_rollup_override, reporting patterns.
**Why**: Unique to LedgerRocket.
**Status**: Needs to be written.

#### 5.7 Event Chain Analysis
**What**: How to trace a financial event through its chain of transfers, using Balance Service chain endpoints.
**Why**: Powerful debugging and audit capability.
**Status**: Needs to be written.

### 6. Operations

#### 6.1 Deployment Architecture
**What**: Service topology, ECS configuration, ALB routing, environment-specific configs.
**Status**: Exists in deployment configs. Needs public-facing doc.

#### 6.2 Health Checks and Monitoring
**What**: /health, /livez, /readyz endpoints, Prometheus metrics, what to alert on.
**Status**: Endpoints documented in API specs. Needs ops guide.

#### 6.3 Cache Management
**What**: Platform cache, customer cache, template cache refresh, cache stats.
**Status**: Partially documented. Needs ops guide.

#### 6.4 Outbox Pattern
**What**: Event outbox lifecycle (prepared -> committed -> published), retry behavior, admin endpoints for monitoring.
**Status**: Covered in Event Service API. Needs conceptual doc.

## Priority Matrix

| Priority | Section | Reason |
|----------|---------|--------|
| **P0** | Quickstart Guide | First thing new users need |
| **P0** | Template Authoring Guide (3.1-3.5) | Core authoring workflow |
| **P0** | API Reference (hosted portal) | Already have specs, just need hosting |
| **P1** | Core Concepts (2.1-2.9) | Foundational understanding |
| **P1** | Modeling Payment Flows | Most common use case |
| **P1** | Common API Patterns | Cross-cutting developer needs |
| **P2** | Template Lifecycle (3.8) | Governance differentiator |
| **P2** | Building a Neobank | Comprehensive showcase |
| **P2** | FBO Account Management | Key enterprise use case |
| **P3** | Revenue Recognition | Advanced regulatory feature |
| **P3** | Operations docs | Internal/ops team focus |
| **P3** | Event Chain Analysis | Advanced debugging |

## Competitor Documentation Gaps We Should Fill

| Area | Formance Has | Twisp Has | We Should Have |
|------|-------------|-----------|---------------|
| Interactive playground | Numscript Playground | No | Template builder/tester UI |
| IDE integration | VS Code extension | No | VS Code extension for template JSON+CEL |
| CLI tools | numscript CLI (check/run/test) | No | Template CLI tools |
| Pre-built templates | Example Numscript scripts | Pre-built financial products | Template library (card auth, ACH, wire, etc.) |
| SDK/client libraries | TypeScript, Python, Go | Go SDK, gRPC stubs | TypeScript, Python, Go (already have generation scripts) |
| Schema registry | Ledger Schema | GraphQL schema introspection | JSON Schema catalog for templates and metadata |
