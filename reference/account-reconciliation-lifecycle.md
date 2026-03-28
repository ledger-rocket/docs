# The 4-Account Reconciliation Model

## The Core Problem

Any system that moves money through an external bank faces a fundamental challenge: two independent realities exist for every movement.

1. **The operational reality** — the system instructed a movement (e.g., initiated a wire, submitted an ACH batch, triggered a card payment)
2. **The external reality** — the bank, payment processor, or counterparty confirms it happened (e.g., bank statement, settlement file, confirmation webhook)

These two realities are produced by different systems, at different times, with different levels of detail. The 4-account model reconciles them structurally — using double-entry balances rather than procedural matching logic — so that discrepancies are isolated instantly and automatically.

## The Four Accounts

Every reconcilable external position (a bank account, a nostro, a payment processor settlement account) is represented by four linked accounts:

| Account      | functional_type | Role                                                                                  |
| ------------ | --------------- | ------------------------------------------------------------------------------------- |
| **Settled**  | STANDARD        | The truth — confirmed position at the external institution                            |
| **Clearing** | CLEARING        | Internal record of expected movements not yet confirmed externally                    |
| **Control**  | CONTROL         | The institution's confirmed view of cash movements, ingested from statements or feeds |
| **Suspense** | SUSPENSE        | The reconciliation "mixing bowl" where Clearing and Control are flushed and netted    |

## Happy Path: Full Lifecycle of a Movement

### Starting State

Settled holds $1,000,000 in confirmed funds. Clearing, Control, and Suspense are all zero.

```
Settled:    $1,000,000 Dr
Clearing:    $0
Control:    $0
Suspense:   $0
```

### Step 1: Operational Execution (Internal Reality)

The system processes an outbound movement of $1,000 — a disbursement, a sweep wire, a payment, anything. The other leg of this transfer comes from whatever account originated the instruction (a customer hold, an operational account, etc.). Clearing records that cash is expected to leave:

```
Cr Clearing  $1,000    <- "We expect $1,000 to leave"
```

Settled is untouched. The bank hasn't confirmed anything yet. Clearing carries the expectation.

```
Settled:    $1,000,000 Dr
Clearing:    $1,000 Cr       <- our expectation
Control:    $0
Suspense:   $0
```

The same works in reverse for inbound movements — Clearing would carry a debit representing cash we expect to arrive.

### Step 2: External Confirmation (External Reality)

The bank's statement (or settlement file, or confirmation feed) confirms $1,000 left the account. This is ingested as-is — recording exactly what the external source reports, without interpretation or matching at this stage:

```
Cr Settled  $1,000    <- confirmed position decreased
Dr Control  $1,000    <- external source says it happened
```

Now two independent records exist: Clearing says we expected it, Control says the bank confirmed it.

```
Settled:    $999,000 Dr      <- external source confirmed funds left
Clearing:    $1,000 Cr        <- we expected funds to leave
Control:    $1,000 Dr        <- external source says they left
Suspense:   $0
```

### Step 3: Reconciliation (The Mixing Bowl)

The matching engine finds that Clearing and Control both have entries for the same transaction reference. It flushes both into Suspense:

**Flush Clearing:**

```
Dr Clearing  $1,000     <- clear our expectation
Cr Suspense $1,000     <- move to mixing bowl
```

**Flush Control:**

```
Cr Control  $1,000     <- clear the external entry
Dr Suspense $1,000     <- move to mixing bowl
```

The two Suspense entries — one debit, one credit — net against each other:

```
Settled:    $999,000 Dr      <- true confirmed position
Clearing:    $0               <- all expectations accounted for
Control:    $0               <- all external entries matched
Suspense:   $0               <- Dr and Cr cancel = CLEAN
```

This is the happy path. The movement flowed cleanly through all four accounts and netted to zero.

## What Happens When Things Don't Match

### Scenario A: External Source Reports a Different Amount

We expected $1,000 out (Clearing). The bank confirms $1,015 out (Control) — an unexpected $15 fee was deducted.

After the matching engine pairs and flushes both to Suspense:

```
Clearing:    $0               <- our $1,000 expectation cleared
Control:    $0               <- bank's $1,015 entry cleared
Suspense:   $15 Dr           <- ORPHAN - unexplained difference
```

The $15 sits in Suspense until someone investigates and books it (e.g., debit a Fee Expense account, credit Suspense). The discrepancy is isolated to a single balance — no scanning, no report generation, just one account read.

### Scenario B: We Executed but External Confirmation Hasn't Arrived (T+1/T+2 Delay)

Clearing has $1,000 but Control has nothing — the bank statement hasn't arrived yet:

```
Clearing:    $1,000 Cr        <- still waiting for external confirmation
Control:    $0               <- no statement yet
Suspense:   $0
```

