# Guide: Handling FBO Clearing (Payment Facilitation) with Twisp

> Source: https://www.twisp.com/docs/guides/fbo-clearing-payment-facilitation

## Overview

This guide provides step-by-step instructions for managing FBO (Fund Bank Omnibus) clearing using Twisp's accounting tools. It addresses the complete payment facilitation lifecycle.

## Setting Up Accounts

### Settlement Account

Create an account to manage transactions between parties during the clearing process:

- Code: "FBO_SETTLE"
- Normal Balance Type: DEBIT

### Payment Processing Fees Account

Dedicated account for tracking collected fees:

- Code: "PROCESSING_FEES"
- Normal Balance Type: DEBIT

### Party-Specific Accounts

For merchants and customers, create separate accounts with CREDIT balance types to track individual fund movements and transaction histories.

## Defining Transaction Codes

Three primary transaction codes structure the FBO clearing process:

1. **FBO_TRANSFER**: Moves funds between parties (Sender debit, Receiver credit)
2. **FBO_FEE**: Handles fee deductions (Sender debit, Fee account credit)
3. **FBO_DISBURSE**: Manages final disbursements (Receiver debit, Destination credit)

## Modeling Transactions

Post transactions using the defined tran codes through the `postTransaction` mutation:

- Recording fund transfers between parties
- Deducting processing fees via `FEE_DEDUCTION` tran code
- Disbursing funds to final destinations using `FUNDS_DISBURSEMENT`

## Monitoring Settlement

Query settlement accounts and transactions using GraphQL to track balances and transaction status. Pay attention to transaction layers (SETTLED, PENDING, ENCUMBRANCE) and sequencing.

## Reconciliation and Settlement

- Utilize timing and sequencing features for precise event recording
- Leverage the Three Layer Model to manage transactions at different clearing stages
- Compare ledger entries with external records to identify discrepancies
- Move transactions from Pending to Settled layer upon confirmation

## Reporting and Compliance

Extract transaction data via filtered GraphQL queries for auditing. Generate financial reports aggregating transaction data by account and period, using layered accounting to distinguish settlement stages.

## Optimization

Monitor transaction patterns to identify trends and bottlenecks. Update transaction codes and account structures to improve efficiency and reduce processing costs.
