# Guide: Position-Based Accounting for Securities Trading

> Source: https://www.twisp.com/docs/guides/position-accounting

## Overview

This guide explains how to implement position-based accounting for securities trading using Twisp. It covers designing a chart of accounts, defining transaction codes, posting trades, and monitoring activities.

## Design the Chart of Accounts

### 1. Create Securities Trading Accounts for Customers

Establish separate accounts for each customer's trading activities. Since accounts support multiple currencies, treat each security as its own currency unit. A robust structure uses account sets per customer:

- `CLIENT_*.CASH` (asset, debit-normal)
- `CLIENT_*.INVESTMENT` (equity, credit-normal)
- `CLIENT_*.HOLDINGS` (asset, debit-normal)
- `CLIENT_*.DIVIDEND_REVENUE` (revenue, credit-normal)

### 2. Establish Revenue and Expense Accounts for Fees

Create separate revenue and expense accounts to track trading fees and transaction costs.

### 3. Add Accounts for Realized and Unrealized Gains/Losses

Track realized gains/losses (when securities sell) separately from unrealized gains/losses (value changes of held securities).

## Define Transaction Codes

### 1. Transaction Code for Buying Securities

Create `BUY_SECURITY` transaction code with parameters:

- `clientHoldingsAcct` (UUID)
- `clientInvestAcct` (UUID)
- `clientCashAcct` (UUID)
- `security` (STRING)
- `shares` (DECIMAL)
- `price` (DECIMAL)
- `feeRate` (DECIMAL)
- `effective` (DATE)

The transaction generates seven ledger entries capturing cash payment, investment posting, broker revenue, fees, settlement, and share movements.

### 2. Transaction Code for Selling Securities

Establish `SELL_SECURITY` code debiting customer's securities account and crediting cash account for sale proceeds.

### 3. Transaction Codes for Dividends/Interest

Define `DIVIDEND_PAYMENT` or `INTEREST_PAYMENT` codes that debit asset accounts and credit customer cash accounts.

## Post Transactions

### Example: Buying a Security

For CLIENT_A purchasing 100 AAPL shares at $149.45 with 0.75% fee (effective 2022-10-12):

- clientHoldingsAcct: UUID for CLIENT_A.HOLDINGS
- clientInvestAcct: UUID for CLIENT_A.INVESTMENT
- clientCashAcct: UUID for CLIENT_A.CASH
- security: 'AAPL'
- shares: 100
- price: 149.45
- feeRate: 0.0075

Total cash debit: $15,057.0875 (includes fee)

## Monitor and Analyze

### 1. Query Account Balances and History

Use Twisp's GraphQL API to query balances and transaction history to identify discrepancies.

### 2. Track Portfolio Performance Over Time

Use Twisp's layering system to create balance snapshots at different dates and compare them.

### 3. Review and Audit Trading Activities

Leverage immutable ledger records and versioned account history for audit, fraud detection, and regulatory compliance.

### 4. Generate Financial Reports

Create reports from ledger data for regulatory compliance and management decision-making.
