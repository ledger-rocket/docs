# Payment Flows

How to model common payment flows using LedgerRocket templates. All examples reference real templates from the site 5 configuration.

## Core Concepts

Each payment flow is a sequence of template-driven events. Templates define the double-entry legs (debit/credit pairs) that fire atomically. Workflows chain templates together to model the full lifecycle of a payment.

Key account codes used across flows:

| Code | Name | Role |
|------|------|------|
| 10100 | Settled | Confirmed bank cash |
| 10101 | Clearing Total | Expected cash (allocation files) |
| 10102 | Control | Per-file bank statement staging |
| 10103 | Suspense | Temporary reconciliation parking |
| 10104 | Allocation Clearing / ACH Clearing Files | Per-file loading checksum |
| 20100 | Customer Available | Spendable balance |
| 20101 | Customer Hold | Reserved pending confirmation |
| 20102 | Pending Disbursements | Queued for ACH payout |
| 20900 | Unallocated Customer | Deposits not yet matched |

## Deposit Flow (Templates 5108, 5103, 5104, 5401, 5407, 5402)

Processes inbound deposit allocation files through ingestion, consumer matching, and bank-confirmed release.

### Step 1: Allocation Batch Header (T5108)

Record the arrival of an allocation file. Establishes the per-file clearing balance as a loading checksum.

- DR Clearing Total (10101) / CR Allocation Clearing per-file (10104)
- Fires once per allocation file

```bash
curl -s -X POST https://ledger.dev.ledgerrocket.com/v2/event-service/api/v1/events \
  -H "Content-Type: application/json" \
  -H "x-api-key: lr-test-20250701" \
  -d '{
    "event_id": "019a0001-0000-7000-8000-000000000001",
    "site_id": 5,
    "template_ids": [5108],
    "amount": 50000,
    "occurred_at": 1705315800000,
    "event_source": "allocation-engine",
    "accounts": [
      {"account_id": "CLEARING_TOTAL_UUID", "purpose": "clearing_total"},
      {"account_id": "ALLOC_CLEARING_FILE_UUID", "purpose": "allocation_clearing_file"}
    ],
    "extra_metadata": {"allocation_file_id": "ALLOC-2026-03-28-001"}
  }'
```

### Step 2: Allocation Detail Lines (T5103)

Load individual deposit lines into the unallocated staging area. Draws down the per-file clearing balance.

- DR Allocation Clearing per-file (10104) / CR Unallocated per-file (10104)
- Fires per line item in the allocation file

### Step 3: Consumer Match (T5104)

Reconciliation service resolves raw consumer references to real account IDs. Moves funds from unallocated staging to consumer hold.

- DR Unallocated (10104) / CR Consumer Hold (20101)
- Mirrors on allocation ledger for FDIC tracking
- Funds go to Hold (not Available) because matching may complete before bank confirmation

### Step 4: Bank Statement Ingestion (T5401)

Record inbound bank statement lines confirming physical cash arrival.

- DR Settled (10100) / CR Control per-file (10102) on economic ledger
- DR Cash Balances FBO (11900) / CR Cash Balances Control (11902) on allocation ledger
- Fires per bank statement line item

### Step 5: Control Flush (T5407)

Pair the bank statement against the allocation file at the aggregate level.

- DR Control (10102) / CR Suspense (10103)
- Fires once per file pair

### Step 6: Consumer Release (T5402)

Release consumer funds from Hold to Available. Clears suspense.

- DR Suspense (10103) / CR Clearing Total (10101)
- DR Consumer Hold (20101) / CR Consumer Available (20100)
- Fires per consumer

**Final state:** Clearing, Control, and Suspense all net to zero. Customer Available increased. Settled increased.

## Disbursement Flow (Templates 5202, 5203, 5204, 5205, 5210, 5211, 5212)

Records customer payouts via the ACH rail, including returns handling.

### Step 1: Disbursement Initiate (T5202)

Move customer funds from Available to Pending Disbursements. Liability-only -- no house assets touched.

