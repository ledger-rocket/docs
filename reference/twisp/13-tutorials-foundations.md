# Twisp Tutorials: Foundations

> Sources:
> - https://www.twisp.com/docs/tutorials/setting-up-accounts
> - https://www.twisp.com/docs/tutorials/posting-transactions
> - https://www.twisp.com/docs/tutorials/organizing-with-account-sets
> - https://www.twisp.com/docs/tutorials/indexes

---

## Tutorial: Setting Up Accounts

### Creating an Account

The `createAccount` mutation generates new accounts using these parameters:

- **accountId**: Required unique identifier
- **name**: Account designation
- **code**: Abbreviated shorthand
- **description**: Brief explanation
- **normalBalanceType**: Balance orientation (DEBIT or CREDIT)
- **status**: Active or closed state (defaults to `ACTIVE`)

"When an account is `LOCKED`, it cannot be updated unless you are also changing its status back to `ACTIVE`. Any attempt to write a ledger entry to a locked account will raise an error."

```graphql
mutation CreateAccount($accountCardSettlementId: UUID!) {
  createAccount(
    input: {
      accountId: $accountCardSettlementId
      name: "Card Settlement"
      code: "SETTLE.CARD"
      description: "Settlement account for card transactions."
      normalBalanceType: CREDIT
      status: ACTIVE
    }
  ) {
    accountId
    name
    code
    description
    normalBalanceType
  }
}
```

### Updating an Account

The `updateAccount` mutation modifies existing accounts. The response includes version history:

```graphql
mutation UpdateAccount($accountCardSettlementId: UUID!) {
  updateAccount(id: $accountCardSettlementId, input: { code: "CARD.SETTLE" }) {
    accountId
    code
    history(first: 2) {
      nodes {
        version
        code
      }
    }
  }
}
```

### Deleting (Locking) an Account

"Because Twisp is an immutable database, we cannot fully 'delete' an account. Deleting an account in Twisp instead marks its status as `LOCKED`."

```graphql
mutation DeleteAccount($accountCustomerBobbyId: UUID!) {
  deleteAccount(id: $accountCustomerBobbyId) {
    accountId
    status
  }
}
```

---

## Tutorial: Posting Transactions

### Selecting a Transaction Code

Transaction codes function as "pre-defined templates that specify how transactions should be processed." Example: `BOOK_TRANSFER` requires:

- Debit account specification
- Credit account specification
- Transfer amount
- Currency designation
- Effective transaction date

### Posting a Transaction

The `postTransaction` mutation requires:

- **transactionId**: Unique identifier for transaction
- **tranCode**: The transaction template to apply
- **params**: Key-value pairs matching the code's requirements

"It's important to provide a unique `transactionId` when posting a transaction. This can help prevent duplicate transactions from being posted accidentally."

### Querying Results

Posted transactions can be queried to retrieve transaction details, effective dates, codes used, journals affected, metadata, and ledger entries written.

---

## Tutorial: Organizing with Account Sets

### Creating an Account Set

Use the `createAccountSet` mutation with `accountSetId`, `journalId`, `name`, `description`, and `normalBalanceType` (DEBIT or CREDIT).

### Adding Members

The `addToAccountSet` mutation accepts an account set ID and a member object specifying either an `ACCOUNT` or `ACCOUNT_SET` member type.

### Nested Structures

"One powerful feature of AccountSets is that they can be nested within other sets," enabling hierarchical chart of accounts organization.

### Querying Members

The `accountSet` query retrieves set information and paginated member lists. GraphQL inline fragments distinguish between account and account set results.

### Updates and Deletion

`updateAccountSet` modifies existing set properties with version tracking. `deleteAccountSet` removes obsolete sets.

---

## Tutorial: Creating Custom Indexes

### Index Design Components

A custom index requires:

- **On and Name**: Specifies the record type (table) and assigns a unique identifier
- **Partition Key**: Groups data for querying. All queries must filter on partition keys. Example: `document.account_id`
- **Sort Key (Range)**: Defines ordering within partitions. Example: `string(document.metadata.category)` with ascending order
- **Constraints**: Optional CEL expressions filtering which records get indexed. Example: `has(document.metadata.category)`
- **Unique**: Optional boolean ensuring partition-sort key combination uniqueness

### Creating the Index

Use the `schema.createIndex` GraphQL mutation with name, target table, partition configuration, sort configuration, and optional constraints.

### Querying the Index

Queries must specify `index: { name: CUSTOM }` and use `where.custom` arguments. "You must provide an equality filter for all partition key aliases."

### Additional Capabilities

Twisp supports `createHistoricalIndex` for version tracking and `createSearchIndex` for full-text search functionality.