This is normal. Clearing balances age naturally over the settlement window (T+1, T+2, or longer depending on the rail). Only aged Clearing items trigger alerts — the threshold depends on the expected settlement cycle for that institution or payment method.

### Scenario C: External Source Reports Something We Never Initiated

Control has $500 but Clearing has no matching entry:

```
Clearing:    $0               <- we didn't expect this
Control:    $500 Dr          <- bank says it happened
```

The $500 stays orphaned in Control. The matching engine can't flush it because there's no Clearing counterpart. This triggers immediate investigation — the external institution moved funds that the system didn't initiate. Could be a bank fee, an erroneous debit, a fraud event, or a movement from another system that wasn't recorded.

### Scenario D: We Executed but External Source Confirms a Partial Amount

We expected $1,000 out. The bank confirms $950 out — partial execution.

After flushing both to Suspense:

```
Suspense:   $50 Cr           <- we expected more than the bank moved
```

Suspense isolates the $50 difference. Could be a partial execution, a fee offset, or an error. Investigation resolves it without blocking any other reconciliation.

### Scenario E: Duplicate External Entry

The bank statement accidentally reports the same $1,000 movement twice. Clearing has one entry, Control has two:

After flushing the matched pair, one Control entry remains:

```
Clearing:    $0               <- our single expectation cleared
Control:    $1,000 Dr        <- duplicate bank entry, no Clearing counterpart
Suspense:   $0               <- the matched pair netted cleanly
```

The orphaned Control entry immediately flags the duplicate. Without the 4-account model, a duplicate bank entry could silently double-count.

## The Invariant

After every reconciliation cycle, the health check is three balance reads:

| Account  | Healthy State                                                     | Alert Condition                                                               |
| -------- | ----------------------------------------------------------------- | ----------------------------------------------------------------------------- |
| Clearing | Zero, or only entries younger than the expected settlement window | Aged entries = expected movement never confirmed externally                   |
| Control  | Zero                                                              | Any balance = external source reported something with no internal counterpart |
| Suspense | Zero, or only recently matched pairs within the settlement window | Aged balance = confirmed discrepancy requiring human investigation            |

Settled is always authoritative — it tracks the true confirmed position as reported by the external source. The other three accounts exist solely to reconcile the internal operational view against the external view and isolate differences.

## Directionality

The model works identically for inbound and outbound movements. The only difference is which side carries the debit and credit:

| Direction               | Clearing Entry | Control Entry | Settled Entry               |
| ----------------------- | -------------- | ------------- | --------------------------- |
| Outbound (cash leaving) | Credit         | Debit         | Credit (position decreases) |
| Inbound (cash arriving) | Debit          | Credit        | Debit (position increases)  |

The Suspense netting logic is the same regardless of direction.

## Per-Document Isolation

Each external file ingested into the system — a bank statement, a settlement file, a confirmation batch — gets its own dedicated Control account instance. This means:

- **Reconciliation status is per-file**: zero balance on that Control instance = that file is fully matched
- **Parallel processing**: multiple files can process concurrently without cross-contamination
- **Anomaly pinpointing**: a non-zero balance tells you exactly which file has unresolved entries
- **Audit evidence**: each zeroed-out instance is a permanent, queryable proof that a specific file reconciled cleanly

The same pattern applies to Clearing if outbound movements are batched — each batch gets its own Clearing instance, and the invariant applies per-batch.

## Event Granularity

A per-file account instance does NOT mean a single event for the entire file. Each line item in the file is its own atomic event, posting to the shared per-file account instance. For example, a bank statement with 200 line items produces 200 individual events, each posting one transfer to the same per-file Control instance. Similarly, an allocation file with 300,000 consumers produces 300,000 individual events, each drawing down the same per-file Control or Clearing instance.

This means:

- **The zero-balance invariant is a file-level check**, not an event-level check. Each individual event draws the per-file instance closer to zero. Zero is reached only when ALL line items in the file have been processed.
- **Error isolation**: if one line item fails, only that event fails. The other line items process independently. The per-file instance balance tells you exactly how much remains unprocessed.
- **Progress tracking**: the remaining balance on the per-file instance is a real-time progress indicator — it decreases with each successful event and reaches zero when the file is complete.
- **Retry granularity**: a failed line item can be retried individually without reprocessing the entire file.

## Where This Pattern Applies

The 4-account model is not limited to bank accounts. It applies to any external position that has independent confirmation:

- **FBO bank accounts** — daily bank statements confirm physical cash
- **Nostro/vostro accounts** — correspondent bank statements confirm interbank positions
- **Payment processor settlement** — settlement files confirm net positions
- **Card scheme settlement** — scheme files confirm interchange and net settlement
- **Crypto custody** — on-chain confirmations confirm wallet balances
- **Sweep network accounts** — sweep bank confirmations confirm fund placement

Any position where "what we think happened" and "what the external party confirms happened" are two separate data streams benefits from this model.
