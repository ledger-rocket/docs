# Twisp 101 Tutorial

> Sources:
> - https://www.twisp.com/docs/guides/tutorial
> - https://www.twisp.com/docs/tutorials/twisp-101/step-3

## Welcome to the Twisp 101 Tutorial

### Context

This tutorial introduces foundational features of the Twisp Accounting Core. The example uses "Zuzu," an imaginary neobank serving underbanked US customers with checking, savings, and loan products.

Key learning areas include:

- Setting up a chart of accounts
- Defining transaction codes
- Writing ledger entries by posting transactions

The typical implementation process follows four stages:

1. **Design** - Create structural elements for the accounting system
2. **Test** - Verify the design meets requirements
3. **Deploy** - Launch products using Twisp
4. **Monitor** - Track performance, usage, auditing, and reporting

### The Project: Zuzu Neobank

Zuzu offers personal banking services including checking, savings, and loans, partnering with multiple banks while issuing debit cards and providing banking applications.

**Project requirements:**

- Chart of accounts for internal and customer accounts (checking, savings, loans)
- Double-entry ledger for financial activity (deposits, withdrawals, transfers, fees)

### Step 0: Establish API Connection

Twisp is API-first. Connection begins through the Twisp Console, which provides a GraphiQL interface for GraphQL API interaction.

Test your connection with this introspection query:

```graphql
query {
  __schema {
    types {
      name
      kind
      description
    }
  }
}
```

---

## Step 3: Model Intra-Bank Transfers

### Revenue Account Creation

Create a revenue account to track transfer fees using a `CREDIT` normal balance type:

- Account name: "Revenues"
- Account code: "REV"
- Description: "Company revenues (e.g. fees)"

### Bank Transfer TranCode

The `BANK_TRANSFER` tran code handles four distinct journal entries:

1. **Transfer Debit** - Moves funds from sender's checking account
2. **Transfer Credit** - Moves funds to receiver's checking account
3. **Fee Debit** - Charges fee against sender's account
4. **Fee Credit** - Records fee revenue

#### Required Parameters

- `fromAccount` (UUID) - Sender's account ID
- `toAccount` (UUID) - Receiver's account ID
- `amount` (DECIMAL) - Transfer amount
- `fee` (DECIMAL) - Fee percentage as decimal (e.g., 0.01 for 1%)
- `effective` (DATE) - Transaction effective date

#### Fee Calculation

The system uses the expression:

```cel
decimal.Round(decimal.Mul(params.amount, params.fee), 'half_up', 2)
```

### Test Transaction Example

A $2.25 transfer from Ernie to Bert with a 2% fee ($0.05) demonstrates the system posting four balanced entries across three accounts.
