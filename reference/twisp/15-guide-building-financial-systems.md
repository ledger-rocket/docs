# Guide: Building Financial Systems with the Twisp Accounting Core

> Source: https://www.twisp.com/docs/guides/building-financial-systems

## Overview

This guide presents a structured methodology for constructing financial systems using Twisp. It addresses the complete lifecycle, from service definition through scalability planning.

## Defining Financial Services

### Identifying Services

Begin by specifying the financial products your system will offer -- such as checking accounts, savings accounts, loans, and credit cards.

### Determining Requirements

Establish the specific requirements for each service, including account types, transaction categories, and associated fees.

**Example neobank services:**

- **Checking Accounts**: Support deposits, withdrawals, and customer-to-customer transfers; requires accounts for customer holdings and internal company tracking
- **Savings Accounts**: Like checking accounts, plus interest calculations and withdrawal restrictions or penalties
- **Loans**: Necessitate tracking principal balances, interest rates, repayment schedules, and disbursement/repayment accounts

## Designing Accounts and Transactions

### Creating a Chart of Accounts

Steps:

1. Identify required account types (assets, liabilities, income, expenses)
2. Create accounts for specific purposes (customer wallets, settlement accounts, revenue accounts)
3. Organize accounts using AccountSet types for hierarchical structures and balance roll-ups

### Defining Transaction Codes

Steps:

1. Determine transaction type (ACH transfer, card purchase, foreign exchange)
2. Identify involved accounts and money flow directions
3. Establish ledger entry counts and debit/credit assignments
4. Specify whether entries reflect settled or pending amounts

### Establishing Posting Rules

Requirements:

1. Create frameworks for ledger recording using double-entry accounting principles
2. Align transaction codes with accounting standards (two or more balanced entries per transaction)
3. Implement automatic posting based on transaction codes

## Security and Compliance

### Industry Standards

"Twisp's Financial Ledger Database (FLDB) is designed to provide strong guarantees about the lineage of data, ensuring the integrity, transparency, and accuracy of your financial data."

Features include double-entry principle enforcement, immutable append-only ledgers, and proper transaction contextualization.

### Access Controls

Twisp provides centralized authorization policies and explicit security resources, minimizing unauthorized access risks through unified authentication and authorization management.

### Audit Trails

Built-in timing and sequencing mechanisms enable complete transaction and activity history tracking, supporting accurate auditing, reconciliation, and regulatory compliance monitoring.

## Integrations with Twisp

### Payment Processors and Banking APIs

Using the GraphQL API:

1. Identify target payment processor or banking API and gather credentials
2. Develop custom integration layer for system communication
3. Utilize Twisp GraphQL API to post transactions, query balances, and manage ledger data

### CRMs and ERPs

1. Identify target CRM or ERP system and collect credentials
2. Build custom integration layer for data synchronization
3. Use Twisp GraphQL API to manage financial data while ensuring consistency between systems

## Testing and Validation

### Testing with Sample Data

Populate the ledger with realistic sample data covering various use cases -- customer accounts and transactions like deposits, withdrawals, transfers, and fees.

### Testing Use Cases

Validate system handling of edge cases: large transactions, insufficient funds, high volumes, and invalid data.

### Validating Calculations

Verify that balances accurately reflect account structure intent by comparing system results against manually calculated or known values.

### Assessing Performance and Scalability

Simulate substantial customer account influxes and high transaction volumes over short periods. Monitor response times, processing speeds, and system performance.
