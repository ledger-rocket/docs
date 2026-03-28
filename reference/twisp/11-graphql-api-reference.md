# GraphQL API Reference

> Sources:
> - https://www.twisp.com/docs/reference/graphql/mutations
> - https://www.twisp.com/docs/reference/graphql/types

## Overview

Twisp provides a comprehensive GraphQL API for interacting with the accounting core. Mutations modify or create data on the ledger or execute admin actions. The API is accessible at `https://api.us-east-1.cloud.twisp.com/financial/v1/graphql`.

gRPC schemas are also available at https://buf.build/twisp/api.

---

## Mutations Reference

### ACH Operations (`ach` namespace)

- **createConfiguration**: Create a configuration for processing an ACH file. Defines settlement, exception, and suspense accounts.
- **generateFile**: Generate ACH files (currently RDFI return files)
- **processFile**: Process an ACH file at the file key with the defined configuration
- **updateConfiguration**: Modify existing ACH configurations

### Admin Management (`admin` namespace)

- **createGroup**, **deleteGroup**, **updateGroup**: Group lifecycle management
- **createTenant**, **deleteTenant**, **updateTenant**: Tenant operations
- **createUser**, **deleteUser**, **updateUser**: User management
- **restore**: Restore a tenant/region into a new tenant/region

### Account Operations

- **createAccount**: Create a new account
- **updateAccount**: Update fields on an existing account (restricted subset of fields for data integrity)
- **deleteAccount**: Moves the account state to `LOCKED`
- **createAccountSet**: Group multiple accounts together
- **updateAccountSet** / **deleteAccountSet**: Manage account groupings
- **addToAccountSet**: Add a member (account or account set) to an account set

### Transaction and Journal Management

- **postTransaction**: Write a transaction to the ledger using the predefined defaults from the `tranCode` provided
- **updateTransaction**: Modify transaction metadata and details
- **voidTransaction**: Void existing transactions
- **createJournal**, **updateJournal**, **deleteJournal**: Journal lifecycle
- **createTranCode**, **updateTranCode**, **deleteTranCode**: Transaction code definitions

### Velocity Controls

- **createVelocityControl**, **updateVelocityControl**, **deleteVelocityControl**: Control lifecycle
- **createVelocityLimit**, **updateVelocityLimit**, **deleteVelocityLimit**: Limit definitions
- **attachVelocityControl**, **detachVelocityControl**: Link controls to accounts
- **addLimitToControl**, **removeLimitFromControl**: Manage limit associations

### Schema and Indexing (`schema` namespace)

- **createIndex**: Create a custom index for querying records
- **createHistoricalIndex**: Create a custom index for querying all versions of records' histories
- **createSearchIndex**: Full-text search indexes powered by OpenSearch
- **createView**: Create a view for a set of source tables with automatic updates
- **deleteIndex**, **deleteView**: Remove indexes and views

### Authentication (`auth` namespace)

- **createClient**: Create a new security client
- **updateClient**: Update an existing client by replacing policies
- **deleteClient**: Remove client access

### Events and Webhooks (`events` namespace)

- **createEndpoint**: Create a new endpoint for webhooks
- **updateEndpoint**: Modify webhook configurations
- **deleteEndpoint**: Remove endpoints

### Workflow Execution (`workflow` namespace)

- **execute**: Execute a workflow identified by `workflowId`
- **executeTask**: Execute specific workflow tasks

### Warehouse Operations (`warehouse` namespace)

- **executeStatement**, **executeStatementSync**: Run queries asynchronously or synchronously
- **batchExecuteStatement**: Execute multiple statements
- **cancelStatement**: Stop running queries
- **export**: Export warehouse data

### Additional Operations

- **evaluate**: Run CEL (Common Expression Language) expressions
- **attachCalculation** / **detachCalculation**: Balance customization
- **createSchedule**: Schedule automated tasks (scheduler namespace)
- **createDownload** / **createUpload**: File operations

---

## Types Reference

### Core Financial Entities

**Account**: Represents individual ledger accounts that track economic activity.

- Unique identifiers (accountId, externalId)
- Balance tracking across SETTLED, PENDING, and ENCUMBRANCE layers
- Normal balance type (debit or credit)
- Associated entries and account sets
- Config options (e.g., `enableConcurrentPosting`)

**Entry**: Represents one side of a transaction.

- Account reference and amount
- Direction (DEBIT or CREDIT)
- Entry type categorization
- Layer designation
- Transaction association

**Transaction**: Records accounting events with double-entry enforcement.

- Multiple entries (minimum 2)
- Correlation ID for grouping related transactions
- Effective date and journal assignment
- Transaction code reference

**Journal**: Organizes transactions into separate books.

- Status management (ACTIVE/LOCKED)
- Optional code identifier

### Structural Types

**Balance**: Auto-calculated sums showing debit and credit amounts, normal balance calculations, multi-currency support, and historical versions.

**AccountSet**: Groups accounts hierarchically with member accounts and nested sets, aggregate balance calculations, and journal-specific organization.

**TranCode**: Defines transaction patterns with parameter definitions, entry templates using CEL expressions, and transaction specifications.

### Data Types

- **Money**: Multi-currency amounts with formatting support (units, currency code, formatted representation)
- **Timestamp**: Date and time values
- **UUID**: Unique identifiers
- **Decimal**: Fixed-precision numeric values
- **DebitOrCredit**: Enum for balance direction (DEBIT, CREDIT)
- **Status**: Operational states (ACTIVE, LOCKED)
- **Layer**: Entry classification (SETTLED, PENDING, ENCUMBRANCE)

### Authentication and Access Control

- **Client**: Service principals with associated policies
- **User**: Human users within organizations, belonging to multiple groups
- **Group**: Collections of users with combined policy permissions
- **Organization**: Top-level entity containing tenants, groups, and users
- **Tenant**: Environment containers with isolated ledgers

### Connection Types (Pagination)

The schema implements cursor-based pagination through connection objects:

- `AccountConnection`, `EntryConnection`, `TransactionConnection`
- Each provides `nodes`, `edges` with cursors, and `pageInfo`

### Filtering and Indexing

**Indexes** enable optimized queries with pre-defined indexes (ACCOUNT_ID, NAME, etc.), custom indexes with CEL expressions, and unique/historical index variants.

**FilterValue** supports conditional operators: equality (`eq`), comparison (`lt`, `lte`, `gt`, `gte`), and pattern matching (`like`).

### Policy and Authorization

**Policy** defines permissions with effect (ALLOW/DENY), actions and resources, and CEL-based assertions.
