# Processor: ACH

> Source: https://www.twisp.com/docs/processors/ach

## Overview

The Twisp ACH processor manages the complete lifecycle of Automated Clearing House transactions, from initiation through settlement or return. It provides production-ready workflows, file generation, webhook decisioning, and ledger integration.

The processor operates in two primary capacities:

- **ODFI**: Originating Depository Financial Institution (sends ACH transactions)
- **RDFI**: Receiving Depository Financial Institution (receives ACH transactions)

## Key Complexity Factors

- **File Format Compliance**: NACHA format demands precise 94-character fixed-width records. Format errors cause complete file rejection.
- **Asynchronous Settlement**: ACH transactions require 1-3 business days to settle, necessitating tracking of pending, encumbered, and settled states.
- **Return Handling**: Transactions can return days or weeks after origination (up to 60 days for certain types).
- **Multi-Layer Balance Management**: Customer available balances must account for encumbered, pending, and settled funds.
- **Webhook Decisioning**: RDFI operations require real-time business logic decisions within strict processing windows.
- **Regulatory Compliance**: NACHA rules govern transaction types, authorization, return timeframes, and security.

## Architecture Components

### Configuration Layer

Central operational configuration includes:

- Settlement account for fund transit
- Suspense account for unidentifiable transactions
- Exception account for business rule violations
- Fee account for processing fees
- Webhook endpoint for transaction decisioning
- ODFI header information for file generation
- Timezone for date/time interpretation

Configuration versioning maintains immutability -- files reference specific versions, enabling historical analysis after updates.

### File Operations Layer

Manages NACHA file lifecycle:

- **Upload**: Presigned URLs enable secure, time-limited uploads directly to S3-compatible storage
- **Processing**: File parsing validates NACHA format, extracts batch and entry details, partitions entries for parallel processing. Each entry triggers webhook decisioning.
- **Generation**: Queries submitted transactions, groups by batch parameters, produces NACHA-compliant files
- **Download**: Presigned URLs provide time-limited file access

Processing status tracks progression: NEW -> VALIDATING -> PROCESSING -> COMPLETED.

### Workflow Layer

**PUSH Workflow (Credits)**: Sends money to receivers. CREATE encumbers customer funds; SUBMIT settles and includes in file generation. Returns reverse settled entries.

**PULL Workflow (Debits)**: Collects money from receivers. CREATE encumbers settlement funds; SUBMIT moves to pending; SETTLE finalizes after confirmation. Pending layer prevents premature fund availability.

### Webhook Decisioning Layer

- **Request Format**: POST requests include transaction details (amount, account identifiers, entry metadata, trace number, effective date)
- **Response Format**: Specifies settlement instructions (which account to credit/debit)
- **Timeout Handling**: Strict timeframes (typically seconds). Timeouts result in transaction failures.
- **Decision Routing**: Settlement account (normal processing), Suspense account (unknown, manual review), Exception account (business rule violation, returned)

### Balance Layer Management

- **Encumbrance Layer**: Holds funds during CREATE state before transmission. Encumbered funds don't affect available balance but prevent overdrafts.
- **Pending Layer**: PULL workflows use pending between SUBMIT and SETTLE. Represents transmitted funds awaiting confirmation.
- **Settled Layer**: Final layer for completed transactions. Fees post directly to settled layer immediately.
- **Available Balance Formula**: Settled - Pending - Encumbrance.

## Design Rationale

- **Separation of Concerns**: File operations, workflows, ledger accounting, and decisioning are independent components
- **Immutability**: Configuration versions, workflow executions, and file processing records are immutable for audit trails
- **Idempotency**: Workflow executions use correlation IDs; file processing matches returns via trace numbers
- **Parallel Processing**: File processing partitions entries across workers
- **Webhook Flexibility**: Business logic resides in webhooks, not the processor
- **Multi-Tenancy**: Configurations are tenant-scoped with complete isolation

## Workflow State Machines

### PUSH State Machine

```text
CREATE -> SUBMIT -> [completed]
  |
CANCEL -> REIMBURSE_FEE -> [completed]

SUBMIT -> RETURN -> REIMBURSE_FEE -> [completed]
```

### PULL State Machine

```text
CREATE -> SUBMIT -> SETTLE -> [completed]
  |
CANCEL -> REIMBURSE_FEE -> [completed]

SUBMIT -> RETURN -> REIMBURSE_FEE -> [completed]
```

## Return Processing Flow

1. **Receipt**: Return file uploads via presigned URL
2. **Parsing**: Processing extracts return entries with return codes and trace numbers
3. **Matching**: System queries original transactions via trace number indexes
4. **Execution**: RETURN workflow state executes for each matched transaction
5. **Reversal**: Ledger entries reverse from appropriate balance layer
6. **Audit**: Complete return processing history records in file and workflow execution history

## Integration Points

- **Ledger Integration**: Workflows create ledger transactions via tran codes
- **Event System**: Webhook endpoints route to events system; file processing status changes emit events
- **Files Service**: Presigned URLs delegate upload/download
- **Workflow Engine**: ACH workflows are workflow templates with specific parameters and state machines

## Production Considerations

- **ODFI Relationships**: Establish banking relationships for ACH origination
- **File Transmission**: You are responsible for secure file transmission (typically SFTP) to/from financial institutions
- **Return Timeframes**: NACHA rules specify return timeframes (typically 2 business days; some codes allow 60 days)
- **Balance Management**: Properly managing all three layers prevents overdrafts
- **Webhook Performance**: Must respond within strict timeframes; consider caching and fallback strategies
- **Reconciliation**: Daily reconciliation validates file processing, ledger balances, and transaction counts
