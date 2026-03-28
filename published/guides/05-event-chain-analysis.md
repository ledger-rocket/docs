# Event Chain Analysis

How to trace financial events through the system using event chains and the Balance Service query endpoints.

## What Is an Event Chain?

An event chain links a group of related events through the `original_event_id` field. When an event is submitted without an `original_event_id`, it becomes the chain root. Subsequent events that reference it (corrections, reversals, follow-on steps) form the chain.

Chains answer the question: "What happened to this transaction from start to finish?"

For example, a deposit chain might contain:

1. Original allocation event (chain root)
2. Consumer matching event
3. Reconciliation flush event
4. Consumer release event

All of these share the same `original_event_id`, so querying the chain shows the full lifecycle.

## Chain Detail

Retrieve all events and transfers in a chain:

```bash
curl -s "https://ledger.dev.ledgerrocket.com/v2/balance-service/api/v1/events/EVENT_UUID/chain?site_id=5" \
  -H "x-api-key: lr-test-20250701" | jq .
```

The response includes every transfer grouped by event, showing the debit/credit pairs and amounts. This is the primary tool for understanding what a transaction did.

## Chain Balance

Get the net balance impact of an entire chain:

```bash
curl -s "https://ledger.dev.ledgerrocket.com/v2/balance-service/api/v1/events/EVENT_UUID/balance?site_id=5" \
  -H "x-api-key: lr-test-20250701" | jq .
```

This returns the aggregate credits and debits across all events in the chain. For a correctly completed workflow, temporary accounts (clearing, control, suspense) should show zero net impact.

## Per-Chain Account Breakdown

See how a specific account's balance is decomposed by event chain:

```bash
curl -s "https://ledger.dev.ledgerrocket.com/v2/balance-service/api/v1/accounts/ACCOUNT_UUID/chain-balances?site_id=5" \
  -H "x-api-key: lr-test-20250701" | jq .
```

This is valuable for investigating why an account has an unexpected balance. Each chain's contribution is listed separately, making it easy to identify which transaction left a residual.

## Batch Chain Summary

Query multiple chains at once:

```bash
curl -s -X POST "https://ledger.dev.ledgerrocket.com/v2/balance-service/api/v1/chain-summary:query" \
  -H "Content-Type: application/json" \
  -H "x-api-key: lr-test-20250701" \
  -d '{
    "site_id": 5,
    "event_ids": [
      "019a0001-0000-7000-8000-000000000001",
      "019a0001-0000-7000-8000-000000000002",
      "019a0001-0000-7000-8000-000000000003"
    ]
  }' | jq .
```

Use this for bulk reconciliation checks -- pass all event IDs from a batch and verify each chain's net impact.

## Viewing Transfers

### Transfers by Event

```bash
# All transfers created by a single event
curl -s "https://ledger.dev.ledgerrocket.com/v2/event-service/api/v1/events/EVENT_UUID/transfers?site_id=5" \
  -H "x-api-key: lr-test-20250701" | jq .

# Enriched transfers (includes account names, entity details, ledger metadata)
curl -s "https://ledger.dev.ledgerrocket.com/v2/event-service/api/v1/events/EVENT_UUID/transfers/enriched?site_id=5" \
  -H "x-api-key: lr-test-20250701" | jq .
```

### Transfers by Account

```bash
curl -s "https://ledger.dev.ledgerrocket.com/v2/event-service/api/v1/accounts/ACCOUNT_UUID/transfers?site_id=5" \
  -H "x-api-key: lr-test-20250701" | jq .
```

## Debugging Tips

### Follow the Money Through Templates

When investigating a balance discrepancy:

1. **Identify the account** with the unexpected balance
2. **Query chain-balances** to find which chain left the residual:

   ```bash
   curl -s "https://ledger.dev.ledgerrocket.com/v2/balance-service/api/v1/accounts/ACCOUNT_UUID/chain-balances?site_id=5" \
     -H "x-api-key: lr-test-20250701" | jq '.[] | select(.net_balance != "0")'
   ```

3. **Get the chain detail** to see all events in the problematic chain:

   ```bash
   curl -s "https://ledger.dev.ledgerrocket.com/v2/balance-service/api/v1/events/CHAIN_ROOT_UUID/chain?site_id=5" \
     -H "x-api-key: lr-test-20250701" | jq .
   ```

4. **Check which workflow step is missing** -- compare the events in the chain against the expected workflow progression (see [Payment Flows](02-payment-flows.md))

### Common Residual Causes

| Account | Non-Zero Means | Likely Cause |
|---------|---------------|--------------|
| Clearing (10101) | Allocation file loaded but not reconciled | Missing T5402 (consumer release) |
| Control (10102) | Bank statement ingested but not paired | Missing T5407 (control flush) |
| Suspense (10103) | Clearing flushed but reconciliation incomplete | Missing bank payment reconciliation step |
| Unallocated (20900) | Detail lines loaded but consumers not matched | Missing T5104 (consumer match) |

### Validate Before Submitting

Use the dry-run endpoint to verify an event before committing:

```bash
curl -s -X POST "https://ledger.dev.ledgerrocket.com/v2/event-service/api/v1/events:validate" \
  -H "Content-Type: application/json" \
  -H "x-api-key: lr-test-20250701" \
  -d '{
    "site_id": 5,
    "template_ids": [5104],
    "amount": 5000,
    "accounts": [
      {"account_id": "UNALLOC_UUID", "purpose": "unallocated_customer"},
      {"account_id": "HOLD_UUID", "purpose": "consumer_hold"}
    ]
  }' | jq .
```

Use the `:explain` variant (`POST /api/v1/events:validate:explain`) with the same body to get a full execution trace showing variable values, validation results, and computed leg amounts.