- DR Customer Available (20100) / CR Pending Disbursements (20102)
- DR Customer FBO Main (21100) / CR Customer FBO ACH Payable (21102) on allocation ledger

### Step 2: ACH Batch Header (T5203)

Establish the per-file clearing balance for the NACHA file.

- DR ACH Clearing Files per-file (10104) / CR Clearing Total (10101)
- Fires once per NACHA file

### Step 3: Per-Customer ACH Submission (T5204)

Record each customer's entry in the NACHA file. Zeros out Pending Disbursements.

- DR Pending Disbursements (20102) / CR ACH Clearing Files per-file (10104)
- Fires per customer in the file

### Step 4: Bank Statement Received (T5205)

Confirm cash has physically left the FBO account.

- DR Control (10102) / CR Settled (10100) -- reversed from deposit direction
- Fires per bank statement line

### Step 5: Disbursement Clearing Reconciliation (T5210)

Flush disbursement clearing through Suspense per detail line.

- DR Clearing Total (10101) / CR Suspense (10103)

### Step 6: Returns Clearing and Release (T5211)

For ACH returns: flush return clearing and release customer funds from Hold back to Available.

- DR Suspense (10103) / CR Clearing Total (10101) -- reverse direction
- DR Customer Hold (20101) / CR Customer Available (20100)

### Step 7: Bank Payment Reconciliation (T5212)

Pair the bank statement against clearing and return sides.

- DR Suspense (10103) / CR Control (10102)

**Final state:** All clearing, control, and suspense accounts net to zero. Customer Available decreased by net disbursements minus returns.

## FDIC Sweep Flow (Templates 5601-5611)

Manages fund movements between the Master FBO and the Sweep Network for FDIC insurance coverage.

### Phase 1: Aggregation and Allocation

**Aggregate FBO to Sweep (T5601):** Create batch-level clearing positions and queue sweep intentions.

**Customer Allocation (T5602):** Per-customer expected movements on the FDIC tracking ledger. Anchors each customer's cash-in-transit chain.

### Phase 2: Payment and Bank Confirmation

**Payment Initiated (T5603):** Clear sweep intentions from the queue once the bank payment is instructed. Moves sweep from "queued" to "in flight."

**FBO Bank Statement (T5604):** Confirm cash has left the Master FBO. DR Control (10102) / CR Settled (10100).

**Sweep Bank Statement (T5605):** Confirm cash has arrived at the Sweep Network. DR Settled (10200) / CR Control (10202).

### Phase 3: FBO-Side Reconciliation

Three-stage pattern mirrors the deposit reconciliation:

1. **Clearing Flush (T5606):** DR Clearing Total (10101) / CR Suspense (10103)
2. **Bank Payment Pairing (T5607):** DR Suspense (10103) / CR Control (10102)
3. **Per-Customer Cash-in-Transit (T5608):** Move FDIC position from Master FBO to Cash in Transit

### Phase 4: Sweep-Side Reconciliation

Mirrors the FBO pattern on the destination side:

1. **Clearing Flush (T5609):** DR Clearing (10201) / CR Suspense (10203)
2. **Bank Payment Pairing (T5610):** DR Control (10202) / CR Suspense (10203)
3. **Per-Customer Final Position (T5611):** Move FDIC position from Cash in Transit to final Sweep Network position

**Final state:** Sweep queue zeroes when payment is instructed. FBO and Sweep clearing each zero after reconciliation. Cash in Transit zeroes when both sides complete per-customer moves. FDIC positions reflect the physical fund locations.

## Workflow Progression Pattern

All flows follow the same progression:

1. **Intention** -- record the expected movement (clearing, allocation)
2. **Execution** -- perform the payment (ACH file, bank wire, sweep instruction)
3. **Confirmation** -- bank statement confirms physical cash movement
4. **Reconciliation** -- flush temporary accounts (clearing, control) through suspense
5. **Release** -- promote customer funds from Hold to Available

Temporary accounts (clearing, control, suspense) must all net to zero when the workflow completes. A non-zero balance indicates an incomplete or anomalous workflow requiring investigation.
