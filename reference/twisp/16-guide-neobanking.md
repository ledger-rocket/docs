# Guide: Modeling Banking on Twisp

> Source: https://www.twisp.com/docs/guides/neobanking-guide

## Overview

This comprehensive guide demonstrates how to design and implement bank-like core accounting systems using Twisp.

## Context

Financial product companies navigate a maturation cycle: "Crawl" (leveraging BaaS providers), "Walk" (using specialized processors), and "Run" (building in-house capabilities). Twisp serves as a system of record for organizations building financial products.

## Chart of Accounts Structure

### End User Accounts

**Deposit Accounts** function as customer-accessible stores of value with debit instruments attached. The platform treats these as liabilities since funds belong to account holders.

**Credit Accounts** feature a line of credit paired with payable accounts. Customers accumulate charges and owe repayment by specified dates.

Both account types use:

- An Account Set representing total balance
- A default Account for posting customer transactions

#### Deposit Account Setup

The architecture models a hierarchical `Customer -> Account -> Card` relationship using Account Sets:

```graphql
mutation CreateEntityAccount(
  $accountSetId: UUID!
  $accountId: UUID!
  $code: String!
  $description: String
  $metadata: JSON
  $journalId: UUID! = "00000000-0000-0000-0000-000000000000"
  $accountSetName: String! = "Entity Account Set"
  $accountName: String! = "Entity Default Account"
) {
  createAccountSet(
    input: {
      accountSetId: $accountSetId
      name: $accountSetName
      description: $description
      metadata: $metadata
      journalId: $journalId
    }
  ) { accountSetId }
  createAccount(
    input: {
      accountId: $accountId
      name: $accountName
      description: $description
      code: $code
      accountSetIds: [$accountSetId]
      metadata: $metadata
    }
  ) { accountId }
}
```

#### Identifier Management

- Use the `externalId` field for uniqueness enforcement
- Populate `metadata` with external system identifiers
- Consider UUID v5 generation for deterministic Twisp entity IDs matching external counterparts

#### Credit Accounts

Credit accounts differ with a second account called "Credit Line" that establishes the spending limit.

| Entry ID | Account | Amount | Direction | Credit Balance | Payable Balance |
|----------|---------|--------|-----------|---------------|-----------------|
| 1 | Line | $1000 | Credit | $1000 | $0 |
| 2 | Default | $500 | Debit | $500 | $500 |
| 3 | Default | $250 | Credit | $750 | $250 |
| 4 | Default | $10 | Debit | $740 | $260 |

This provides two critical balances:

- **Credit Balance** - determines authorization eligibility
- **Credit Payable** - tracks customer debt

### Platform Accounts

| Account | Description |
|---------|-------------|
| Suspense | Posts transactions with unknown destinations |
| ACH Settlement | ACH transaction clearing |
| Bill Pay Settlement | Bill payment clearing |
| Foreign Checks Account | International check settlement |
| Courtesy Credit Account | Customer service credit entries |
| Charge Off Account | Uncollectable account write-offs |
| Fraud Losses | Fraud loss accounting |
| Levies & Garnishments | Levy and garnishment collections |
| Cashiers Check | Cashier's check settlement |
| Card Disputes | Card dispute reserves |
| ACH Disputes | ACH dispute reserves |
| Interchange Revenue | Shared interchange income |
| Collected Fee Revenue | Customer account fees |
| FBO | "For Benefit Of" omnibus accounts |
| Cash | Asset account for system funds |

## Transaction Workflows

### Transaction Primitives

- **Tran Codes** - Define how entries post for specific transaction types
- **postTransaction** - Posts transactions using designated tran codes
- **voidTransaction** - Reverses transaction entries
- **workflows** - Orchestrates multiple transaction operations

## Card Authorization and Settlement

### Tran Codes

| Code | Description |
|------|-------------|
| CARD_HOLD | Posts pending layer entries between settlement and card accounts |
| CARD_SETTLE | Posts settled layer entries |
| CARD_HOLD_VOID | Optional zero-value pending entries |
| CARD_HOLD_REPLACE | Identical to CARD_HOLD with different labels |
| CARD_DECLINE | Optional zero-value decline entries |

### Card Workflows

- **Authorization Approval**: Initial hold -> void initial -> post final authorization
- **Authorization Decline**: Initial hold -> void -> post decline record
- **Authorization Update**: Hold -> void -> new hold with updated amount
- **Authorization Cancelation**: Hold -> void -> post void record
- **Settlement**: Hold -> void -> post settlement at settled layer
- **Multi-settlement**: Partial settle with hold replacement
- **Settlement (no Auth)**: Force post directly to settled layer
- **Return**: Post settlement with reverse direction

### Check Balance Query

```graphql
query CheckBalance($accountId: UUID!) {
  balance(accountId: $accountId) {
    settled: available(layer: SETTLED) {
      normalBalance { formatted(as: { locale: "en-US" }) }
    }
    pending: available(layer: PENDING) {
      normalBalance { formatted(as: { locale: "en-US" }) }
    }
  }
}
```

## ACH Processing

Two operational modes:

- **RDFI** (Receiving Deposit Financial Institution) - External institutions credit/debit your accounts
- **ODFI** (Originating Deposit Financial Institution) - You initiate credits/debits with external accounts

### ACH Workflows

- **ODFI Push**: CREATE (pending debit) -> SUBMIT (settle) -> optional RETURN or CANCEL
- **ODFI Pull**: CREATE (encumbrance credit) -> SUBMIT (pending) -> SETTLE (settled) -> optional RETURN
- **RDFI Credit**: CREATE (encumbrance credit) -> SETTLE -> optional RETURN
- **RDFI Debit**: CREATE (encumbrance debit) -> SETTLE -> optional RETURN

## Clearing Settlements

The cash account represents total platform assets. Settlement, revenue, and operational accounts interact with cash through clearing operations. Reconcile settlement account balances regularly with actual bank representations.

## Advanced Topics

### Void And Post Workflow

Replaces manual void + post operations with single-identifier management using the workflow engine.

### Custom Balance Computations

Twisp computes balances across journal, account, and currency dimensions by default. Custom calculations enable per-MCC limits or daily/weekly/monthly spending thresholds.

```graphql
query GetEffectiveBalance($accountId: UUID!) {
  balance(
    accountId: $accountId
    currency: "USD"
    calculationId: "5867b5dd-fc69-416c-80f5-62e8a53610d5"
    dimension: { effectiveDate: "2020-01-01" }
  ) {
    available(layer: PENDING) {
      normalBalance { units }
    }
  }
}
```
